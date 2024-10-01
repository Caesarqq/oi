<template>
  <div class="canvasmain">
    <div class="canvas-container">
      <canvas ref="pixelSearch"></canvas>
    </div>
    <image-settings
      ref="imageSettings"
      :resizeDialog="resizeDialog"
      :image="image"
      :imageWidth="imageWidth"
      :imageHeight="imageHeight"
      @update:resizeDialog="resizeDialog = $event"
      @resize="onResize"
      @interpolationChange="onInterpolationChange">
    </image-settings>
    <v-form v-if="type === 'local'">
      <v-file-input label="Выберите файл" @change="uploadLocal"></v-file-input>
    </v-form>
    <v-form v-if="type === 'url'">
      <v-text-field label="Введите URL изображения" v-model="imageUrl"></v-text-field>
      <v-btn @click="uploadFromUrl">Загрузить</v-btn>
    </v-form>
    <div v-if="image">
      <p>Исходное количество пикселей: {{ originalPixels }} Мп</p>
      <p>Количество пикселей после масштабирования: {{ scaledPixels }} Мп</p>
    </div>
    <image-colors :pixelSearchRef="$refs.pixelSearch"></image-colors>
    <image-chart :image="image" @update-image="updateImage" v-if="image"></image-chart>
    <v-btn class="filters-button" @click="showFilters = true" color="blue">Фильтры</v-btn>
    <image-filters 
      v-if="showFilters" 
      @close="showFilters = false"
      :ctxRef="$refs.pixelSearch.getContext('2d')"
      :dx="0"
      :dy="0"
      :nowW="imageWidth"
      :nowH="imageHeight"
      :startImage="image"
    ></image-filters>
  </div>
  </template>
  
  <script>
  import ImageSettings from './ImageSettings.vue';
  import ImageColors from './ImageColors.vue';
  import ImageChart from './ImageChart.vue';
  import ImageFilters from './ImageFilters.vue';
  
  export default {
    components: {
      ImageSettings,
      ImageColors,
      ImageChart,
      ImageFilters
    },
    props: {
      type: String
    },
    data() {
      return {
        imageUrl: "",
        image: null,
        originalPixels: 0,
        scaledPixels: 0,
        imageWidth: 0,
        imageHeight: 0,
        resizeDialog: false,
        interpolation: 'nearest',
        showFilters: false
      };
    },
    methods: {
      uploadLocal(file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          const img = new Image();
          img.onload = () => {
            this.image = img;
            this.imageWidth = Math.round(img.width);
            this.imageHeight = Math.round(img.height);
            this.calculatePixels();
            this.drawImageToCanvas();
          };
          img.src = e.target.result;
        };
        reader.readAsDataURL(file);
      },
      uploadFromUrl() {
        const img = new Image();
        img.crossOrigin = 'Anonymous';
        img.onload = () => {
          this.image = img;
          this.imageWidth = Math.round(img.width);
          this.imageHeight = Math.round(img.height);
          this.calculatePixels();
          this.drawImageToCanvas();
        };
        img.src = this.imageUrl;
      },
      calculatePixels() {
        this.originalPixels = (this.image.width * this.image.height / 1000000).toFixed(2);
        this.scaledPixels = (this.imageWidth * this.imageHeight / 1000000).toFixed(2);
      },
      drawImageToCanvas() {
        if (!this.image) return;
  
        const canvas = this.$refs.pixelSearch;
        const ctx = canvas.getContext('2d');
  
        canvas.width = this.imageWidth;
        canvas.height = this.imageHeight;
  
        ctx.clearRect(0, 0, canvas.width, canvas.height);
  
        if (this.interpolation === 'nearest') {
          this.drawNearestNeighbor(ctx, this.image, this.imageWidth, this.imageHeight);
        } else if (this.interpolation === 'none') {
          ctx.drawImage(this.image, 0, 0, this.imageWidth, this.imageHeight);
        }
      },
      drawNearestNeighbor(ctx, img, width, height) {
        const imgCanvas = document.createElement('canvas');
        const imgCtx = imgCanvas.getContext('2d');
        imgCanvas.width = img.width;
        imgCanvas.height = img.height;
        imgCtx.drawImage(img, 0, 0);
  
        const imgData = imgCtx.getImageData(0, 0, img.width, img.height).data;
        const scaledImgData = ctx.createImageData(width, height);
  
        for (let y = 0; y < height; y++) {
          for (let x = 0; x < width; x++) {
            const srcX = Math.floor(x * img.width / width);
            const srcY = Math.floor(y * img.height / height);
            const srcIndex = (srcY * img.width + srcX) * 4;
            const destIndex = (y * width + x) * 4;
            scaledImgData.data[destIndex] = imgData[srcIndex];
            scaledImgData.data[destIndex + 1] = imgData[srcIndex + 1];
            scaledImgData.data[destIndex + 2] = imgData[srcIndex + 2];
            scaledImgData.data[destIndex + 3] = imgData[srcIndex + 3];
          }
        }
  
        ctx.putImageData(scaledImgData, 0, 0);
      },
      onResize({ width, height }) {
        this.imageWidth = Math.round(width);
        this.imageHeight = Math.round(height);
        this.calculatePixels();
        this.drawImageToCanvas();
      },
      onInterpolationChange(method) {
        this.interpolation = method;
        this.drawImageToCanvas();
      },
      updateImage(imageDataUrl) {
        const img = new Image();
        img.onload = () => {
          this.image = img;
          this.drawImageToCanvas();
        };
        img.src = imageDataUrl;
      }
    },
    mounted() {
      this.$refs.imageSettings.setCanvasRef(this.$refs.pixelSearch);
    }
  }
  </script>
  
  <style>
  canvas {
    width: 1500px;
    height: auto;
    padding: 50px;
  }
  .row{
    margin: 0px;
  }
  .padding{
    padding-left: 50px;
  }
  </style>