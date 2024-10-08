<template>
  <div>
    <v-tooltip bottom>
      <template v-slot:activator="{ on, attrs }">
        <v-btn
          v-bind="attrs"
          v-on="on"
          :color="currentTool === 'hand' ? 'red' : 'blue'"
          @click="selectTool('hand')"
        >
          Рука
        </v-btn>
      </template>
      <span>Перемещение изображения</span>
    </v-tooltip>
    <div class="pipette-panel" :class="{ collapsed: !isPanelOpen }">
      <v-btn icon @click="togglePanel" class="toggle-btn">
        <v-icon>{{ isPanelOpen ? 'mdi-chevron-right' : 'mdi-chevron-left' }}</v-icon>
      </v-btn>
      <div v-show="isPanelOpen" class="panel-content">
        <v-btn
          :color="currentTool === 'pipette' ? 'blue' : 'blue'"
          @click="selectTool('pipette')"
          class="pipette-btn"
        >
          Пипетка
        </v-btn>
        <div class="color-info">
          <div class="color-box" :style="{ backgroundColor: color1.hex }"></div>
          <div>Цвет: {{ color1.hex }}</div>
          <div>Координаты: x: {{ color1.x }}, y: {{ color1.y }}</div>
          <div>RGB: {{ color1.rgb }}</div>
          <div>XYZ: {{ color1.xyz }}</div>
          <div>Lab: {{ color1.lab }}</div>
        </div>
        <div class="color-info">
          <div class="color-box" :style="{ backgroundColor: color2.hex }"></div>
          <div>Цвет: {{ color2.hex }}</div>
          <div>Координаты: x: {{ color2.x }}, y: {{ color2.y }}</div>
          <div>RGB: {{ color2.rgb }}</div>
          <div>XYZ: {{ color2.xyz }}</div>
          <div>Lab: {{ color2.lab }}</div>
        </div>
        <div :class="{ insufficient: contrastRatio < 4.5 }">
          <div>Контрастное соотношение: {{ contrastRatio.toFixed(2) }}:1</div>
          <div v-if="contrastRatio < 4.5">Контраст недостаточный</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import colorConvert from 'color-convert';

export default {
  props: {
    pixelSearchRef: Object
  },
  data() {
    return {
      currentTool: null,
      color1: this.createEmptyColor(),
      color2: this.createEmptyColor(),
      contrastRatio: 1,
      isPanelOpen: true,
      isDragging: false,
      startX: 0,
      startY: 0,
      offsetX: 0,
      offsetY: 0
    };
  },
  methods: {
    createEmptyColor() {
      return {
        hex: '#000000',
        rgb: 'rgb(0, 0, 0)',
        xyz: 'XYZ(0, 0, 0)',
        lab: 'Lab(0, 0, 0)',
        x: 0,
        y: 0
      };
    },
    selectTool(tool) {
      this.currentTool = tool;
      if (tool === 'pipette') {
        this.$emit('pipette-active', true);
        this.pixelSearchRef.addEventListener('click', this.updateColor);
      } else {
        this.$emit('pipette-active', false);
        this.pixelSearchRef.removeEventListener('click', this.updateColor);
      }

      if (tool === 'hand') {
        this.enableHandTool();
      } else {
        this.disableHandTool();
      }
    },
    togglePanel() {
      this.isPanelOpen = !this.isPanelOpen;
    },
    updateColor(event) {
      if (this.currentTool !== 'pipette') return;

      const rect = this.pixelSearchRef.getBoundingClientRect();
      const x = Math.floor((event.clientX - rect.left) * (this.pixelSearchRef.width / rect.width));
      const y = Math.floor((event.clientY - rect.top) * (this.pixelSearchRef.height / rect.height));

      const ctx = this.pixelSearchRef.getContext('2d');
      const imageData = ctx.getImageData(x, y, 1, 1);
      const colorData = imageData.data;

      const hex = `#${colorConvert.rgb.hex(colorData[0], colorData[1], colorData[2])}`;
      const rgb = `rgb(${colorData[0]}, ${colorData[1]}, ${colorData[2]})`;
      const xyz = colorConvert.rgb.xyz(colorData[0], colorData[1], colorData[2]);
      const lab = colorConvert.rgb.lab(colorData[0], colorData[1], colorData[2]);

      const color = {
        hex,
        rgb,
        xyz: `XYZ(${xyz[0].toFixed(2)}, ${xyz[1].toFixed(2)}, ${xyz[2].toFixed(2)})`,
        lab: `Lab(${lab[0].toFixed(2)}, ${lab[1].toFixed(2)}, ${lab[2].toFixed(2)})`,
        x,
        y
      };

      if (event.altKey || event.ctrlKey || event.shiftKey) {
        this.color2 = color;
      } else {
        this.color1 = color;
      }
      this.calculateContrast();
    },
    enableHandTool() {
      window.addEventListener('mousedown', this.startDrag);
      window.addEventListener('mousemove', this.onDrag);
      window.addEventListener('mouseup', this.endDrag);
    },
    disableHandTool() {
      window.removeEventListener('mousedown', this.startDrag);
      window.removeEventListener('mousemove', this.onDrag);
      window.removeEventListener('mouseup', this.endDrag);
    },
    startDrag(event) {
      if (this.currentTool !== 'hand') return;
      this.isDragging = true;
      this.startX = event.clientX - this.offsetX;
      this.startY = event.clientY - this.offsetY;
    },
    onDrag(event) {
      if (!this.isDragging || this.currentTool !== 'hand') return;
      this.offsetX = event.clientX - this.startX;
      this.offsetY = event.clientY - this.startY;
      this.updateCanvasPosition();
    },
    endDrag() {
      if (this.currentTool !== 'hand') return;
      this.isDragging = false;
    },
    updateCanvasPosition() {
      const canvas = this.pixelSearchRef;
      canvas.style.transform = `translate(${this.offsetX}px, ${this.offsetY}px)`;
    },
    calculateLuminance(color) {
      const rgb = colorConvert.hex.rgb(color.hex);
      const [r, g, b] = rgb.map(c => {
        c /= 255;
        return c <= 0.03928 ? c / 12.92 : Math.pow((c + 0.055) / 1.055, 2.4);
      });
      return 0.2126 * r + 0.7152 * g + 0.0722 * b;
    },
    calculateContrast() {
      const lum1 = this.calculateLuminance(this.color1);
      const lum2 = this.calculateLuminance(this.color2);
      this.contrastRatio = (Math.max(lum1, lum2) + 0.05) / (Math.min(lum1, lum2) + 0.05);
    }
  },
  mounted() {
    this.selectTool('pipette');
  }
};
</script>

<style>
.pipette-panel {
  position: absolute;
  right: 0;
  top: 30px;
  width: 250px;
  height: 100%;
  background-color: #f9f9f9;
  border-left: 1px solid #ddd;
  box-shadow: -2px 0 5px rgba(0, 0, 0, 0.1);
  transition: width 0.3s ease;
}

.pipette-panel.collapsed {
  width: 40px;
}

.panel-content {
  padding: 15px;
}

.toggle-btn {
  position: absolute;
  left: -25px;
  top: 10px;
  background-color: #ddd;
  border-radius: 50%;
}

.color-info {
  display: flex;
  align-items: center;
  flex-direction: column;
  margin-bottom: 15px;
}

.color-box {
  width: 30px;
  height: 30px;
  margin-bottom: 10px;
  border: 1px solid #000;
}

.insufficient {
  color: red;
}

.pipette-btn {
  width: 100%;
  margin-bottom: 20px;
  font-size: 14px;
}
</style>