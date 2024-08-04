<template>
  <div class="chord-analyzer">
    <h1>Guitar Chord Analyzer</h1>
    <button @click="toggleRecording" :disabled="!isSupported">
      {{ isRecording ? 'Stop' : 'Start' }} Recording
    </button>
    <div v-if="currentChord" class="chord-display">
      <h2>Current Chord:</h2>
      <p class="chord">{{ currentChord }}</p>
    </div>
    <p v-if="!isSupported" class="error">
      Microphone access is not supported in this browser.
    </p>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import { SoundAnalyzer } from 'sound-analyzer';

const analyzer = new SoundAnalyzer();
const isRecording = ref(false);
const isSupported = ref(false);
const currentChord = ref('');

let audioContext: AudioContext | null = null;
let mediaStream: MediaStream | null = null;
let analyzerNode: AnalyserNode | null = null;
let animationFrame: number | null = null;

onMounted(() => {
  isSupported.value = !!navigator.mediaDevices && !!navigator.mediaDevices.getUserMedia;
});

onUnmounted(() => {
  stopRecording();
});

const toggleRecording = async () => {
  if (isRecording.value) {
    stopRecording();
  } else {
    await startRecording();
  }
};

const startRecording = async () => {
  try {
    audioContext = new (window.AudioContext || (window as any).webkitAudioContext)();
    mediaStream = await navigator.mediaDevices.getUserMedia({ audio: true });

    const sourceNode = audioContext.createMediaStreamSource(mediaStream);
    analyzerNode = audioContext.createAnalyser();
    analyzerNode.fftSize = 2048;
    sourceNode.connect(analyzerNode);

    isRecording.value = true;
    requestAnimationFrame(analyzeAudio);
  } catch (error) {
    console.error('Error accessing microphone:', error);
  }
};

const stopRecording = () => {
  if (mediaStream) {
    mediaStream.getTracks().forEach(track => track.stop());
  }
  if (audioContext) {
    audioContext.close();
  }
  if (animationFrame !== null) {
    cancelAnimationFrame(animationFrame);
  }
  isRecording.value = false;
  currentChord.value = '';
};

const analyzeAudio = () => {
  if (!analyzerNode) return;

  const bufferLength = analyzerNode.frequencyBinCount;
  const dataArray = new Float32Array(bufferLength);
  analyzerNode.getFloatTimeDomainData(dataArray);

  const { chord } = analyzer.analyze(dataArray);
  currentChord.value = chord;

  animationFrame = requestAnimationFrame(analyzeAudio);
};
</script>

<style scoped>
.chord-analyzer {
  text-align: center;
  padding: 20px;
}

button {
  font-size: 18px;
  padding: 10px 20px;
  margin: 20px 0;
}

.chord-display {
  margin-top: 20px;
}

.chord {
  font-size: 48px;
  font-weight: bold;
  color: #4CAF50;
}

.error {
  color: red;
}
</style>