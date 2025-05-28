<template>
  <div class="controls-panel">
    <div class="control-item">
      <label class="control-label">像素大小</label>
      <div class="slider-container">
        <input type="range" min="2" max="64" step="2" v-model="pixelSize" class="pixel-slider" />
        <span class="slider-value">{{ pixelSize }}px</span>
      </div>
    </div>

    <div class="control-item">
      <label class="control-label">调色板</label>
      <select v-model="selectedPalette" class="palette-select">
        <option value="">无调色板</option>
        <option value="gameboy">Game Boy (4色)</option>
        <option value="nes">NES (54色)</option>
        <option value="cga">CGA (4色)</option>
        <option value="grayscale">灰度 (8色)</option>
        <option value="sepia">复古棕褐色</option>
        <option value="neon">霓虹色彩</option>
        <option value="pastel">柔和色调</option>
      </select>
    </div>

    <div class="control-item">
      <label class="control-label">图像处理</label>
      <div class="processing-options">
        <label class="checkbox-label">
          <input type="checkbox" v-model="enableDithering" />
          <span class="checkmark"></span>
          启用抖动效果
        </label>
        <label class="checkbox-label">
          <input type="checkbox" v-model="preserveTransparency" />
          <span class="checkmark"></span>
          保持透明度
        </label>
      </div>
    </div>

    <div class="control-item">
      <label class="control-label">对比度调整</label>
      <div class="slider-container">
        <input
          type="range"
          min="0.5"
          max="2"
          step="0.1"
          v-model="contrast"
          class="contrast-slider"
        />
        <span class="slider-value">{{ contrast }}x</span>
      </div>
    </div>

    <div class="control-item">
      <label class="control-label">亮度调整</label>
      <div class="slider-container">
        <input
          type="range"
          min="0.5"
          max="2"
          step="0.1"
          v-model="brightness"
          class="brightness-slider"
        />
        <span class="slider-value">{{ brightness }}x</span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue'

// 定义事件
const emit = defineEmits<{
  'settings-changed': [
    settings: {
      pixelSize: number
      palette: string
      dithering: boolean
      transparency: boolean
      contrast: number
      brightness: number
    }
  ]
}>()

// 定义属性
const props = defineProps({
  initialPixelSize: {
    type: Number,
    default: 16,
  },
  initialPalette: {
    type: String,
    default: '',
  },
  initialDithering: {
    type: Boolean,
    default: false,
  },
  initialTransparency: {
    type: Boolean,
    default: true,
  },
  initialContrast: {
    type: Number,
    default: 1,
  },
  initialBrightness: {
    type: Number,
    default: 1,
  },
})

// 响应式数据
const pixelSize = ref(props.initialPixelSize)
const selectedPalette = ref(props.initialPalette)
const enableDithering = ref(props.initialDithering)
const preserveTransparency = ref(props.initialTransparency)
const contrast = ref(props.initialContrast)
const brightness = ref(props.initialBrightness)

// 监听属性变化
watch(
  () => props.initialPixelSize,
  (newValue) => {
    pixelSize.value = newValue
  }
)

watch(
  () => props.initialPalette,
  (newValue) => {
    selectedPalette.value = newValue
  }
)

watch(
  () => props.initialDithering,
  (newValue) => {
    enableDithering.value = newValue
  }
)

watch(
  () => props.initialTransparency,
  (newValue) => {
    preserveTransparency.value = newValue
  }
)

watch(
  () => props.initialContrast,
  (newValue) => {
    contrast.value = newValue
  }
)

watch(
  () => props.initialBrightness,
  (newValue) => {
    brightness.value = newValue
  }
)

// 监听设置变化并发射事件
watch(
  [pixelSize, selectedPalette, enableDithering, preserveTransparency, contrast, brightness],
  () => {
    emit('settings-changed', {
      pixelSize: pixelSize.value,
      palette: selectedPalette.value,
      dithering: enableDithering.value,
      transparency: preserveTransparency.value,
      contrast: contrast.value,
      brightness: brightness.value,
    })
  },
  { immediate: false }
)
</script>

<style scoped>
.controls-panel {
  display: flex;
  flex-direction: column;
  gap: 28px;
  background: rgba(255, 255, 255, 0.5);
  border-radius: 16px;
  padding: 24px;
  border: 1px solid rgba(255, 255, 255, 0.3);
  backdrop-filter: blur(10px);
}

.control-item {
  display: flex;
  flex-direction: column;
  gap: 14px;
  padding: 16px;
  background: rgba(255, 255, 255, 0.7);
  border-radius: 12px;
  border: 1px solid rgba(102, 126, 234, 0.1);
  transition: all 0.3s ease;
}

.control-item:hover {
  background: rgba(255, 255, 255, 0.9);
  border-color: rgba(102, 126, 234, 0.2);
  transform: translateY(-1px);
  box-shadow: 0 4px 16px rgba(102, 126, 234, 0.08);
}

.control-label {
  font-size: 15px;
  font-weight: 700;
  color: #475569;
  margin: 0;
  letter-spacing: 0.5px;
}

.slider-container {
  display: flex;
  align-items: center;
  gap: 12px;
}

.pixel-slider,
.contrast-slider,
.brightness-slider {
  flex: 1;
  height: 8px;
  border-radius: 6px;
  background: linear-gradient(135deg, #e2e8f0 0%, #cbd5e1 100%);
  outline: none;
  -webkit-appearance: none;
  appearance: none;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.1);
}

.pixel-slider:hover,
.contrast-slider:hover,
.brightness-slider:hover {
  background: linear-gradient(135deg, #ddd6fe 0%, #c4b5fd 100%);
}

.pixel-slider::-webkit-slider-thumb,
.contrast-slider::-webkit-slider-thumb,
.brightness-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.3);
  transition: all 0.3s ease;
  border: 3px solid white;
}

.pixel-slider::-webkit-slider-thumb:hover,
.contrast-slider::-webkit-slider-thumb:hover,
.brightness-slider::-webkit-slider-thumb:hover {
  background: linear-gradient(135deg, #5a6fd8 0%, #6d28d9 100%);
  transform: scale(1.15);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.4);
}

.pixel-slider::-moz-range-thumb,
.contrast-slider::-moz-range-thumb,
.brightness-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #667eea;
  cursor: pointer;
  border: none;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  transition: all 0.2s ease;
}

.pixel-slider::-moz-range-thumb:hover,
.contrast-slider::-moz-range-thumb:hover,
.brightness-slider::-moz-range-thumb:hover {
  background: #5a6fd8;
  transform: scale(1.1);
}

.pixel-slider:focus,
.contrast-slider:focus,
.brightness-slider:focus {
  background: #dee2e6;
}

.processing-options {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  font-size: 14px;
  color: #495057;
  user-select: none;
}

.checkbox-label input[type='checkbox'] {
  width: 18px;
  height: 18px;
  accent-color: #667eea;
  cursor: pointer;
}

.checkmark {
  position: relative;
}

.checkbox-label:hover {
  color: #667eea;
}

.slider-value {
  min-width: 48px;
  font-size: 14px;
  font-weight: 700;
  color: #667eea;
  text-align: center;
  background: linear-gradient(135deg, #f8fafc 0%, #f1f5f9 100%);
  padding: 8px 12px;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
  box-shadow: 0 2px 4px rgba(102, 126, 234, 0.05);
}

.palette-select {
  width: 100%;
  padding: 14px 18px;
  border: 2px solid #e2e8f0;
  border-radius: 12px;
  background: linear-gradient(135deg, #ffffff 0%, #f8fafc 100%);
  font-size: 15px;
  color: #475569;
  cursor: pointer;
  transition: all 0.3s ease;
  appearance: none;
  background-image: url('data:image/svg+xml;charset=US-ASCII,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 4 5"><path fill="%23667eea" d="M2 0L0 2h4zm0 5L0 3h4z"/></svg>');
  background-repeat: no-repeat;
  background-position: right 16px center;
  background-size: 14px;
  padding-right: 48px;
  font-weight: 600;
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.05);
}

.palette-select:hover {
  border-color: #667eea;
  background: linear-gradient(135deg, #f8fafc 0%, #f1f5f9 100%);
  transform: translateY(-1px);
  box-shadow: 0 4px 16px rgba(102, 126, 234, 0.1);
}

.palette-select:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 4px rgba(102, 126, 234, 0.15);
  background: white;
}

.palette-select option {
  padding: 8px;
  background: white;
  color: #495057;
}

/* 响应式设计 */
@media (max-width: 480px) {
  .controls-panel {
    gap: 20px;
  }

  .control-item {
    gap: 10px;
  }

  .control-label {
    font-size: 13px;
  }

  .palette-select {
    padding: 10px 14px;
    font-size: 13px;
  }

  .slider-value {
    font-size: 13px;
    min-width: 35px;
  }
}
</style>
