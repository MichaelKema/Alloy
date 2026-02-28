<script setup>
import { computed, onBeforeUnmount, ref } from "vue";
import jsmediatags from "jsmediatags/dist/jsmediatags.min.js";

const fileInputEl = ref(null);
const audioEl = ref(null);

const fileInputMode = ref("playlist");

const tracksById = ref({});
const playlists = ref([]);
const activePlaylistId = ref(null);
const currentTrackId = ref(null);
const libraryTrackIds = ref([]);

const audioSrc = ref("");
const isPlaying = ref(false);
const currentTime = ref(0);
const duration = ref(0);
const isSeeking = ref(false);
const isShuffling = ref(false);
const playOrder = ref([]);
const orderPosition = ref(-1);

const newPlaylistName = ref("");
const renamePlaylistName = ref("");

let trackSeq = 1;
let playlistSeq = 1;

const mediaTags = jsmediatags?.default ?? jsmediatags;

const activePlaylist = computed(
  () =>
    playlists.value.find(
      (playlist) => playlist.id === activePlaylistId.value,
    ) || null,
);

const activeTrackIds = computed(() => activePlaylist.value?.trackIds ?? []);

const activeTracks = computed(() =>
  activeTrackIds.value.map((id) => tracksById.value[id]).filter(Boolean),
);

const libraryTracks = computed(() =>
  libraryTrackIds.value.map((id) => tracksById.value[id]).filter(Boolean),
);

const currentTrack = computed(() =>
  currentTrackId.value ? tracksById.value[currentTrackId.value] : null,
);

const nowPlaying = computed(() =>
  currentTrack.value
    ? currentTrack.value.title || currentTrack.value.name
    : "No track loaded",
);

function openFileDialog(mode = "playlist") {
  fileInputMode.value = mode;
  fileInputEl.value?.click();
}

function createPlaylist(name) {
  const finalName = name?.trim() || `Playlist ${playlistSeq}`;
  const playlist = {
    id: `pl_${playlistSeq}`,
    name: finalName,
    trackIds: [],
  };
  playlistSeq += 1;
  playlists.value = [...playlists.value, playlist];

  if (!activePlaylistId.value) {
    activePlaylistId.value = playlist.id;
  }

  return playlist;
}

function createPlaylistFromInput() {
  const name = newPlaylistName.value.trim();
  const playlist = createPlaylist(name || `Playlist ${playlistSeq}`);
  newPlaylistName.value = "";
  setActivePlaylist(playlist.id);
}

function updatePlaylist(id, patch) {
  const index = playlists.value.findIndex((playlist) => playlist.id === id);
  if (index === -1) return;
  const updated = { ...playlists.value[index], ...patch };
  playlists.value.splice(index, 1, updated);
}

function renameActivePlaylist() {
  if (!activePlaylist.value) return;
  const name = renamePlaylistName.value.trim();
  if (!name) return;
  updatePlaylist(activePlaylist.value.id, { name });
  renamePlaylistName.value = "";
}

function deleteActivePlaylist() {
  if (playlists.value.length <= 1 || !activePlaylist.value) return;
  const removedPlaylist = activePlaylist.value;
  const removedTrackIds = [...removedPlaylist.trackIds];
  const nextList = playlists.value.filter(
    (playlist) => playlist.id !== removedPlaylist.id,
  );
  playlists.value = nextList;
  activePlaylistId.value = nextList[0]?.id ?? null;

  const orphaned = removedTrackIds.filter(
    (trackId) =>
      !nextList.some((playlist) => playlist.trackIds.includes(trackId)),
  );
  addTracksToLibrary(orphaned);

  if (!activePlaylistId.value) {
    clearCurrentTrack();
    return;
  }

  if (!activeTrackIds.value.includes(currentTrackId.value)) {
    if (activeTrackIds.value.length) {
      setCurrentTrackById(activeTrackIds.value[0], {
        shouldPlay: false,
        rebuildOrder: true,
      });
    } else {
      clearCurrentTrack();
    }
  }
}

function setActivePlaylist(id) {
  if (!id || id === activePlaylistId.value) return;
  activePlaylistId.value = id;
  rebuildPlayOrder();

  if (activeTrackIds.value.length) {
    setCurrentTrackById(activeTrackIds.value[0], {
      shouldPlay: false,
      rebuildOrder: false,
      skipOrderSync: true,
    });
  } else {
    clearCurrentTrack();
  }
}

function createTrack(file) {
  return {
    id: `trk_${trackSeq++}`,
    file,
    name: file.name,
    url: URL.createObjectURL(file),
    title: "",
    artist: "",
    album: "",
    coverUrl: "",
    duration: 0,
  };
}

function addTracksToLibrary(trackIds) {
  if (!trackIds?.length) return;
  const next = [...libraryTrackIds.value];
  trackIds.forEach((trackId) => {
    if (!next.includes(trackId)) next.push(trackId);
  });
  libraryTrackIds.value = next;
}

function removeTrackFromLibrary(trackId) {
  libraryTrackIds.value = libraryTrackIds.value.filter((id) => id !== trackId);
}

function isTrackInAnyPlaylist(trackId) {
  return playlists.value.some((playlist) =>
    playlist.trackIds.includes(trackId),
  );
}

function addFiles(fileList, { addToPlaylist = true } = {}) {
  const files = Array.from(fileList || []);
  if (!files.length) return;

  const newTracks = files.map(createTrack);
  const trackMap = { ...tracksById.value };
  newTracks.forEach((track) => {
    trackMap[track.id] = track;
  });
  tracksById.value = trackMap;

  if (addToPlaylist) {
    const playlist = activePlaylist.value || createPlaylist("My Playlist");
    if (activePlaylistId.value !== playlist.id) {
      activePlaylistId.value = playlist.id;
    }

    const nextTrackIds = [
      ...playlist.trackIds,
      ...newTracks.map((track) => track.id),
    ];
    updatePlaylist(playlist.id, { trackIds: nextTrackIds });
  } else {
    addTracksToLibrary(newTracks.map((track) => track.id));
  }

  newTracks.forEach((track) => {
    readMetadataForTrack(track.id, track.file);
  });

  if (!currentTrackId.value) {
    setCurrentTrackById(newTracks[0].id, {
      shouldPlay: true,
      rebuildOrder: addToPlaylist,
    });
  } else {
    rebuildPlayOrder();
  }
}

function onFileSelected(event) {
  addFiles(event.target.files, {
    addToPlaylist: fileInputMode.value !== "library",
  });
  event.target.value = "";
}

function updateTrack(trackId, patch) {
  const existing = tracksById.value[trackId];
  if (!existing) return;

  if (
    "coverUrl" in patch &&
    existing.coverUrl &&
    existing.coverUrl !== patch.coverUrl
  ) {
    URL.revokeObjectURL(existing.coverUrl);
  }

  tracksById.value = {
    ...tracksById.value,
    [trackId]: { ...existing, ...patch },
  };
}

function readMetadataForTrack(trackId, file) {
  try {
    new mediaTags.Reader(file).read({
      onSuccess: ({ tags }) => {
        const next = {
          title: tags?.title || "",
          artist: tags?.artist || "",
          album: tags?.album || "",
        };

        const picture = tags?.picture;
        if (picture?.data?.length) {
          const byteArray = new Uint8Array(picture.data);
          const blob = new Blob([byteArray], {
            type: picture.format || "image/jpeg",
          });
          next.coverUrl = URL.createObjectURL(blob);
        }

        updateTrack(trackId, next);
      },
      onError: () => {
        updateTrack(trackId, { title: "", artist: "", album: "" });
      },
    });
  } catch {
    updateTrack(trackId, { title: "", artist: "", album: "" });
  }
}

function shuffleArray(array) {
  for (let i = array.length - 1; i > 0; i -= 1) {
    const j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
  return array;
}

function rebuildPlayOrder() {
  const trackIds = activeTrackIds.value;
  if (!trackIds.length) {
    if (currentTrackId.value) {
      playOrder.value = [currentTrackId.value];
      orderPosition.value = 0;
    } else {
      playOrder.value = [];
      orderPosition.value = -1;
    }
    return;
  }

  if (!isShuffling.value) {
    playOrder.value = [...trackIds];
    orderPosition.value = currentTrackId.value
      ? playOrder.value.indexOf(currentTrackId.value)
      : -1;
    return;
  }

  const current = currentTrackId.value;
  const rest = trackIds.filter((id) => id !== current);
  shuffleArray(rest);
  playOrder.value = current ? [current, ...rest] : rest;
  orderPosition.value = current ? 0 : -1;
}

function syncOrderPosition() {
  if (!playOrder.value.length) {
    if (currentTrackId.value) {
      playOrder.value = [currentTrackId.value];
      orderPosition.value = 0;
      return;
    }
    orderPosition.value = -1;
    return;
  }
  const pos = playOrder.value.indexOf(currentTrackId.value);
  if (pos >= 0) {
    orderPosition.value = pos;
    return;
  }
  rebuildPlayOrder();
}

function clearCurrentTrack() {
  currentTrackId.value = null;
  audioSrc.value = "";
  currentTime.value = 0;
  duration.value = 0;
  if (audioEl.value) {
    audioEl.value.pause();
  }
  isPlaying.value = false;
  rebuildPlayOrder();
}

function setCurrentTrackById(
  trackId,
  { shouldPlay = true, rebuildOrder = false, skipOrderSync = false } = {},
) {
  const track = tracksById.value[trackId];
  if (!track) return;

  currentTrackId.value = trackId;
  audioSrc.value = track.url;
  currentTime.value = 0;
  duration.value = track.duration || 0;

  if (rebuildOrder) {
    rebuildPlayOrder();
  } else if (!skipOrderSync) {
    syncOrderPosition();
  }

  if (shouldPlay) {
    requestPlay();
  }
}

function selectTrack(trackId) {
  setCurrentTrackById(trackId, {
    shouldPlay: true,
    rebuildOrder: isShuffling.value,
  });
}

function toggleShuffle() {
  isShuffling.value = !isShuffling.value;
  rebuildPlayOrder();
}

function playNext() {
  if (!playOrder.value.length) return;
  if (orderPosition.value < 0) {
    setCurrentTrackById(playOrder.value[0], {
      shouldPlay: true,
      skipOrderSync: true,
    });
    orderPosition.value = 0;
    return;
  }

  let nextPos = orderPosition.value + 1;
  if (nextPos >= playOrder.value.length) nextPos = 0;
  const nextId = playOrder.value[nextPos];
  setCurrentTrackById(nextId, { shouldPlay: true, skipOrderSync: true });
  orderPosition.value = nextPos;
}

function playPrevious() {
  if (!playOrder.value.length) return;
  if (orderPosition.value < 0) {
    setCurrentTrackById(playOrder.value[0], {
      shouldPlay: true,
      skipOrderSync: true,
    });
    orderPosition.value = 0;
    return;
  }

  let prevPos = orderPosition.value - 1;
  if (prevPos < 0) prevPos = playOrder.value.length - 1;
  const prevId = playOrder.value[prevPos];
  setCurrentTrackById(prevId, { shouldPlay: true, skipOrderSync: true });
  orderPosition.value = prevPos;
}

function removeFromActivePlaylist(trackId) {
  if (!activePlaylist.value) return;
  const nextTrackIds = activePlaylist.value.trackIds.filter(
    (id) => id !== trackId,
  );
  updatePlaylist(activePlaylist.value.id, { trackIds: nextTrackIds });
  if (!isTrackInAnyPlaylist(trackId)) {
    addTracksToLibrary([trackId]);
  }

  if (currentTrackId.value === trackId) {
    if (nextTrackIds.length) {
      setCurrentTrackById(nextTrackIds[0], {
        shouldPlay: false,
        rebuildOrder: true,
      });
    } else {
      clearCurrentTrack();
    }
  } else {
    rebuildPlayOrder();
  }
}

function addTrackToActivePlaylist(trackId) {
  if (!trackId) return;
  const playlist = activePlaylist.value || createPlaylist("My Playlist");
  if (activePlaylistId.value !== playlist.id) {
    activePlaylistId.value = playlist.id;
  }
  if (!playlist.trackIds.includes(trackId)) {
    updatePlaylist(playlist.id, { trackIds: [...playlist.trackIds, trackId] });
  }
  removeTrackFromLibrary(trackId);
  rebuildPlayOrder();
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
  if (!audioEl.value || !currentTrackId.value) return;
  const trackDuration = audioEl.value.duration || 0;
  duration.value = trackDuration;
  updateTrack(currentTrackId.value, { duration: trackDuration });
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
  playNext();
}

onBeforeUnmount(() => {
  Object.values(tracksById.value).forEach((track) => {
    if (track.url) URL.revokeObjectURL(track.url);
    if (track.coverUrl) URL.revokeObjectURL(track.coverUrl);
  });
});
</script>

<template>
  <div class="app">
    <header class="header">
      <div class="brand">Alloy.fm</div>
      <nav class="menu" aria-label="App menu">
        <button
          class="menu-btn"
          type="button"
          @click="openFileDialog('playlist')"
        >
          File ▾
        </button>
        <button
          class="menu-btn"
          type="button"
          @click="openFileDialog('library')"
        >
          Library ▾
        </button>
      </nav>
    </header>

    <main class="content">
      <div class="playlist-manager">
        <div class="playlist-row">
          <div class="playlist-heading">Playlists</div>
          <div class="playlist-tabs">
            <button
              v-for="playlist in playlists"
              :key="playlist.id"
              class="playlist-tab"
              :class="{
                'playlist-tab--active': playlist.id === activePlaylistId,
              }"
              type="button"
              @click="setActivePlaylist(playlist.id)"
            >
              <span>{{ playlist.name }}</span>
              <span class="playlist-badge">{{ playlist.trackIds.length }}</span>
            </button>
          </div>
        </div>
        <div class="playlist-row">
          <input
            v-model="newPlaylistName"
            class="playlist-input"
            type="text"
            placeholder="New playlist name"
          />
          <button
            class="menu-btn"
            type="button"
            @click="createPlaylistFromInput"
          >
            Create
          </button>
          <input
            v-model="renamePlaylistName"
            class="playlist-input"
            type="text"
            :disabled="!activePlaylist"
            placeholder="Rename active"
          />
          <button
            class="menu-btn"
            type="button"
            :disabled="!activePlaylist"
            @click="renameActivePlaylist"
          >
            Rename
          </button>
          <button
            class="menu-btn"
            type="button"
            :disabled="playlists.length <= 1"
            @click="deleteActivePlaylist"
          >
            Delete
          </button>
        </div>
      </div>

      <p>Now playing: {{ nowPlaying }}</p>

      <div class="meta-card" v-if="currentTrack">
        <div class="cover" :class="{ 'cover--empty': !currentTrack.coverUrl }">
          <img
            v-if="currentTrack.coverUrl"
            :src="currentTrack.coverUrl"
            alt="Album art"
          />
          <div v-else class="cover-placeholder">♪</div>
        </div>
        <div class="meta-text">
          <div class="meta-title">{{ currentTrack.title || nowPlaying }}</div>
          <div class="meta-list">
            <div class="meta-item" v-if="currentTrack.artist">
              <svg class="meta-icon" viewBox="0 0 24 24" aria-hidden="true">
                <path
                  d="M12 12a4 4 0 1 0-4-4 4 4 0 0 0 4 4Zm0 2c-4.4 0-8 2-8 4.5V20h16v-1.5c0-2.5-3.6-4.5-8-4.5Z"
                  fill="currentColor"
                />
              </svg>
              <span class="meta-value">{{ currentTrack.artist }}</span>
            </div>
            <div class="meta-item" v-if="currentTrack.album">
              <svg class="meta-icon" viewBox="0 0 24 24" aria-hidden="true">
                <path
                  d="M12 3a9 9 0 1 0 9 9 9 9 0 0 0-9-9Zm0 4a5 5 0 1 1-5 5 5 5 0 0 1 5-5Zm0 3a2 2 0 1 0 2 2 2 2 0 0 0-2-2Z"
                  fill="currentColor"
                />
              </svg>
              <span class="meta-value">{{ currentTrack.album }}</span>
            </div>
          </div>
        </div>
      </div>

      <div class="playlist" v-if="libraryTracks.length">
        <div class="playlist-header">
          <div class="playlist-title">Library</div>
          <div class="playlist-count">
            {{ libraryTracks.length }} track{{
              libraryTracks.length === 1 ? "" : "s"
            }}
          </div>
        </div>
        <div class="playlist-list">
          <div
            v-for="track in libraryTracks"
            :key="track.id"
            class="track"
            role="button"
            tabindex="0"
            :class="{ 'track--active': track.id === currentTrackId }"
            @click="selectTrack(track.id)"
            @keydown.enter="selectTrack(track.id)"
          >
            <div class="track-cover">
              <img v-if="track.coverUrl" :src="track.coverUrl" alt="" />
              <div v-else class="track-cover-placeholder">♪</div>
            </div>
            <div class="track-info">
              <div class="track-title">{{ track.title || track.name }}</div>
              <div class="track-meta">
                <span>{{ track.artist || "Unknown artist" }}</span>
                <span v-if="track.album">• {{ track.album }}</span>
              </div>
            </div>
            <div class="track-time">{{ formatTime(track.duration) }}</div>
            <button
              class="track-action track-action--add"
              type="button"
              @click.stop="addTrackToActivePlaylist(track.id)"
            >
              Add to playlist
            </button>
          </div>
        </div>
      </div>

      <div class="playlist" v-if="activeTracks.length">
        <div class="playlist-header">
          <div class="playlist-title">
            {{ activePlaylist?.name || "Playlist" }}
          </div>
          <div class="playlist-count">
            {{ activeTracks.length }} track{{
              activeTracks.length === 1 ? "" : "s"
            }}
          </div>
        </div>
        <div class="playlist-list">
          <div
            v-for="track in activeTracks"
            :key="track.id"
            class="track"
            role="button"
            tabindex="0"
            :class="{ 'track--active': track.id === currentTrackId }"
            @click="selectTrack(track.id)"
            @keydown.enter="selectTrack(track.id)"
          >
            <div class="track-cover">
              <img v-if="track.coverUrl" :src="track.coverUrl" alt="" />
              <div v-else class="track-cover-placeholder">♪</div>
            </div>
            <div class="track-info">
              <div class="track-title">{{ track.title || track.name }}</div>
              <div class="track-meta">
                <span>{{ track.artist || "Unknown artist" }}</span>
                <span v-if="track.album">• {{ track.album }}</span>
              </div>
            </div>
            <div class="track-time">{{ formatTime(track.duration) }}</div>
            <button
              class="track-remove"
              type="button"
              @click.stop="removeFromActivePlaylist(track.id)"
            >
              Remove
            </button>
          </div>
        </div>
      </div>
      <p v-else-if="!libraryTracks.length" class="empty-state">
        Add audio files to the active playlist.
      </p>

      <input
        ref="fileInputEl"
        type="file"
        accept="audio/*"
        multiple
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
        <button class="ctrl" aria-label="Previous" @click="playPrevious">
          ⏮
        </button>
        <button
          class="ctrl ctrl--primary"
          :aria-label="isPlaying ? 'Pause' : 'Play'"
          @click="togglePlay"
        >
          {{ isPlaying ? "⏸" : "▶" }}
        </button>
        <button class="ctrl" aria-label="Next" @click="playNext">⏭</button>
        <button
          class="ctrl ctrl--secondary"
          :class="{ 'ctrl--active': isShuffling }"
          aria-label="Shuffle"
          @click="toggleShuffle"
        >
          ⇄
        </button>
      </div>
    </div>
  </div>
</template>

<style src="./style/style.css"></style>
