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

    <v-tooltip bottom>
      <template v-slot:activator="{ on, attrs }">
        <v-btn
          v-bind="attrs"
          v-on="on"
          :color="currentTool === 'pipette' ? 'red' : 'blue'"
          @click="selectTool('pipette')"
        >
          Пипетка
        </v-btn>
      </template>
      <span>Выбор цвета</span>
    </v-tooltip>

    <v-menu
      v-if="currentTool === 'pipette'"
      v-model="colorMenu"
      top
      :close-on-content-click="false"
      persistent
      activator="parent"
    >
      <template v-slot:activator="{ on, attrs }">
        <v-btn v-bind="attrs" v-on="on" icon @click.stop="toggleMenu">
          <v-icon>mdi-palette</v-icon>
        </v-btn>
      </template>
      <v-card>
        <v-card-title>Выбранные цвета</v-card-title>
        <v-card-text>
          <div class="color-info">
            <div class="color-box" :style="{ backgroundColor: color1.hex }" @click.stop></div>
            <div>{{ color1.hex }}</div>
            <div>Координаты: x: {{ color1.x }}, y: {{ color1.y }}</div>
          </div>
          <div class="color-info">
            <div class="color-box" :style="{ backgroundColor: color2.hex }" @click.stop></div>
            <div>{{ color2.hex }}</div>
            <div>Координаты: x: {{ color2.x }}, y: {{ color2.y }}</div>
          </div>
        </v-card-text>
      </v-card>
    </v-menu>

    <div v-if="currentTool === 'pipette'">
      <div class="color-info">
        <div class="color-box" :style="{ backgroundColor: color1.hex }"></div>
        <div>{{ color1.hex }}</div>
        <div>Координаты: x: {{ color1.x }}, y: {{ color1.y }}</div>
      </div>
      <div class="color-info">
        <div class="color-box" :style="{ backgroundColor: color2.hex }"></div>
        <div>{{ color2.hex }}</div>
        <div>Координаты: x: {{ color2.x }}, y: {{ color2.y }}</div>
      </div>
      <div class="{ insufficient: contrastRatio < 4.5 }">
        <div>Контрастное соотношение: {{ contrastRatio.toFixed(2) }}:1</div>
        <div v-if="contrastRatio < 4.5">Контраст недостаточный</div>
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
      colorMenu: false
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
        this.openMenu();
        this.pixelSearchRef.addEventListener('click', this.updateColor); // Добавить обработчик кликов только при выборе пипетки
      } else {
        this.$emit('pipette-active', false);
        this.closeMenu();
        this.pixelSearchRef.removeEventListener('click', this.updateColor); // Удалить обработчик при смене инструмента
      }
    },
    openMenu() {
      this.colorMenu = true;
    },
    closeMenu() {
      this.colorMenu = false;
    },
    toggleMenu() {
      this.colorMenu = !this.colorMenu;
    },
    updateColor(event) {
      if (this.currentTool !== 'pipette') return;

      const rect = this.pixelSearchRef.getBoundingClientRect();
      const x = Math.floor(
        (event.clientX - rect.left) * (this.pixelSearchRef.width / rect.width)
      );
      const y = Math.floor(
        (event.clientY - rect.top) * (this.pixelSearchRef.height / rect.height)
      );

      const ctx = this.pixelSearchRef.getContext('2d');
      const imageData = ctx.getImageData(x, y, 1, 1);
      const colorData = imageData.data;

      const hex = `#${colorConvert.rgb.hex(colorData[0], colorData[1], colorData[2])}`;
      const rgb = `rgb(${colorData[0]}, ${colorData[1]}, ${colorData[2]})`;
      const xyz = colorConvert.rgb.xyz(colorData[0], colorData[1], colorData[2]);
      const lab = colorConvert.rgb.lab(colorData[0], colorData[1], colorData[2]);

      const color = {
        hex: hex,
        rgb: rgb,
        xyz: `XYZ(${xyz[0].toFixed(2)}, ${xyz[1].toFixed(2)}, ${xyz[2].toFixed(2)})`,
        lab: `Lab(${lab[0].toFixed(2)}, ${lab[1].toFixed(2)}, ${lab[2].toFixed(2)})`,
        x: x,
        y: y
      };

      if (event.altKey || event.ctrlKey || event.shiftKey) {
        this.color2 = color;
      } else {
        this.color1 = color;
      }
      this.calculateContrast();
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
    // Удаляем глобальный обработчик, добавляем только на канвас
  },
  beforeDestroy() {
    if (this.pixelSearchRef) {
      this.pixelSearchRef.removeEventListener('click', this.updateColor);
    }
  }
};
</script>

<style>
.color-info {
  display: flex;
  align-items: center;
  margin-bottom: 10px;
}

.color-box {
  width: 20px;
  height: 20px;
  margin-right: 10px;
  border: 1px solid #000;
}

.insufficient {
  color: red;
}
</style>
