<template>
<div class="canvas">
    <div class="header">
    <v-btn @click="toggleUploadModal">Загрузить изображение</v-btn>
    <v-btn @click="clearCanvas">Очистить</v-btn>
    <v-btn @click="toggleScaleModal">Масштаб</v-btn>
    <v-btn @click="togglePippetModal">Пипетка</v-btn>
    <v-btn @click="toggleGrab">Перемещение</v-btn>
    <v-btn @click="toggleCurvesModal">Кривые</v-btn>
    <v-btn @click="toggleFilterModal">Фильтрация</v-btn>
    <v-btn @click="saveImage">Сохранить</v-btn>
    </div>
        <div class="wrapper">
        <aside-panel
        @scale="scaleImage"
        :x="x"
        :y="y"
        :height="height"
        :width="width"
        :resl="resl"
        :showData="showData"
        />
        <div class="drawing">
        <canvas
        ref="canvas"
        width="1250"
        height="640"
        @mousedown="handleMouseDown"
        @mouseup="handleMouseUp"
        @mousemove="handleMouseMove"
        @mousewheel="handleMouseWheel"
        />
        </div>
    <pippet-modal
        v-if="showPippetModal"
        @toggle="togglePippetModal"
        :resl="resl"
        :startX="startX"
        :startY="startY"
        :nowW="nowW"
        :nowH="nowH"
    />
    <curves-modal
        v-if="showCurvesModal"
        @toggle="toggleCurvesModal"
        :ctxRef="ctx"
        :dx="dx"
        :dy="dy"
        :nowW="nowW"
        :nowH="nowH"
        :startImage="startImage"
    />
    <image-filters
        v-if="showFilterModal"
        @toggle="toggleFilterModal"
        :ctxRef="ctx"
        :dx="dx"
        :dy="dy"
        :nowW="nowW"
        :nowH="nowH"
        :startImage="startImage"
    />
    </div>
    <div style="display: none">
    <canvas ref="helperCanvas"></canvas>
    </div>
    <image-uploader v-if="showUploadModal" @toggle="toggleUploadModal" @upload="loadImage" />
    <scale-modal v-if="showScaleModal" @toggle="toggleScaleModal" @resize="resizeImage" />
</div>
</template>

<script>
import AsidePanel from './AsidePanel.vue';
import PippetModal from './PippetModal.vue';
import CurvesModal from './CurvesModal.vue';
import ImageFilters from './ImageFilters.vue';
import ImageUploader from './ImageUploader.vue';
import ScaleModal from './ScaleModal.vue';

export default {
components: {
    AsidePanel,
    PippetModal,
    CurvesModal,
    ImageFilters,
    ImageUploader,
    ScaleModal
},
data() {
    return {
    startImage: null,
    canvas: null,
    ctx: null,
    helperCanvas: null,
    helperCtx: null,
    x: 0,
    y: 0,
    showUploadModal: false,
    showScaleModal: false,
    showPippetModal: false,
    showCurvesModal: false,
    showFilterModal: false,
    showData: false,
    height: 0,
    width: 0,
    nowH: null,
    nowW: null,
    dx: null,
    dy: null,
    isDragging: false,
    startX: null,
    startY: null,
    state: '',
    dx: 0, // Начальная координата X
    dy: 0, // Начальная координата Y
    resl: null,
    isShift: false
    };
},
mounted() {
    this.canvas = this.$refs.canvas;
    this.ctx = this.canvas.getContext('2d', { willReadFrequently: true });
    this.helperCanvas = this.$refs.helperCanvas;
    this.helperCtx = this.helperCanvas.getContext('2d', { willReadFrequently: true });
    window.addEventListener('keydown', this.handleShiftKeyDown);
    window.addEventListener('keyup', this.handleShiftKeyUp);
},
methods: {
    toggleUploadModal() {
    this.showUploadModal = !this.showUploadModal;
    },
    toggleScaleModal() {
    this.showScaleModal = !this.showScaleModal;
    },
    togglePippetModal() {
    this.showPippetModal = !this.showPippetModal;
    },
    toggleCurvesModal() {
    this.showCurvesModal = !this.showCurvesModal;
    },
    toggleFilterModal() {
    this.showFilterModal = !this.showFilterModal;
    },
    handleShiftKeyDown(event) {
    if (event.key === 'Shift') {
        this.isShift = true;
    }
    },
    handleShiftKeyUp(event) {
    if (event.key === 'Shift') {
        this.isShift = false;
    }
    },
    handleMouseDown(event) {
    this.isDragging = true;
    this.startX = event.offsetX - this.dx;
    this.startY = event.offsetY - this.dy;
    if (this.state === 'pippet' && this.startImage) {
        this.resl = this.ctx.getImageData(this.x, this.y, 1, 1).data;
    }
    },
    handleMouseMove(event) {
    this.x = event.offsetX;
    this.y = event.offsetY;
    if (this.isDragging && this.state === 'grab') {
        this.dx = event.offsetX - this.startX;
        this.dy = event.offsetY - this.startY;
        this.moveImage();
    }
    },
    handleMouseUp() {
    this.isDragging = false;
    },
    handleMouseWheel(event) {
    if (this.state !== 'pippet' && this.startImage) {
        event.preventDefault();
        const delta = Math.sign(event.deltaY);
        if (this.isShift) {
        this.dx -= delta * 6;
        } else {
        this.dy += delta * 6;
        }
        this.moveImage();
    }
    },
    moveImage() {
    this.clearCanvas();
    this.ctx.drawImage(this.startImage, this.dx, this.dy, this.nowW, this.nowH);
    },
    clearCanvas() {
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    this.resl = null;
    },
    loadImage(image) {
        this.startImage = image;
        this.draw();
    },
    draw() {
        if (this.startImage) {
        this.height = this.startImage.height;
        this.width = this.startImage.width;

        // Рассчитываем масштаб и начальные координаты
        const scale = Math.min(this.canvas.width / this.width, this.canvas.height / this.height);
        const newWidth = this.width * scale;
        const newHeight = this.height * scale;

        this.nowW = newWidth;
        this.nowH = newHeight;

        this.dx = (this.canvas.width - newWidth) / 2; // Используем центрирование
        this.dy = (this.canvas.height - newHeight) / 2; // Используем центрирование

        this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
        this.ctx.drawImage(this.startImage, this.dx, this.dy, newWidth, newHeight);
        }
    },
    resizeImage() {
    const newWidth = this.$refs.scaleModal.outputW;
    const newHeight = this.$refs.scaleModal.outputH;
    this.helperCanvas.width = newWidth;
    this.helperCanvas.height = newHeight;
    this.helperCtx.drawImage(this.startImage, 0, 0, newWidth, newHeight);
    const imageData = this.helperCtx.getImageData(0, 0, newWidth, newHeight);
    this.clearCanvas();
    this.ctx.putImageData(imageData, this.dx, this.dy);
    this.nowW = newWidth;
    this.nowH = newHeight;
    this.updateImage();
    },
    updateImage() {
    const newImage = this.ctx.getImageData(this.dx, this.dy, this.nowW, this.nowH);
    this.helperCanvas.width = this.nowW;
    this.helperCanvas.height = this.nowH;
    this.helperCtx.putImageData(newImage, 0, 0);
    const image = new Image();
    image.src = this.helperCanvas.toDataURL();
    this.startImage = image;
    },
    scaleImage(scalePercentage) {
    const scaleFactor = scalePercentage / 100;
    const newWidth = this.width * scaleFactor;
    const newHeight = this.height * scaleFactor;
    this.helperCanvas.width = newWidth;
    this.helperCanvas.height = newHeight;
    this.helperCtx.drawImage(this.startImage, 0, 0, newWidth, newHeight);
    const imageData = this.helperCtx.getImageData(0, 0, newWidth, newHeight);
    this.clearCanvas();
    this.ctx.putImageData(imageData, this.dx, this.dy);
    this.nowW = newWidth;
    this.nowH = newHeight;
    this.updateImage();
    },
    saveImage() {
    const link = document.createElement('a');
    link.href = this.canvas.toDataURL('image/png');
    link.download = 'filtered_image.png';
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    }
}
};
</script>

<style scoped>
.canvas {
height: 100%;
width: 100%;
display: flex;
flex-direction: column;
}
.header {
display: flex;
justify-content: space-between;
padding: 10px;
background-color: #f5f5f5;
}
.wrapper {
display: flex;
height: calc(100% - 50px);
}
.drawing {
flex-grow: 1;
position: relative;
}
</style>