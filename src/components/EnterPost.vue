<script setup lang="ts">
import { ref } from "vue";
import { EmojiClickEventDetail } from "emoji-picker-element/shared";
import { IconMoodHappy } from "@tabler/icons-vue";
import {
  PopoverContent,
  PopoverPortal,
  PopoverRoot,
  PopoverTrigger,
} from "radix-vue";
import { useI18n } from "vue-i18n";
import PickEmoji from "./PickEmoji.vue";
import { APIChat } from "../lib/schemas/chat";
import {
  autoResizeTextarea,
  resetTextareaSize,
} from "../lib/autoResizeTextarea";
import { apiRequest, getResponseFromAPIRequest } from "../lib/apiRequest";
import { DiscordSticker } from "../lib/discordEmoji";
import { getReply } from "../lib/getReply";
import { postSchema, APIPost } from "../lib/schemas/post";
import { useAuthStore } from "../stores/auth";
import { useCloudlinkStore } from "../stores/cloudlink";
import { useDialogStore } from "../stores/dialog";
import { useIdsStore } from "../stores/uniqueIds";
import { useSettingsStore } from "../stores/settings";

const { chat } = defineProps<{
  chat?: APIChat | "livechat";
}>();
const emit = defineEmits<{
  optimistic: [post: APIPost];
  pessimistic: [id: string];
}>();

const authStore = useAuthStore();
const cloudlinkStore = useCloudlinkStore();
const dialogStore = useDialogStore();
const settingsStore = useSettingsStore();
const idsStore = useIdsStore();
const { t } = useI18n();

const postContent = ref("");
const posting = ref(false);

const post = async (e?: Event) => {
  e?.preventDefault();
  if (posting.value || !postContent.value) {
    return;
  }
  posting.value = true;
  const content = postContent.value.trim();
  postContent.value = "";
  if (inputRef.value) {
    resetTextareaSize(inputRef.value);
  }
  const id = emitOptimistic(content);
  const response = await getResponseFromAPIRequest(
    chat ? `/posts/${chat === "livechat" ? "livechat" : chat._id}` : "/home",
    {
      method: "POST",
      auth: true,
      body: JSON.stringify({
        content: content,
      }),
      schema: postSchema,
    },
  );
  if ("status" in response) {
    await dialogStore.alert(t("postFail", { status: response.status }));
    posting.value = false;
    emit("pessimistic", id);
    return;
  }
  posting.value = false;
};

const trimmedPost = (post: string) => {
  const reply = getReply(post);
  const postContent = reply ? reply.postContent : post;
  const slicedPostContent = postContent.slice(0, 40);
  const replacedPostContent = slicedPostContent
    .slice(0, 40)
    .replace(/: /g, ":  ") // images shouldn't appear in replies

    .replace(/!/g, "!\u200c") // markdown images
    .replace(/\n/g, " ");
  return `"${replacedPostContent.slice(0, 40).trim()}${
    postContent.length > 39 ? "…" : ""
  }"`;
};

const emitOptimistic = (content: string) => {
  if (!authStore.username) {
    throw new Error("Auth required to post");
  }
  const id = idsStore.newOptimisicPostId();
  emit("optimistic", {
    edited_at: undefined,
    isDeleted: false,
    p: content,
    post_id: id,
    post_origin: chat === "livechat" ? "livechat" : chat ? chat._id : "home",
    t: {
      e: new Date().getTime(),
    },
    type: 1,
    u: authStore.username,
    unfiltered_p: undefined,
  });
  return id;
};

const inputRef = ref<HTMLTextAreaElement | null>(null);
const reply = (username: string, content: string, postId: string) => {
  postContent.value =
    `@${username} ${trimmedPost(content)} (${postId})\n` + postContent.value;
  if (inputRef.value) {
    inputRef.value.focus();
    autoResizeTextarea(inputRef.value);
  }
};

const lastTypingIndicatorSent = ref<number | null>(null);
const input = async () => {
  if (chat === "livechat") {
    return;
  }
  const currentDate = new Date().getTime();
  if (inputRef.value) {
    autoResizeTextarea(inputRef.value);
  }
  if (
    lastTypingIndicatorSent.value === null ||
    lastTypingIndicatorSent.value + 1500 < currentDate
  ) {
    lastTypingIndicatorSent.value = currentDate;
    const status = await apiRequest(
      chat ? `/chats/${chat._id}/typing` : "/home/typing",
      {
        auth: true,
        method: "POST",
      },
    );
    if (status.status !== 200) {
      // intentionally not localized, this is a console error
      console.error(`Failed to send typing indicator (${status})`);
    }
  }
};

const keydown = (e: KeyboardEvent) => {
  if (
    e.key === "Enter" &&
    ((!e.shiftKey && settingsStore.enterSends) ||
      (e.shiftKey && !settingsStore.enterSends))
  ) {
    e.preventDefault();
    post();
  }
};

const addEmoji = (emoji: EmojiClickEventDetail) => {
  if (!inputRef.value) {
    return;
  }
  postContent.value += emoji.unicode ?? emoji.emoji.shortcodes?.[0];
  autoResizeTextarea(inputRef.value);
};

const postSticker = (sticker: DiscordSticker) => {
  if (!inputRef.value) {
    return;
  }
  postContent.value += ` [(sticker) ${sticker.name}: ${sticker.url}]`;
  autoResizeTextarea(inputRef.value);
};

defineExpose({ reply });
</script>

<template>
  <form @submit="post" class="flex gap-2">
    <textarea
      class="w-full resize-none rounded-lg border-2 border-accent bg-transparent px-2 py-1"
      :placeholder="$t('enterPostPlaceholder')"
      @input="input"
      @keydown="keydown"
      v-model="postContent"
      ref="inputRef"
      rows="1"
    ></textarea>
    <PopoverRoot>
      <PopoverTrigger class="rounded-xl bg-accent px-2 py-1 text-accent-text">
        <IconMoodHappy aria-hidden />
        <span class="sr-only">Emoji</span>
      </PopoverTrigger>
      <PopoverPortal>
        <PopoverContent
          class="z-20 mt-2 rounded-lg bg-accent px-2 py-1 text-accent-text shadow-lg"
        >
          <PickEmoji @emoji="addEmoji" @sticker="postSticker" />
        </PopoverContent>
      </PopoverPortal>
    </PopoverRoot>
    <button
      type="submit"
      class="whitespace-nowrap rounded-xl bg-accent px-2 py-1 text-accent-text"
      v-if="postContent.trim()"
    >
      {{ $t("enterPostSend") }}
    </button>
  </form>
</template>
