<template>
<v-dialog :value="resizeDialog" persistent max-width="600px" @input="$emit('update:resizeDialog', $event)">
    <template v-slot:activator="{ on }">
    <v-btn color="primary" dark v-on="on" class="mt-3 padding">Настроить размер</v-btn>
    </template>
    <v-card>
    <v-card-title>Настройки изменения размера</v-card-title>
    <v-card-text>
        <v-select
        v-model="resizeMode"
        :items="resizeModes"
        label="Изменить размер в"
        class="mt-3"
        ></v-select>
        <v-checkbox
        v-model="maintainAspectRatio"
        label="Сохранять пропорции"
        class="mt-3"
        ></v-checkbox>
        <div v-if="resizeMode === 'percent'">
        <v-text-field
            v-model="percentWidth"
            label="Ширина (%)"
            type="number"
            class="mt-3"
            @input="onWidthChange('percent')"
        ></v-text-field>
        <v-text-field
            v-model="percentHeight"
            label="Высота (%)"
            type="number"
            class="mt-3"
            @input="onHeightChange('percent')"
        ></v-text-field>
        </div>
        <div v-else>
        <v-text-field
            v-model="scaledWidth"
            label="Ширина (px)"
            type="number"
            class="mt-3"
            @input="onWidthChange('pixels')"
        ></v-text-field>
        <v-text-field
            v-model="scaledHeight"
            label="Высота (px)"
            type="number"
            class="mt-3"
            @input="onHeightChange('pixels')"
        ></v-text-field>
        </div>
        <v-slider
        v-model="slider"
        :max="max"
        :min="min"
        class="align-center"
        hide-details
        @input="onSliderChange"
        >
        <template v-slot:append>
            <v-text-field
            v-model="sliderPercent"
            density="compact"
            style="width: 40px"
            type="text"
            hide-details
            single-line
            ></v-text-field>
        </template>
        </v-slider>
        <v-tooltip bottom>
        <template v-slot:activator="{ on, attrs }">
            <v-select
            v-model="interpolation"
            :items="interpolationMethods"
            label="Алгоритм интерполяции"
            class="mt-3"
            v-bind="attrs"
            v-on="on"
            ></v-select>
        </template>
        <span>
            Ближайший сосед: – чтобы
            определить значение тона пикселя
            масштабированного изображения необходимо найти
            ближайший к нему пиксель исходного изображения и
            задать уровень его тона.
        </span>
        </v-tooltip>
    </v-card-text>
    <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn color="primary" @click="applyResizeSettings">Применить</v-btn>
        <v-btn color="primary" @click="closeResizeDialog">Отменить</v-btn>
        <v-btn color="primary" @click="saveImage">Сохранить</v-btn> 
    </v-card-actions>
    </v-card>
</v-dialog>
</template>

<script>
export default {
props: {
    resizeDialog: Boolean,
    image: Object,
    imageWidth: Number,
    imageHeight: Number
},
data() {
    return {
    min: 12,
    max: 300,
    slider: 100,
    resizeMode: 'percent',
    resizeModes: [
        { text: 'Проценты', value: 'percent' },
        { text: 'Пиксели', value: 'pixels' }
    ],
    scaledWidth: this.imageWidth,
    scaledHeight: this.imageHeight,
    percentWidth: 100,
    percentHeight: 100,
    maintainAspectRatio: true,
    interpolation: 'nearest',
    interpolationMethods: [
        { text: 'Ближайший сосед', value: 'nearest' },
        { text: 'Без интерполяции', value: 'none' }
    ],
    canvas: null
    };
},
computed: {
    sliderPercent() {
    return `${this.slider}%`;
    },
    aspectRatio() {
    return this.image ? this.image.width / this.image.height : 1;
    }
},
methods: {
    onWidthChange(mode) {
    if (this.maintainAspectRatio) {
        if (mode === 'percent') {
        this.percentHeight = (this.percentWidth / this.aspectRatio).toFixed(2);
        } else {
        this.scaledHeight = Math.round(this.scaledWidth / this.aspectRatio);
        }
    }
    this.emitResizeEvent();
    },
    onHeightChange(mode) {
    if (this.maintainAspectRatio) {
        if (mode === 'percent') {
        this.percentWidth = (this.percentHeight * this.aspectRatio).toFixed(2);
        } else {
        this.scaledWidth = Math.round(this.scaledHeight * this.aspectRatio);
        }
    }
    this.emitResizeEvent();
    },
    onSliderChange() {
    const scale = this.slider / 100;
    this.percentWidth = (scale * 100).toFixed(2);
    this.percentHeight = (scale * 100).toFixed(2);
    this.scaledWidth = Math.round(this.image.width * scale);
    this.scaledHeight = Math.round(this.image.height * scale);
    this.emitResizeEvent();
    },
    emitResizeEvent() {
    this.$emit('resize', {
        width: this.resizeMode === 'percent' ? this.image.width * (this.percentWidth / 100) : this.scaledWidth,
        height: this.resizeMode === 'percent' ? this.image.height * (this.percentHeight / 100) : this.scaledHeight
    });
    },
    applyResizeSettings() {
    this.emitResizeEvent();
    this.closeResizeDialog();
    },
    closeResizeDialog() {
    this.$emit('update:resizeDialog', false);
    },
    setCanvasRef(canvas) {
    this.canvas = canvas;
    },
    saveImage() {
    const canvas = this.canvas;
    const link = document.createElement('a');
    link.download = 'image.png';
    link.href = canvas.toDataURL('image/png');
    link.click();
    }
},
watch: {
    interpolation(val) {
    this.$emit('interpolationChange', val);
    }
}
};
</script>
  
<style>
.padding {
    padding-left: 50px;
}
</style>
  