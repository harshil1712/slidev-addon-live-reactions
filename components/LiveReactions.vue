<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from "vue";

import { configs, useNav } from "@slidev/client";

const { isPresenter, slides, currentPage, queryClicks, go } = useNav();

enum ConnectionStatus {
  CONNECTED,
  DISCONNECTED,
  ERROR,
  IDLE,
}

enum SendType {
  REACTIONS = "reactions",
  SLIDEUPDATE = "slideupdate",
}

interface SlideUpdateState {
  mtype: SendType.SLIDEUPDATE;
  page: number | string;
  click: number;
  type: "broadcast";
}

interface ReactionState {
  mtype: SendType.REACTIONS;
  emoji: string;
  type: "broadcast";
  page: number | string;
  slideTitle: string;
  feedback: "okay" | "good" | "great" | "mindBlown";
}

interface AnimatedEmojiData {
  id: number;
  emoji: string;
  x: number;
  delay: number;
}

type SendState = ReactionState | SlideUpdateState;

let webSocket: WebSocket;

const connectState = ref<ConnectionStatus>(ConnectionStatus.DISCONNECTED);
const animatedEmojis = ref<AnimatedEmojiData[]>([]);
let emojiCounter = 0;

// Get WS server from th config
function getWsServer() {
  const wsServer =
    configs.liveReactions.server
      .replace("http://", "ws://")
      .replace("https://", "wss://")
      .replace(/\/+$/, "") || "ws://localhost:8787";
  const title = configs.title;
  return wsServer + `/ws/${title}`;
}

// Initialize the WebSocket connection
function initWebSocket() {
  if (!webSocket) {
    webSocket = new WebSocket(getWsServer());
    webSocket.addEventListener("message", onMessage);
    webSocket.addEventListener("open", onOpen);
    webSocket.addEventListener("close", onClose);
  } else {
    onOpen();
  }
}

// Close the WebSocket connection
function closeWebSocket() {
  if (webSocket) {
    connectState.value = ConnectionStatus.DISCONNECTED;
    webSocket.close();
  } else {
    connectState.value = ConnectionStatus.DISCONNECTED;
  }
}

// Handle WebSocket onOpen event
function onOpen() {
  connectState.value = ConnectionStatus.CONNECTED;
  console.log("WebSocket connected");
}

// Handle WebSocket onClose event
function onClose() {
  if (connectState.value === ConnectionStatus.IDLE) {
    connectState.value = ConnectionStatus.ERROR;
  } else {
    connectState.value = ConnectionStatus.DISCONNECTED;
  }
}

// Handle WebSocket onMessage event
function onMessage(event) {
  const { mtype, type, page } = JSON.parse(event.data) as
    | ReactionState
    | SlideUpdateState;

  // check if it is a presenter
  if (!isPresenter.value && type === "broadcast") {
    if (mtype === SendType.SLIDEUPDATE) {
      const { click } = JSON.parse(event.data) as SlideUpdateState;
      go(page, click);
    }
    if (mtype === SendType.REACTIONS && page === currentPage.value) {
      const { emoji } = JSON.parse(event.data) as ReactionState;
      addAnimatedEmoji(emoji);
    }
  }
}

// send reactions to the server
function sendReaction(emojiKey: keyof ReturnType<typeof getEmoji>) {
  const page = currentPage.value;
  const slideTitle = slides.value[page - 1].meta.slide.title;
  if (connectState.value === ConnectionStatus.CONNECTED) {
    const emojis = getEmoji();
    const emoji = emojis[emojiKey];
    const reactionState: SendState = {
      mtype: SendType.REACTIONS,
      emoji,
      type: "broadcast",
      page: page,
      slideTitle: slideTitle,
      feedback: emojiKey,
    };
    webSocket.send(JSON.stringify(reactionState));
    addAnimatedEmoji(emoji);
  }
}

// send slide updates
function broadcastSlideUpdate(currentPage: number, clicks: number) {
  if (connectState.value === ConnectionStatus.CONNECTED) {
    const slideUpdateState: SlideUpdateState = {
      mtype: SendType.SLIDEUPDATE,
      page: currentPage,
      click: clicks,
      type: "broadcast",
    };
    webSocket.send(JSON.stringify(slideUpdateState));
  }
}

// Add animated emoji to the screen
function addAnimatedEmoji(emoji: string) {
  const x = Math.random() * (window.innerWidth - 50);
  const delay = Math.random() * 500; // Random delay for natural effect
  const newEmoji = { id: emojiCounter++, emoji, x, delay };
  animatedEmojis.value.push(newEmoji);
  setTimeout(() => {
    animatedEmojis.value = animatedEmojis.value.filter(
      (e) => e.id !== newEmoji.id
    );
  }, 3000); // Increased to account for animation duration
}

function getEmoji() {
  const reactions = configs.liveReactions || {};
  const okay = reactions.okay || "👀";
  const good = reactions.good || "👍";
  const great = reactions.great || "❤️";
  const mindBlown = reactions.mindBlown || "🤯";
  return {
    okay,
    good,
    great,
    mindBlown,
  };
}

onMounted(() => {
  if (isPresenter.value) {
    return;
  }
  initWebSocket();
});

watch([currentPage, queryClicks], (newPage) => {
  if (isPresenter.value) {
    broadcastSlideUpdate(currentPage.value, queryClicks.value);
  }
});

onUnmounted(() => {
  closeWebSocket();
});
</script>

<template>
  <div v-if="!isPresenter" class="reaction-buttons">
    <button
      v-for="(emoji, key) in getEmoji()"
      :key="key"
      @click="sendReaction(key)"
    >
      {{ emoji }}
    </button>
  </div>
  <div class="emoji-container">
    <div
      v-for="emoji in animatedEmojis"
      :key="emoji.id"
      class="animated-emoji"
      :style="{ left: `${emoji.x}px`, animationDelay: `${emoji.delay}ms` }"
    >
      {{ emoji.emoji }}
    </div>
  </div>
</template>

<style scoped>
.reaction-buttons {
  position: fixed;
  bottom: 20px;
  right: 20px;
  z-index: 1000;
}

.reaction-buttons button {
  margin: 0 5px;
  font-size: 24px;
  background: none;
  border: none;
  cursor: pointer;
}

.emoji-container {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  height: 100vh;
  pointer-events: none;
  overflow: hidden;
}

.animated-emoji {
  position: absolute;
  bottom: -50px;
  font-size: 2rem;
  animation: floatUp 3s ease-out forwards;
}

@keyframes floatUp {
  0% {
    transform: translateY(0);
    opacity: 0;
  }
  10% {
    opacity: 0.5;
  }
  100% {
    transform: translateY(-100vh);
    opacity: 0;
  }
}
</style>
