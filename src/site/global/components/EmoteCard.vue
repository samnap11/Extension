<template>
	<main class="seventv-emote-card-container">
		<div class="seventv-emote-card">
			<div class="seventv-emote-card-image">
				<img :srcset="srcset" :style="{}" />
			</div>
			<div class="seventv-emote-card-display">
				<div>
					<h3 class="seventv-emote-card-title">{{ emote.name }}</h3>
					<p class="seventv-emote-card-subtitle">{{ subtitle }}</p>
					<a v-if="owner.id" class="seventv-emote-card-user" :href="owner.url" target="_blank">
						<img :src="owner.avatarURL" />
						<span>{{ owner.displayName }}</span>
					</a>

					<div v-if="actor.id" class="seventv-emote-card-data seventv-emote-card-actor">
						<p>Added by</p>
						<p class="seventv-emote-card-user">
							<img :src="actor.avatarURL" />
							<span>{{ actor.displayName }}</span>
						</p>
					</div>

					<div v-if="timestamp" class="seventv-emote-card-data">
						<p>Added on</p>
						<span>{{ timestamp }}</span>
					</div>
				</div>
			</div>
		</div>
	</main>
</template>

<script setup lang="ts">
import { reactive, ref, watch, watchEffect } from "vue";
import { imageHostToSrcsetWithsize } from "@/common/Image";
import { log } from "@/common/Logger";
import { convertTwitchEmote } from "@/common/Transform";
import { useApollo } from "@/composable/useApollo";
import { userQuery } from "@/assets/gql/seventv.user.gql";
import { emoteCardQuery } from "@/assets/gql/tw.emote-card.gql";
import { useQuery } from "@vue/apollo-composable";

const props = defineProps<{
	emote: SevenTV.ActiveEmote;
	size: [width: number, height: number];
}>();

const host = ref<SevenTV.ImageHost | null>(props.emote.data?.host ?? null);
const srcset = ref("");
const owner = reactive(emptyUser());
const actor = reactive(emptyUser());
const subtitle = ref("");
const timestamp = ref("");

function emptyUser() {
	return {
		id: "",
		username: "",
		displayName: "",
		avatarURL: "",
		url: "",
	};
}

watchEffect(async () => {
	// for twitch emotes we
	if (props.emote.provider === "TWITCH") {
		const apollo = useApollo();
		if (!apollo) return;

		const res = await apollo
			.query<emoteCardQuery.Result, emoteCardQuery.Variables>({
				query: emoteCardQuery,
				variables: {
					emoteID: props.emote.id,
					artistEnabled: true,
					octaneEnabled: true,
				},
			})
			.catch((err) => log.error("failed to fetch emote card", err));
		if (!res) return;

		const { emote } = res.data;
		if (!emote) return;

		// extract displayable data
		const e = convertTwitchEmote(emote as unknown as Twitch.TwitchEmote);
		host.value = e.host;

		if (emote.owner) {
			owner.id = emote.owner.id;
			owner.username = emote.owner.login;
			owner.displayName = emote.owner.displayName;
			owner.avatarURL = emote.owner.profileImageURL;
			owner.url = `https://twitch.tv/${emote.owner?.login}`;
		}

		subtitle.value = emote.subscriptionTier?.split("_").join(" ") ?? emote.type;
	} else if (props.emote.provider === "7TV") {
		const { result: emoteActorResult } = useQuery<userQuery.Result, userQuery.Variables>(
			userQuery,
			{
				id: props.emote.actor_id ?? "",
			},
			() => ({
				enabled: !!props.emote.actor_id,
			}),
		);

		watch(
			emoteActorResult,
			(value) => {
				if (!value?.user) return;

				actor.id = value?.user.id;
				actor.username = value?.user.username;
				actor.displayName = value?.user.display_name;
				actor.avatarURL = value?.user.avatar_url;
			},
			{ immediate: true },
		);

		timestamp.value = new Date(props.emote.timestamp ?? 0).toLocaleDateString();
	}
});

watch(
	host,
	(host) => {
		if (!host || !host.files.length) return;

		srcset.value = imageHostToSrcsetWithsize(props.size[0], props.size[1], host, props.emote.provider);
	},
	{ immediate: true },
);
</script>

<style scoped lang="scss">
main.seventv-emote-card-container {
	display: block;
	width: 100%;
	height: 100%;

	.seventv-emote-card {
		display: grid;
		column-gap: 0.25em;
		place-content: center;
		grid-template-columns: 8rem 1fr;
		width: 32rem;
		height: 9rem;
		background-color: var(--seventv-background-transparent-1);
		outline: 0.1em solid var(--seventv-border-transparent-1);
		backdrop-filter: blur(0.5rem);
		border-radius: 0.25rem;

		.seventv-emote-card-image {
			display: grid;
			place-items: center;
			padding: 12.5%;
		}

		.seventv-emote-card-display {
			display: grid;
			padding: 0.5em 0;

			h3 {
				font-size: 2rem;
				font-weight: 600;
				color: var(--seventv-text-primary);
			}
		}

		.seventv-emote-card-subtitle {
			font-size: 0.88rem;
			font-weight: 800;
			color: var(--seventv-text-primary);
		}

		.seventv-emote-card-user {
			display: flex;
			align-items: center;
			gap: 0.5rem;

			> img {
				width: 2rem;
				height: 2rem;
				clip-path: circle(50% at 50% 50%);
			}

			> span {
				font-size: 1.25rem;
				font-weight: 600;
				color: var(--seventv-text-primary);
			}
		}

		.seventv-emote-card-data {
			p:first-child {
				line-height: 1.35;
				color: var(--seventv-muted);
				font-size: 0.88rem;
				font-weight: 700;
				text-transform: uppercase;
			}
		}

		.seventv-emote-card-actor {
			img {
				width: 1.5rem;
				height: 1.5rem;
			}
		}
	}
}
</style>
