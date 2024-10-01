<template>
  <div class="canvas-container">
    <canvas ref="canvas" @click="getPixelColor" @mousemove="getPixelColor"></canvas>
    <div v-if="pixelInfo" class="padding">
      <p>Цвет: rgb({{ pixelInfo.color.r }}, {{ pixelInfo.color.g }}, {{ pixelInfo.color.b }})</p>
      <p>Координаты: x: {{ pixelInfo.x }}, y: {{ pixelInfo.y }}</p>
      <p>Размер изображения: ширина: {{ imageWidth }}px, высота: {{ imageHeight }}px</p>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    imageWidth: {
      type: Number,
      required: true
    },
    imageHeight: {
      type: Number,
      required: true
    }
  },
  data() {
    return {
      pixelInfo: null
    };
  },
  methods: {
    getPixelColor(event) {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext('2d');
      const rect = canvas.getBoundingClientRect();
      const x = Math.floor((event.clientX - rect.left) * (canvas.width / rect.width));
      const y = Math.floor((event.clientY - rect.top) * (canvas.height / rect.height));
      const pixel = ctx.getImageData(x, y, 1, 1).data;
      this.pixelInfo = {
        color: {
          r: pixel[0],
          g: pixel[1],
          b: pixel[2]
        },
        x: x,
        y: y
      };
    },
    getPixelColorAtCoordinates(x, y) {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext('2d');
      const pixel = ctx.getImageData(x, y, 1, 1).data;
      return {
        r: pixel[0],
        g: pixel[1],
        b: pixel[2]
      };
    }
  }
};
</script>

<style>
.canvas-container {
  width: 100%;
  height: 100vh;
  overflow: auto;
}
canvas {
  display: block;
}
.padding {
  padding-left: 50px;
}
</style>