<template>
  <div id="paint">
  </div>
</template>

<script setup lang="ts">
import * as PIXI from 'pixi.js';
import { inject, onMounted, Ref } from 'vue';
import FastVector from 'fast-vector';
import { FingerPositions } from '../types/fingers';
import { fingerColors } from '../common/colors';

const fingerInput = inject<Ref<FingerPositions>>('fingerPositions');
let lastPosition = new FastVector(0, 0);

const VIDEO_WIDTH = 640;
const VIDEO_HEIGHT = 500;
const app = new PIXI.Application({ width: VIDEO_WIDTH, height: VIDEO_HEIGHT, transparent: true });
const graphics = new PIXI.Graphics();

function drawPoint(newPos: FastVector, c: number): void {
  graphics.lineStyle(6, c, 1);
  graphics.moveTo(lastPosition.y, lastPosition.x);
  graphics.lineTo(newPos.y, newPos.x);
  graphics.endFill();
}

function paintWithFingers(delta: number): void {
  const MIN_DISTANCE = 30;

  if (fingerInput) {
    let closestDistance = 999;
    const [thumbX, thumbY] = fingerInput.value.thumb;
    const thumbVector = new FastVector(thumbX, thumbY);

    for (const finger in fingerInput.value) {
      if (finger !== 'thumb') {
        const fingerPosition = fingerInput.value[finger as keyof FingerPositions];
        if (fingerPosition && fingerPosition.length > 0) {
          const x = fingerPosition[0];
          const y = fingerPosition[1];
          const fingerVector = new FastVector(x, y);
          const distance = thumbVector.distance(fingerVector);
          
          if (distance < closestDistance) {
            closestDistance = distance;

            if (closestDistance < MIN_DISTANCE) {
              drawPoint(fingerVector, fingerColors[finger]);
              lastPosition = fingerVector.clone();
            }
          }
        }
      }
    }
  }
}

const update = () => {
  app.ticker.add((delta) => paintWithFingers(delta));
};

function initCanvas() {
  document.getElementById('paint')?.appendChild(app.view);
  app.stage.addChild(graphics);
}

onMounted(() => {
  initCanvas();
  update();
});

</script>

<style>
#paint {
  position: absolute;
  width: 640px;
  height: 500px;
}
</style>