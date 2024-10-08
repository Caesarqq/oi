<template>
  <div>
    <v-btn color="primary" @click="toggleCurveCorrection">Коррекция кривыми</v-btn>
    <div v-if="showCurveCorrection" class="modal-overlay">
      <div class="modal">
        <div class="modal__container header">
          <p class="header__text">Градационное преобразование</p>
        </div>
        <div class="modal__content">
          <div class="modal__line">
            <div class="modal__element">
              <label class="modal__label">
                X1:
                <input
                  type="number"
                  v-model.number="point1.x"
                  class="modal__input"
                  min="0"
                  max="254"
                  @change="changeX1"
                />
              </label>
            </div>
            <div class="modal__element">
              <label class="modal__label">
                Y1:
                <input
                  type="number"
                  v-model.number="point1.y"
                  class="modal__input"
                  min="1"
                  max="255"
                  @change="changeY1"
                />
              </label>
            </div>
          </div>
          <div class="modal__line">
            <div class="modal__element">
              <label class="modal__label">
                X2:
                <input
                  type="number"
                  v-model.number="point2.x"
                  class="modal__input"
                  :min="point1.x + 1"
                  max="255"
                  @change="changeX2"
                />
              </label>
            </div>
            <div class="modal__element">
              <label class="modal__label">
                Y2:
                <input
                  type="number"
                  v-model.number="point2.y"
                  class="modal__input"
                  min="1"
                  max="255"
                  @change="changeY2"
                />
              </label>
            </div>
          </div>
          <div class="modal__line">
            <v-checkbox v-model="preview" label="Предпросмотр" @change="applyPreview"></v-checkbox>
          </div>
          <div class="modal__line">
            <div class="modal__element">
              <v-btn color="primary" @click="applyCurves">Применить</v-btn>
            </div>
            <div class="modal__element">
              <v-btn color="secondary" @click="resetPoints">Сброс</v-btn>
            </div>
            <div class="modal__element">
              <v-btn color="secondary" @click="toggleCurveCorrection">Закрыть</v-btn>
            </div>
          </div>
          <div class="modal__line chart">
            <canvas id="rgbHistogram" width="250" height="150" ref="chart"></canvas>
          </div>
        </div>
      </div>
    </div>
    <v-dialog v-model="showResultDialog" persistent max-width="600">
      <v-card>
        <v-card-title>Результат</v-card-title>
        <v-card-text>
          <canvas id="resultChart" width="500" height="300"></canvas>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="primary" @click="closeResultDialog">Закрыть</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script>
import { Chart, registerables } from 'chart.js';
import annotationPlugin from 'chartjs-plugin-annotation';

Chart.register(...registerables, annotationPlugin);

export default {
  name: "ImageChart",
  props: {
    image: Object,
  },
  data() {
    return {
      point1: { x: 0, y: 0 },
      point2: { x: 255, y: 255 },
      histograms: {
        red: [],
        green: [],
        blue: []
      },
      originalImageData: null,
      chart: null,
      showCurveCorrection: false,
      showResultDialog: false,
      preview: false,
    };
  },
  watch: {
    image: {
      handler(newImage) {
        if (newImage) {
          this.calculateHistograms();
          this.renderHistogram();
        }
      },
      immediate: true
    }
  },
  methods: {
    toggleCurveCorrection() {
      this.showCurveCorrection = !this.showCurveCorrection;
      if (this.showCurveCorrection) {
        this.calculateHistograms();
        this.$nextTick(() => {
          this.renderHistogram(true);
        });
      }
    },
    applyCurves() {
      this.correctImage(); 
      this.preview = false; 
      this.calculateHistograms(); 
      this.renderHistogram(true); 
    },
    closeResultDialog() {
      this.showResultDialog = false;
    },
    calculateHistograms() {
      const red = new Array(256).fill(0);
      const green = new Array(256).fill(0);
      const blue = new Array(256).fill(0);

      const canvas = document.createElement('canvas');
      const ctx = canvas.getContext('2d');
      canvas.width = this.image.width;
      canvas.height = this.image.height;
      ctx.drawImage(this.image, 0, 0, this.image.width, this.image.height);

      const imageData = ctx.getImageData(0, 0, this.image.width, this.image.height).data;
      for (let i = 0; i < imageData.length; i += 4) {
        red[imageData[i]]++;
        green[imageData[i + 1]]++;
        blue[imageData[i + 2]]++;
      }

      this.histograms.red = this.normalizeHistogram(red);
      this.histograms.green = this.normalizeHistogram(green);
      this.histograms.blue = this.normalizeHistogram(blue);

      this.originalImageData = ctx.getImageData(0, 0, this.image.width, this.image.height);
    },
    normalizeHistogram(histogram) {
      const max = Math.max(...histogram);
      return histogram.map(value => (value / max) * 255);
    },
    renderHistogram(forceRender = false) {
      if (this.chart && !forceRender) return;

      const ctx = this.$refs.chart.getContext('2d');
      if (this.chart) {
        this.chart.destroy();
      }
      this.chart = new Chart(ctx, {
        type: 'scatter',
        data: {
          labels: Array.from({ length: 256 }, (_, i) => i),
          datasets: [
            {
              label: 'Red',
              data: this.histograms.red.map((value, index) => ({ x: index, y: value })),
              backgroundColor: "rgba(255, 0, 0, 0.8)",
            },
            {
              label: 'Green',
              data: this.histograms.green.map((value, index) => ({ x: index, y: value })),
              backgroundColor: "rgba(0, 255, 0, 0.8)",
            },
            {
              label: 'Blue',
              data: this.histograms.blue.map((value, index) => ({ x: index, y: value })),
              backgroundColor: "rgba(0, 0, 255, 0.8)",
            },
            {
              label: 'line',
              data: [
                { x: 0, y: this.point1.y },
                { x: this.point1.x, y: this.point1.y },
                { x: this.point2.x, y: this.point2.y },
                { x: 255, y: this.point2.y },
              ],
              borderColor: "rgba(0,0,0,1)",
              borderWidth: 1,
              fill: false,
              showLine: true,
            },
          ]
        },
        options: {
          animation: false,
          aspectRatio: 1,
          scales: {
            x: {
              type: "linear",
              position: "bottom",
              min: 0,
              max: 255,
              ticks: {
                stepSize: 10,
                color: 'black',
              },
            },
            y: {
              beginAtZero: true,
              min: 0,
              max: 255,
              ticks: {
                stepSize: 10,
                color: 'black',
              },
            }
          },
        },
      });
    },
    changeX1() {
      if (this.point1.x < 0) this.point1.x = 0;
      if (this.point1.x > 254) this.point1.x = 254;
      if (this.point2.x <= this.point1.x) {
        this.point2.x = this.point1.x + 1;
      }
      this.updateCurve();
      this.applyPreview(); 
    },
    changeY1() {
      if (this.point1.y < 1) this.point1.y = 1;
      if (this.point1.y > 255) this.point1.y = 255;
      this.updateCurve();
      this.applyPreview(); 
    },
    changeX2() {
      if (this.point2.x <= this.point1.x) this.point2.x = this.point1.x + 1;
      if (this.point2.x > 255) this.point2.x = 255;
      this.updateCurve();
      this.applyPreview();
    },
    changeY2() {
      if (this.point2.y < 1) this.point2.y = 1;
      if (this.point2.y > 255) this.point2.y = 255;
      this.updateCurve();
      this.applyPreview(); 
    },
    applyPreview() {
      if (this.preview) {
        this.correctImage();
      } else {
        this.restoreOriginalImage();
      }
      this.calculateHistograms();
      this.renderHistogram();
    },
    correctImage() {
      const canvas = document.createElement('canvas');
      const ctx = canvas.getContext('2d');
      canvas.width = this.image.width;
      canvas.height = this.image.height;

      ctx.drawImage(this.image, 0, 0, this.image.width, this.image.height);

      const imageData = ctx.getImageData(0, 0, this.image.width, this.image.height);
      const data = imageData.data;
      const lut = this.createLookupTable();

      for (let i = 0; i < data.length; i += 4) {
        data[i] = lut[data[i]];
        data[i + 1] = lut[data[i + 1]];
        data[i + 2] = lut[data[i + 2]];
      }

      ctx.putImageData(imageData, 0, 0);
      this.$emit('update-image', ctx.canvas.toDataURL());
    },
    createLookupTable() {
      const lut = new Uint8Array(256);
      for (let i = 0; i < 256; i++) {
        if (i <= this.point1.x) {
          lut[i] = Math.round((i / this.point1.x) * this.point1.y);
        } else if (i >= this.point2.x) {
          lut[i] = Math.round(((i - this.point2.x) / (255 - this.point2.x)) * (255 - this.point2.y) + this.point2.y);
        } else {
          lut[i] = Math.round(((i - this.point1.x) / (this.point2.x - this.point1.x)) * (this.point2.y - this.point1.y) + this.point1.y);
        }
      }
      return lut;
    },
    resetPoints() {
      this.point1 = { x: 0, y: 0 };
      this.point2 = { x: 255, y: 255 };
      this.updateCurve();
      this.applyPreview();
    },
    restoreOriginalImage() {
      const canvas = document.createElement('canvas');
      const ctx = canvas.getContext('2d');
      canvas.width = this.image.width;
      canvas.height = this.image.height;
      ctx.putImageData(this.originalImageData, 0, 0);
      this.$emit('update-image', ctx.canvas.toDataURL());
    },
    updateCurve() {
      if (this.chart) {
        this.chart.data.datasets[3].data = [
          { x: 0, y: this.point1.y },
          { x: this.point1.x, y: this.point1.y },
          { x: this.point2.x, y: this.point2.y },
          { x: 255, y: this.point2.y },
        ];
        this.chart.update();
      }
    },
  },
};
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal {
  width: 400px;
  max-height: 80vh;
  padding: 15px;
  background-color: #fff;
  border-radius: 20px;
  display: flex;
  flex-direction: column;
  row-gap: 7px;
  overflow-y: auto;
}

.modal__line {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: center;
}

.chart {
  padding: 5px;
  background-color: white;
  border-radius: 10px;
  border: #32373d solid 1px;
}
</style>
