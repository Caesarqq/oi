<template>
  <div class="modal">
      <div class="modal__container header">
          <p class="header__text">Фильтрация изображения</p>
          <button class="header__button" @click="$emit('close')">X</button>
      </div>
      <div class="modal__line">
          <div class="modal__element matrix">
              <div class="matrix__line">
                  <input class="matrix__input" type="number" v-model.number="matrix[0][0]" />
                  <input class="matrix__input" type="number" v-model.number="matrix[0][1]" />
                  <input class="matrix__input" type="number" v-model.number="matrix[0][2]" />
              </div>
              <div class="matrix__line">
                  <input class="matrix__input" type="number" v-model.number="matrix[1][0]" />
                  <input class="matrix__input" type="number" v-model.number="matrix[1][1]" />
                  <input class="matrix__input" type="number" v-model.number="matrix[1][2]" />
              </div>
              <div class="matrix__line">
                  <input class="matrix__input" type="number" v-model.number="matrix[2][0]" />
                  <input class="matrix__input" type="number" v-model.number="matrix[2][1]" />
                  <input class="matrix__input" type="number" v-model.number="matrix[2][2]" />
              </div>
          </div>
          <div class="modal__element type">
              <label class="modal__label">
                  <input type="radio" value="same" v-model="selectedFilter" /> Тождественное
              </label>
              <label class="modal__label">
                  <input type="radio" value="sharp" v-model="selectedFilter" /> Резкость
              </label>
              <label class="modal__label">
                  <input type="radio" value="gaus" v-model="selectedFilter" /> Гаусс
              </label>
              <label class="modal__label">
                  <input type="radio" value="rect" v-model="selectedFilter" /> Прямоугольное
              </label>
              <label class="modal__label">
                  <input type="radio" value="sobel" v-model="selectedFilter" /> Собель
              </label>
          </div>
      </div>
      <div class="modal__line">
          <label>
              <input type="checkbox" v-model="preview" @change="applyPreview" /> Предпросмотр
          </label>
      </div>
      <div class="modal__line">
          <div class="modal__element">
              <button class="modal__button" @click="applyFilter">Применить</button>
          </div>
          <div class="modal__element">
              <button class="modal__button" @click="resetFilter">Сбросить</button>
          </div>
      </div>
  </div>
  </template>
  
  <script>
  export default {
      name: "FilteringModal",
      props: {
          dx: Number,
          dy: Number,
          nowW: Number,
          nowH: Number,
          ctxRef: CanvasRenderingContext2D,
          startImage: Object,
      },
      data() {
          return {
              preview: false,
              selectedFilter: "same",
              matrix: [
                  [0, 0, 0],
                  [0, 1, 0],
                  [0, 0, 0],
              ],
              isProcessing: false, 
              sobelX: [
                  [1, 0, -1],
                  [2, 0, -2],
                  [1, 0, -1],
              ],
              sobelY: [
                  [-1, -2, -1],
                  [0, 0, 0],
                  [1, 2, 1],
              ],
              sharp: [
                  [-1, -1, -1],
                  [-1, 9, -1],
                  [-1, -1, -1],
              ],
              gaus: [
                  [1, 2, 1],
                  [2, 4, 2],
                  [1, 2, 1],
              ],
              rect: [
                  [1, 1, 1],
                  [1, 1, 1],
                  [1, 1, 1],
              ],
              same: [
                  [0, 0, 0],
                  [0, 1, 0],
                  [0, 0, 0],
              ],
          };
      },
      watch: {
          selectedFilter(value) {
              switch (value) {
                  case 'same':
                      this.matrix = this.same;
                      break;
                  case 'sharp':
                      this.matrix = this.sharp;
                      break;
                  case 'gaus':
                      this.matrix = this.gaus;
                      break;
                  case 'rect':
                      this.matrix = this.rect;
                      break;
                  case 'sobel':
                      this.matrix = this.sobelX; 
                      break;
              }
              if (this.preview) {
                  this.applyFilter();
              }
          },
      },
      methods: {
          applyFilter() {
              if (this.isProcessing) return; 
              this.isProcessing = true;
  
              const imageData = this.ctxRef.getImageData(this.dx, this.dy, this.nowW, this.nowH);
              const newData = new Uint8ClampedArray(imageData.data.length);
              const paddedData = this.padImageData(imageData.data, imageData.width, imageData.height);
  
              this.processFilterAsync(imageData, newData, paddedData, () => {
                  imageData.data.set(newData);
                  this.ctxRef.putImageData(imageData, this.dx, this.dy);
                  this.isProcessing = false; 
              });
          },
  
          processFilterAsync(imageData, newData, paddedData, callback) {
              let y = 0;
              const step = () => {
                  if (y >= imageData.height) {
                      callback(); 
                      return;
                  }
                  for (let x = 0; x < imageData.width; x++) {
                      for (let c = 0; c < 3; c++) { 
                          const outputIndex = (y * imageData.width + x) * 4 + c;
                          let sum = 0;
                          for (let ky = 0; ky < 3; ky++) {
                              for (let kx = 0; kx < 3; kx++) {
                                  const inputIndex = ((y + ky) * (imageData.width + 2) + (x + kx)) * 4 + c;
                                  sum += paddedData[inputIndex] * this.matrix[ky][kx];
                              }
                          }
                          newData[outputIndex] = Math.min(255, Math.max(0, sum));
                      }
                      newData[(y * imageData.width + x) * 4 + 3] = imageData.data[(y * imageData.width + x) * 4 + 3];
                  }
                  y++;
                  requestAnimationFrame(step); 
              };
              step();
          },
  
          padImageData(data, width, height) {
              const paddedWidth = width + 2;
              const paddedHeight = height + 2;
              const paddedData = new Uint8ClampedArray(paddedWidth * paddedHeight * 4);
  
              for (let y = 0; y < height; y++) {
                  for (let x = 0; x < width; x++) {
                      const inputIndex = (y * width + x) * 4;
                      const outputIndex = ((y + 1) * paddedWidth + x + 1) * 4;
                      paddedData.set(data.subarray(inputIndex, inputIndex + 4), outputIndex);
                  }
              }
              return paddedData;
          },
  
          resetFilter() {
              this.selectedFilter = "same";
              if (this.preview) {
                  this.ctxRef.drawImage(this.startImage, this.dx, this.dy, this.nowW, this.nowH);
              }
          },
      },
  };
  </script>
  
  
  
  <style scoped>
  .modal {
      position: absolute;
      bottom: -20px;
      left: 12px;
      background-color: #fff;
      width: 25vw;
      padding: 15px;
      border-radius: 20px;
      display: flex;
      flex-direction: column;
      row-gap: 20px;
  }
  .modal__element {
      display: flex;
      text-align: left;
      align-items: flex-start;
      justify-content: flex-start;
      flex-direction: column;
  }
  .modal__container {
      display: flex;
      flex-direction: row;
      justify-content: space-between;
  }
  .modal__line {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
  }
  .modal__button {
      background-color: #007bff;
      border: none;
      border-radius: 5px;
      width: 110px;
      padding: 14px;
      font-size: 16px;
      color: white;
  }
  .modal__label {
      display: flex;
      flex-direction: row;
      column-gap: 20px;
      align-items: center;
  }
  .matrix {
      display: flex;
      flex-direction: column;
      row-gap: 5px;
  }
  .matrix__line {
      display: flex;
      flex-direction: row;
      column-gap: 5px;
  }
  .matrix__input {
      width: 45px;
  }
  </style>
  