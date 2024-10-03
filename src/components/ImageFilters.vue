<template>
<div class="modal">
<div class="modal__container header">
<p class="header__text">Фильтрация изображения</p>
<button class="header__button" @click="$emit('close')">X</button>
</div>
<div class="modal__line">
<div class="modal__element matrix">
    <div class="matrix__line">
    <input class="matrix__input" type="number" v-model="matrix[0][0]" />
    <input class="matrix__input" type="number" v-model="matrix[0][1]" />
    <input class="matrix__input" type="number" v-model="matrix[0][2]" />
    </div>
    <div class="matrix__line">
    <input class="matrix__input" type="number" v-model="matrix[1][0]" />
    <input class="matrix__input" type="number" v-model="matrix[1][1]" />
    <input class="matrix__input" type="number" v-model="matrix[1][2]" />
    </div>
    <div class="matrix__line">
    <input class="matrix__input" type="number" v-model="matrix[2][0]" />
    <input class="matrix__input" type="number" v-model="matrix[2][1]" />
    <input class="matrix__input" type="number" v-model="matrix[2][2]" />
    </div>
</div>
<div class="modal__element type">
    <label class="modal__label">
    <input type="radio" value="same" v-model="selectedFilter" />
    Тождественное
    </label>
    <label class="modal__label">
    <input type="radio" value="sharp" v-model="selectedFilter" />
    Резкость
    </label>
    <label class="modal__label">
    <input type="radio" value="gaus" v-model="selectedFilter" />
    Гаусс
    </label>
    <label class="modal__label">
    <input type="radio" value="rect" v-model="selectedFilter" />
    Прямоугольное
    </label>
    <label class="modal__label">
    <input type="radio" value="sobel" v-model="selectedFilter" />
    Собель
    </label>
</div>
</div>
<div class="modal__line">
<label>
    <input type="checkbox" v-model="preview" @change="applyPreview" />
    Предпросмотр
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
    [0, 0, 0]
],
sobelX: [
    [-1, 0, 1],
    [-2, 0, 2],
    [-1, 0, 1]
],
sobelY: [
    [-1, -2, -1],
    [0, 0, 0],
    [1, 2, 1]
]
};
},
watch: {
selectedFilter(value) {
switch (value) {
    case 'same':
    this.matrix = [
        [0, 0, 0],
        [0, 1, 0],
        [0, 0, 0]
    ];
    break;
    case 'sharp':
    this.matrix = [
        [-1, -1, -1],
        [-1, 9, -1],
        [-1, -1, -1]
    ];
    break;
    case 'gaus':
    this.matrix = [
        [1, 2, 1],
        [2, 4, 2],
        [1, 2, 1]
    ];
    break;
    case 'rect':
    this.matrix = [
        [1, 1, 1],
        [1, 1, 1],
        [1, 1, 1]
    ];
    break;
    case 'sobel':
    this.matrix = [
        [1, 0, -1],
        [2, 0, -2],
        [1, 0, -1]
    ];
    break;
}
if (this.preview) {
    this.applyFilter();
}
}
},
methods: {
changeX1() {
this.matrix[0][0] = Number(this.x1);
if (this.preview) {
    this.applyFilter();
}
},
changeX2() {
this.matrix[0][1] = Number(this.x2);
if (this.preview) {
    this.applyFilter();
}
},
changeX3() {
this.matrix[0][2] = Number(this.x3);
if (this.preview) {
    this.applyFilter();
}
},
changeX4() {
this.matrix[1][0] = Number(this.x4);
if (this.preview) {
    this.applyFilter();
}
},
changeX5() {
this.matrix[1][1] = Number(this.x5);
if (this.preview) {
    this.applyFilter();
}
},
changeX6() {
this.matrix[1][2] = Number(this.x6);
if (this.preview) {
    this.applyFilter();
}
},
changeX7() {
this.matrix[2][0] = Number(this.x7);
if (this.preview) {
    this.applyFilter();
}
},
changeX8() {
this.matrix[2][1] = Number(this.x8);
if (this.preview) {
    this.applyFilter();
}
},
changeX9() {
this.matrix[2][2] = Number(this.x9);
if (this.preview) {
    this.applyFilter();
}
},
resetFilter() {
this.selectedFilter = "same";
if (this.preview) {
    this.ctxRef.drawImage(
    this.startImage,
    this.dx,
    this.dy,
    this.nowW,
    this.nowH
    );
}
},
applyPreview() {
if (this.preview) {
    this.applyFilter();
} else {
    this.ctxRef.drawImage(
    this.startImage,
    this.dx,
    this.dy,
    this.nowW,
    this.nowH
    );
}
},
applyFilter() {
if (!this.preview && this.selectedFilter !== "sobel") {
    this.ctxRef.drawImage(
    this.startImage,
    this.dx,
    this.dy,
    this.nowW,
    this.nowH
    );
}

const imageData = this.ctxRef.getImageData(
    this.dx,
    this.dy,
    this.nowW,
    this.nowH
);
const newData = new Uint8ClampedArray(imageData.data.length);

const paddedData = this.padImageData(
    imageData.data,
    imageData.width,
    imageData.height
);

if (this.selectedFilter === "sobel") {
    const gradientX = this.applyKernel(
    paddedData,
    imageData.width,
    imageData.height,
    this.sobelX
    );
    const gradientY = this.applyKernel(
    paddedData,
    imageData.width,
    imageData.height,
    this.sobelY
    );
    for (let i = 0; i < newData.length; i += 4) {
    const magnitude = Math.sqrt(
        gradientX[i] ** 2 + gradientY[i] ** 2
    );
    newData[i] = magnitude;
    newData[i + 1] = magnitude;
    newData[i + 2] = magnitude;
    newData[i + 3] = 255;
    }
} else {
    for (let y = 0; y < imageData.height; y++) {
    for (let x = 0; x < imageData.width; x++) {
        for (let c = 0; c < 4; c++) {
        const outputIndex = (y * imageData.width + x) * 4 + c;
        let sum = 0;
        let kernelSum = 0;
        for (let ky = 0; ky < 3; ky++) {
            for (let kx = 0; kx < 3; kx++) {
            const inputIndex =
                ((y + ky) * (imageData.width + 2) + (x + kx)) * 4 + c;
            sum += paddedData[inputIndex] * this.matrix[ky][kx];
            kernelSum += this.matrix[ky][kx];
            }
        }
        newData[outputIndex] = sum / kernelSum;
        }
    }
    }
}

imageData.data.set(newData);
this.ctxRef.putImageData(imageData, this.dx, this.dy);
},
applyKernel(paddedData, width, height, kernel) {
const result = new Uint8ClampedArray(paddedData.length);
for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
    for (let c = 0; c < 4; c++) {
        let sum = 0;
        for (let ky = 0; ky < 3; ky++) {
        for (let kx = 0; kx < 3; kx++) {
            const inputIndex =
            ((y + ky) * (width + 2) + (x + kx)) * 4 + c;
            sum += paddedData[inputIndex] * kernel[ky][kx];
        }
        }
        result[(y * width + x) * 4 + c] = sum;
    }
    }
}
return result;
},
padImageData(data, width, height) {
const paddedWidth = width + 2;
const paddedHeight = height + 2;
const paddedData = new Uint8ClampedArray(paddedWidth * paddedHeight * 4);

for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
    const inputIndex = (y * width + x) * 4;
    const outputIndex = ((y + 1) * paddedWidth + x + 1) * 4;
    paddedData.set(
        data.subarray(inputIndex, inputIndex + 4),
        outputIndex
    );
    }
}

for (let y = 0; y < paddedHeight; y++) {
    for (let x = 0; x < paddedWidth; x++) {
    const outputIndex = (y * paddedWidth + x) * 4;
    if (
        x === 0 ||
        x === paddedWidth - 1 ||
        y === 0 ||
        y === paddedHeight - 1
    ) {
        const nearestX = Math.max(1, Math.min(x, paddedWidth - 2));
        const nearestY = Math.max(1, Math.min(y, paddedHeight - 2));
        const nearestIndex = (nearestY * paddedWidth + nearestX) * 4;
        paddedData.set(
        paddedData.subarray(nearestIndex, nearestIndex + 4),
        outputIndex
        );
    }
    }
}
return paddedData;
}
}
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

.modal__input {
width: 30%;
border-radius: 6px;
padding: 3px;
}

.type {
display: flex;
flex-direction: column;
margin-left: 20px;
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

/* .header__text {
font-size: 20px;
color: #007bff; 
} */

/* .header__button {
width: 30px;
font-size: 20px;
color: #007bff; 
background-color: transparent;
border: none;
cursor: pointer;
} */
</style>
