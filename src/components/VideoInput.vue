<template>
  <div class="output-wrapper">
    <PaintCanvas />
    <CanvasDisplay />
    <video id="video" playsinline></video>
    <div id="info">{{ errorMessage }}</div>
  </div>
</template>

<script setup lang="ts">
import * as handpose from '@tensorflow-models/handpose';
import * as tf from '@tensorflow/tfjs-core';
import * as tfjsWasm from '@tensorflow/tfjs-backend-wasm';
import { provide, ref, onMounted } from 'vue';
import { FingerPositions } from '../types/fingers';
import CanvasDisplay from './CanvasDisplay.vue';
import PaintCanvas from './PaintCanvas.vue';

tfjsWasm.setWasmPaths({
  'tfjs-backend-wasm.wasm': `https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-backend-wasm@${
        tfjsWasm.version_wasm}/dist/tfjs-backend-wasm.wasm`,
  'tfjs-backend-wasm-simd.wasm': `https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-backend-wasm@${
        tfjsWasm.version_wasm}/dist/tfjs-backend-wasm-simd.wasm`,
  'tfjs-backend-wasm-threaded-simd.wasm': `https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-backend-wasm@${
        tfjsWasm.version_wasm}/dist/tfjs-backend-wasm-threaded-simd.wasm`,
  });

  const fingers = ref({
    thumb: [0, 0],
    indexFinger: [0, 0],
    middleFinger: [0, 0],
    ringFinger: [0, 0],
    pinky: [0, 0]
  });

  const fingerTipLookupIndices: Record<string, number> = {
      thumb: 4,
      indexFinger: 8,
      middleFinger: 12,
      ringFinger: 16,
      pinky: 20
    };
  let model: handpose.HandPose;

const VIDEO_WIDTH = 640;
const VIDEO_HEIGHT = 500;

const errorMessage = ref('');

async function setupCamera(): Promise<HTMLVideoElement> {
  if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
    throw new Error(
        'Browser API navigator.mediaDevices.getUserMedia not available');
  }

  const video = document.getElementById('video') as HTMLVideoElement;
  const stream = await navigator.mediaDevices.getUserMedia({
    'audio': false,
    'video': {
      facingMode: 'user',
      // Only setting the video to a specified size in order to accommodate a
      // point cloud, so on mobile devices accept the default size.
      width: VIDEO_WIDTH,
      height: VIDEO_HEIGHT
    },
  });

  if (!stream) {
    throw new Error('No stream available');
  }

  video.srcObject = stream;

  return new Promise((resolve) => {
    video.onloadedmetadata = () => {
      resolve(video);
    };
  });
}

async function loadVideo(): Promise<HTMLVideoElement> {
  const video = await setupCamera();
  video.play();
  return video;
}

const landmarksRealTime = async (video: HTMLVideoElement) => {
  async function frameLandmarks() {
    const predictions = await model.estimateHands(video);
    if (predictions.length > 0) {
      const result = predictions[0].landmarks;
      const update = {
        thumb: [0, 0],
        indexFinger: [0, 0],
        middleFinger: [0, 0],
        ringFinger: [0, 0],
        pinky: [0, 0]
      };
      for (const finger in fingerTipLookupIndices) {
        const fingerProperties = fingerTipLookupIndices[finger];
        const y = result[fingerProperties][0];
        const x = result[fingerProperties][1];
        update[finger as keyof FingerPositions] = [x, y];
      }
      fingers.value = update;
    }

    requestAnimationFrame(frameLandmarks);
  };

  frameLandmarks();
};

onMounted(async () => {
  await tf.setBackend('wasm');
  
  model = await handpose.load();
  let video;

  try {
    video = await loadVideo();
  } catch (e) {
    errorMessage.value = (e as unknown as Error).message;
    throw e;
  }

  landmarksRealTime(video);
});

provide('fingerPositions', fingers);

</script>

<style>
.output-wrapper {
  position: relative;
  width: 640px;
  height: 500px;
}

#video {
  position: absolute;
  top: 0;
  left: 0;
  -webkit-transform: scaleX(-1);
    transform: scaleX(-1);
    visibility: hidden;
}
</style>