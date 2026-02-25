<script setup>
import { onBeforeUnmount, ref } from "vue";

const fileInputEl = ref(null);
const audioEl = ref(null);
const audioSrc = ref("");
const nowPlaying = ref("No track loaded");
const isPlaying = ref(false);
const currentTime = ref(0);
const duration = ref(0);
const isSeeking = ref(false);
let objectUrl = "";

function openFileDialog() {
  fileInputEl.value?.click();
}

function setSourceFromFile(file) {
  if (!file) return;
  if (objectUrl) {
    URL.revokeObjectURL(objectUrl);
  }
  objectUrl = URL.createObjectURL(file);
  audioSrc.value = objectUrl;
  nowPlaying.value = file.name;
  requestPlay();
}

function onFileSelected(event) {
  const file = event.target.files?.[0];
  setSourceFromFile(file);
  event.target.value = "";
}

function requestPlay() {
  if (!audioEl.value) return;
  audioEl.value.play().catch(() => {
    isPlaying.value = false;
  });
}

function togglePlay() {
  if (!audioEl.value) return;
  if (audioEl.value.paused) {
    requestPlay();
  } else {
    audioEl.value.pause();
  }
}

function onLoadedMetadata() {
  if (!audioEl.value) return;
  duration.value = audioEl.value.duration || 0;
}

function onTimeUpdate() {
  if (!audioEl.value) return;
  if (!isSeeking.value) {
    currentTime.value = audioEl.value.currentTime || 0;
  }
  duration.value = audioEl.value.duration || 0;
}

function onSeekInput(event) {
  isSeeking.value = true;
  currentTime.value = Number(event.target.value);
}

function onSeekCommit(event) {
  if (!audioEl.value) return;
  const nextTime = Number(event.target.value);
  audioEl.value.currentTime = nextTime;
  currentTime.value = nextTime;
  isSeeking.value = false;
}

function formatTime(value) {
  if (!Number.isFinite(value) || value <= 0) return "--:--";
  const total = Math.floor(value);
  const hours = Math.floor(total / 3600);
  const minutes = Math.floor((total % 3600) / 60);
  const seconds = total % 60;
  if (hours > 0) {
    return `${hours}:${String(minutes).padStart(2, "0")}:${String(seconds).padStart(2, "0")}`;
  }
  return `${minutes}:${String(seconds).padStart(2, "0")}`;
}

function onAudioPlay() {
  isPlaying.value = true;
}

function onAudioPause() {
  isPlaying.value = false;
}

function onAudioEnded() {
  isPlaying.value = false;
}

onBeforeUnmount(() => {
  if (objectUrl) {
    URL.revokeObjectURL(objectUrl);
  }
});
</script>

<template>
  <div class="app">
    <header class="header">
      <div class="brand">Alloy</div>
      <nav class="menu" aria-label="App menu">
        <button class="menu-btn" type="button" @click="openFileDialog">File ▾</button>
        <button class="menu-btn" type="button" aria-haspopup="true">Select ▾</button>
        <button class="menu-btn" type="button" aria-haspopup="true">View ▾</button>
      </nav>
    </header>

    <main class="content">
      <p>Now playing: {{ nowPlaying }}</p>

      <input
        ref="fileInputEl"
        type="file"
        accept="audio/*"
        @change="onFileSelected"
        hidden
      />
      <audio
        ref="audioEl"
        :src="audioSrc"
        @loadedmetadata="onLoadedMetadata"
        @timeupdate="onTimeUpdate"
        @play="onAudioPlay"
        @pause="onAudioPause"
        @ended="onAudioEnded"
      />
    </main>

    <div class="player-bar" role="group" aria-label="Media controls">
      <div class="scrubber-row">
        <span class="time time--current">{{ formatTime(currentTime) }}</span>
        <input
          class="scrubber"
          type="range"
          min="0"
          :max="duration"
          step="0.01"
          :value="currentTime"
          :disabled="!duration"
          aria-label="Seek"
          @input="onSeekInput"
          @change="onSeekCommit"
        />
        <span class="time time--duration">{{ formatTime(duration) }}</span>
      </div>
      <div class="player-controls">
        <button class="ctrl ctrl--secondary" aria-label="Loop">⟲</button>
        <button class="ctrl" aria-label="Previous">⏮</button>
        <button class="ctrl ctrl--primary" :aria-label="isPlaying ? 'Pause' : 'Play'" @click="togglePlay">
          {{ isPlaying ? "⏸" : "▶" }}
        </button>
        <button class="ctrl" aria-label="Next">⏭</button>
        <button class="ctrl ctrl--secondary" aria-label="Shuffle">⇄</button>
      </div>
    </div>
  </div>
</template>

<style src="./style/style.css">

</style>
