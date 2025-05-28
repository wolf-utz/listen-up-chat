<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from "vue";

const props = defineProps({
  audioUrl: {
    type: String,
    required: true,
  },
  initialPlaybackRate: {
    type: Number,
    default: 1.0,
  },
});

const emit = defineEmits(["play", "pause", "timeupdate", "ended"]);

const audioElement = new Audio();
const isPlaying = ref(false);
const audioProgress = ref(0);
const isDragging = ref(false);
const progressBar = ref<HTMLElement | null>(null);

audioElement.playbackRate = props.initialPlaybackRate;
const playbackRate = ref(props.initialPlaybackRate);
const showSpeedOptions = ref(false);
const speedOptions = [0.5, 0.75, 1.0, 1.25, 1.5];
const audioDuration = ref(0);

// Handle progress bar dragging
const startDrag = (e: MouseEvent) => {
  isDragging.value = true;
  updateProgressFromEvent(e);
};

const handleDrag = (e: MouseEvent) => {
  if (!isDragging.value) return;
  e.preventDefault();
  updateProgressFromEvent(e);
};

const stopDrag = () => {
  isDragging.value = false;
};

const updateProgressFromEvent = (e: MouseEvent) => {
  if (!progressBar.value) return;

  const rect = progressBar.value.getBoundingClientRect();
  const position = (e.clientX - rect.left) / rect.width;
  const newPosition = Math.max(0, Math.min(1, position));

  audioProgress.value = newPosition * 100;

  if (audioElement.duration) {
    audioElement.currentTime = newPosition * audioElement.duration;
  }
};

// Get full audio URL
const getFullAudioUrl = (url: string) => {
  // If URL is already absolute, return as is
  if (url.startsWith("http")) return url;
  // Otherwise, prepend the base URL
  return `${import.meta.env.VITE_API_BASE_URL}${url}`;
};

// Handle play/pause toggle
const togglePlayPause = () => {
  if (!props.audioUrl) return;

  if (isPlaying.value) {
    audioElement.pause();
    isPlaying.value = false;
    emit("pause");
  } else {
    // Make sure we have the full URL before playing
    if (audioElement.src !== getFullAudioUrl(props.audioUrl)) {
      audioElement.src = getFullAudioUrl(props.audioUrl);
    }
    audioElement.play().catch(console.error);
    isPlaying.value = true;
    emit("play");
  }
};

// Set playback speed
const setPlaybackSpeed = (speed: number) => {
  playbackRate.value = speed;
  audioElement.playbackRate = speed;
  showSpeedOptions.value = false;
};

// Close speed options when clicking outside
const handleClickOutside = (event: MouseEvent) => {
  const target = event.target as HTMLElement;
  if (!target.closest(".playback-speed-container")) {
    showSpeedOptions.value = false;
  }
};

// Format time in MM:SS format with leading zeros
const formatTime = (seconds: number) => {
  if (isNaN(seconds)) return "00:00";
  const mins = Math.floor(seconds / 60);
  const secs = Math.floor(seconds % 60);
  return `${mins.toString().padStart(2, "0")}:${secs
    .toString()
    .padStart(2, "0")}`;
};

// Handle progress bar click
const handleProgressClick = (e: MouseEvent) => {
  if (!audioElement.duration || isDragging.value) return;

  const rect = (e.currentTarget as HTMLElement).getBoundingClientRect();
  const clickPosition = (e.clientX - rect.left) / rect.width;
  const newTime = clickPosition * audioElement.duration;

  audioElement.currentTime = newTime;
  audioProgress.value = (newTime / audioElement.duration) * 100;

  // Force update the progress immediately
  if (audioElement.paused) {
    updateProgress();
  }
};

// Update progress bar
const updateProgress = () => {
  console.log("Updating progress...", {
    currentTime: audioElement.currentTime,
    duration: audioElement.duration,
    readyState: audioElement.readyState,
    buffered:
      audioElement.buffered.length > 0 ? audioElement.buffered.end(0) : 0,
  });

  if (
    audioElement.duration &&
    isFinite(audioElement.duration) &&
    audioElement.duration > 0
  ) {
    const progress = (audioElement.currentTime / audioElement.duration) * 100;
    console.log("Progress calculated:", progress + "%");
    audioProgress.value = Math.min(100, Math.max(0, progress));
    emit("timeupdate", {
      currentTime: audioElement.currentTime,
      duration: audioElement.duration,
    });
  }
};

// Event listeners
const initAudio = () => {
  audioElement.src = props.audioUrl;
  audioElement.load();

  audioElement.addEventListener("timeupdate", updateProgress);
  audioElement.addEventListener("loadedmetadata", () => {
    audioDuration.value = audioElement.duration;
  });
  audioElement.addEventListener("ended", () => {
    isPlaying.value = false;
    emit("ended");
  });
};

// Cleanup
const cleanup = () => {
  audioElement.pause();
  audioElement.removeEventListener("timeupdate", updateProgress);
  audioElement.removeEventListener("ended", () => emit("ended"));
};

// Reset audio to beginning
const resetAudio = () => {
  if (audioElement) {
    audioElement.currentTime = 0;
    audioProgress.value = 0;
    if (isPlaying.value) {
      audioElement.play().catch((error) => {
        console.error("Error playing audio after reset:", error);
      });
    }
  }
};

// Watch for URL changes
watch(
  () => props.audioUrl,
  (newUrl) => {
    if (newUrl) {
      console.log("URL changed to:", newUrl);
      cleanup();
      const fullUrl = getFullAudioUrl(newUrl);
      console.log("Setting audio src to:", fullUrl);
      audioElement.src = fullUrl;

      // Reset progress when URL changes
      audioProgress.value = 0;
      audioDuration.value = 0;

      // Add event listeners after setting the src
      initAudio();

      // Load the audio
      audioElement.load();
    }
  },
  { immediate: true }
);

// Component lifecycle
onMounted(() => {
  document.addEventListener("click", handleClickOutside);
  initAudio();
});

onUnmounted(() => {
  cleanup();
  document.removeEventListener("click", handleClickOutside);
  audioElement.pause();
});
</script>

<template>
  <div class="audio-player-container">
    <div class="audio-controls">
      <div class="playback-controls">
        <button @click="togglePlayPause" class="play-pause-btn">
          <svg
            v-if="!isPlaying"
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <polygon points="5 3 19 12 5 21 5 3"></polygon>
          </svg>
          <svg
            v-else
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <rect x="6" y="4" width="4" height="16"></rect>
            <rect x="14" y="4" width="4" height="16"></rect>
          </svg>
        </button>
        <button
          @click="resetAudio"
          class="reset-btn"
          title="Reset to beginning"
        >
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="20"
            height="20"
            viewBox="0 0 24 24"
            fill="none"
            stroke="#5f6368"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path d="M1 4v6h6"></path>
            <path d="M3.51 15a9 9 0 1 0 2.13-9.36L1 10"></path>
          </svg>
        </button>
      </div>

      <!-- Playback speed control -->
      <div class="playback-speed-container">
        <button
          @click="showSpeedOptions = !showSpeedOptions"
          class="speed-btn"
          :class="{ active: showSpeedOptions }"
          title="Playback speed"
        >
          {{ playbackRate }}x
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="16"
            height="16"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <polyline points="6 9 12 15 18 9"></polyline>
          </svg>
        </button>

        <div v-if="showSpeedOptions" class="speed-options" @click.stop>
          <button
            v-for="speed in speedOptions"
            :key="speed"
            @click="setPlaybackSpeed(speed)"
            :class="{ active: playbackRate === speed }"
          >
            {{ speed }}x
          </button>
        </div>
      </div>
    </div>

    <div class="progress-container">
      <div
        class="progress-bar"
        @click="handleProgressClick"
        @mousedown="startDrag"
        @mousemove="handleDrag"
        @mouseup="stopDrag"
        @mouseleave="isDragging && stopDrag()"
        ref="progressBar"
      >
        <div class="progress" :style="{ width: `${audioProgress}%` }">
          <div class="progress-knob" :class="{ dragging: isDragging }"></div>
        </div>
      </div>
      <div class="time-display">
        <span>{{ formatTime(audioElement.currentTime || 0) }}</span>
      </div>
    </div>

    <div class="audio-player-hint">
      <svg
        xmlns="http://www.w3.org/2000/svg"
        width="14"
        height="14"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <circle cx="12" cy="12" r="10"></circle>
        <line x1="12" y1="16" x2="12" y2="12"></line>
        <line x1="12" y1="8" x2="12.01" y2="8"></line>
      </svg>
      <span>Höre dir die Geschichte an</span>
    </div>
  </div>
</template>

<style scoped>
/* Reset Button */
.reset-button {
  background: none;
  border: none;
  color: #5f6368;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  cursor: pointer;
  opacity: 0;
  transition: all 0.2s;
  visibility: hidden;
}

.reset-button.is-visible,
.audio-player:hover .reset-button {
  opacity: 1;
  visibility: visible;
}

.reset-button:hover {
  background: #f1f3f4;
  color: #202124;
}

.reset-button svg {
  width: 16px;
  height: 16px;
}
.audio-player-container {
  position: relative;
  margin-top: 8px;
  background: #f8f9fa;
  border-radius: 12px;
  padding: 12px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}

.audio-controls {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 8px;
  width: 100%;
}

.playback-controls {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
}

.reset-btn {
  background: none;
  border: none;
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  cursor: pointer;
  transition: all 0.2s;
  padding: 0;
  margin: 0;
  outline: none;
}

.reset-btn:hover {
  background: #f1f3f4;
}

.reset-btn:active {
  transform: scale(0.92);
}

.reset-btn svg {
  width: 20px;
  height: 20px;
}

.progress-container {
  flex: 1;
  min-width: 0;
  margin: 0 8px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

/* Progress Bar */
.progress-bar {
  position: relative;
  z-index: 1;
  cursor: pointer;
  user-select: none;
  -webkit-tap-highlight-color: transparent;
  height: 4px;
  background: #e0e0e0;
  border-radius: 2px;
  width: 100%;
  margin: 10px 0;
  overflow: visible;
  transition: height 0.2s ease;
}

.progress-bar:hover {
  height: 6px;
}

/* Progress Bar - Filled Portion */
.progress {
  position: absolute;
  top: 0;
  left: 0;
  bottom: 0;
  background: #1a73e8;
  border-radius: 2px;
  transition: width 0.1s linear;
  min-width: 4px;
  will-change: width;
  display: flex;
  justify-content: flex-end;
  align-items: center;
  z-index: 2;
  pointer-events: none;
}

/* Progress Knob */
.progress-knob {
  width: 16px;
  height: 16px;
  background: #1a73e8;
  border: 2px solid white;
  border-radius: 50%;
  margin-right: -8px;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.3);
  cursor: pointer;
  position: relative;
  z-index: 3;
  opacity: 0;
  transition: all 0.2s ease;
  pointer-events: auto;
}

/* Progress Bar States */
.progress-bar:hover .progress-knob,
.progress-knob.dragging {
  opacity: 1;
  transform: scale(1.2);
}

.progress-bar:active .progress-knob,
.progress-knob.dragging:active {
  transform: scale(1.3);
}

.progress-debug {
  position: absolute;
  right: -30px;
  top: -20px;
  font-size: 10px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 2px 4px;
  border-radius: 3px;
  white-space: nowrap;
}

.time-display {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  color: #5f6368;
  margin-top: 2px;
}

.playback-speed-container {
  position: relative;
  display: flex;
  align-items: center;
  margin-left: auto;
  flex-shrink: 0;
}

.speed-btn {
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 16px;
  padding: 4px 12px;
  font-size: 13px;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 4px;
  transition: all 0.2s;
  color: #3c4043;
  font-weight: 500;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
}

.speed-btn:hover {
  background: #f1f3f4;
  border-color: #dadce0;
}

.speed-btn.active {
  background: #e8f0fe;
  border-color: #d2e3fc;
  color: #1967d2;
}

.speed-options {
  position: absolute;
  bottom: calc(100% + 8px);
  left: 0;
  right: 0;
  background: white;
  border: 1px solid #dadce0;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  z-index: 100;
  min-width: 120px;
  padding: 4px;
  margin: 0;
  box-sizing: border-box;
}

.speed-options button {
  display: block;
  width: 100%;
  padding: 8px 16px;
  text-align: left;
  background: none;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.15s;
  font-size: 13px;
  font-family: inherit;
  color: #202124;
  white-space: nowrap;
  box-sizing: border-box;
  margin: 2px 0;
}

.speed-options button:hover {
  background: #f1f3f4;
  color: #1967d2;
}

.speed-options button.active {
  background: #e8f0fe;
  color: #1967d2;
  font-weight: 500;
}

.speed-options button:active {
  transform: scale(0.98);
}

.speed-options button.active {
  background: #e8f0fe;
  color: #1967d2;
  font-weight: 500;
}

.audio-player-hint {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: #5f6368;
  margin-top: 8px;
  padding: 0 8px;
}

.audio-player-hint svg {
  flex-shrink: 0;
}

.play-pause-btn {
  background: none;
  border: none;
  cursor: pointer;
  padding: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #1a73e8;
  border-radius: 50%;
  transition: background-color 0.2s;
}

.play-pause-btn:hover {
  background-color: rgba(26, 115, 232, 0.1);
}
</style>
