<template>
  <div id="finger-input-display">
  </div>
</template>

<script setup lang="ts">
  import * as PIXI from 'pixi.js';
import { inject, onMounted, Ref, onUpdated } from 'vue';
import { FingerPositions } from '../types/fingers';
import { fingerColors } from '../common/colors';

const fingers = inject<Ref<FingerPositions>>('fingerPositions');
const VIDEO_WIDTH = 640;
const VIDEO_HEIGHT = 500;
const app = new PIXI.Application({ width: VIDEO_WIDTH, height: VIDEO_HEIGHT, transparent: true });
const graphics = new PIXI.Graphics();

function drawPoint(y: number, x: number, r: number, c: number, a: number): void {
  graphics.lineStyle(3, c, a);
  graphics.drawCircle(x, y, r);
  graphics.endFill();
}

function drawKeypoints(): void {
  graphics.clear();

  if (fingers) {
    for (const finger in fingers.value) {
      const fingerPosition = fingers.value[finger as keyof FingerPositions];
      if (fingerPosition && fingerPosition.length > 0) {
        const x = fingerPosition[0];
        const y = fingerPosition[1];
        drawPoint(x, y, 6, fingerColors[finger], finger === 'thumb' ? 0.25 : 0.65);
      }
    }
  }
}

const update = () => {
  app.ticker.add(drawKeypoints);
};

function initCanvas() {
  document.getElementById('finger-input-display')?.appendChild(app.view);
  app.stage.addChild(graphics);
}

onMounted(() => {
  initCanvas();
  update();
});


</script>

<style>
#finger-input-display {
  position: absolute;
  width: 640px;
  height: 500px;
}
</style>