<template>
  <div class="pixel-editor" v-if="canvasElement && pixelArtResolution">
    <div class="editor-header">
      <h3 class="editor-title">🎨 像素编辑器</h3>
      <div class="editor-controls">
        <button @click="toggleEditMode" class="edit-btn" :class="{ active: isEditMode }">
          <span class="btn-icon">✏️</span>
          {{ isEditMode ? '退出编辑' : '编辑模式' }}
        </button>
        <button @click="resetCanvas" class="reset-btn" :disabled="!isEditMode">
          <span class="btn-icon">🔄</span>
          重置
        </button>
      </div>
    </div>

    <div class="editor-content" v-if="isEditMode">
      <div class="color-palette">
        <h4>调色板</h4>
        <div class="palette-colors">
          <div
            v-for="(color, index) in currentPalette"
            :key="index"
            class="color-swatch"
            :class="{ selected: selectedColor === color }"
            :style="{ backgroundColor: `rgb(${color[0]}, ${color[1]}, ${color[2]})` }"
            @click="selectColor(color)"
          ></div>
        </div>
      </div>

      <div class="brush-settings">
        <label class="setting-label">画笔大小</label>
        <div class="brush-size-controls">
          <button
            v-for="size in [1, 2, 3, 4]"
            :key="size"
            class="brush-size-btn"
            :class="{ active: brushSize === size }"
            @click="setBrushSize(size)"
          >
            {{ size }}x{{ size }}
          </button>
        </div>
      </div>
    </div>

    <div class="canvas-container">
      <canvas
        ref="editableCanvas"
        class="editable-canvas"
        :class="{ 'edit-mode': isEditMode }"
        @mousedown="startDrawing"
        @mousemove="draw"
        @mouseup="stopDrawing"
        @mouseleave="stopDrawing"
      ></canvas>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, nextTick, onMounted } from 'vue'

// 定义事件
const emit = defineEmits<{
  'canvas-updated': [canvas: HTMLCanvasElement]
}>()

// 定义属性
const props = defineProps<{
  canvasElement: HTMLCanvasElement | null
  palette: number[][]
  pixelArtResolution: { width: number; height: number } | null
}>()

// 响应式数据
const editableCanvas = ref<HTMLCanvasElement | null>(null)
const isEditMode = ref(false)
const isDrawing = ref(false)
const selectedColor = ref<number[]>([0, 0, 0])
const brushSize = ref(1)
const currentPalette = ref<number[][]>([])
// 新增：存储低分辨率的像素艺术画布，这是实际编辑的画布
const lowResCanvas = ref<HTMLCanvasElement | null>(null)

// 默认调色板
const defaultPalette = [
  [0, 0, 0],
  [255, 255, 255],
  [255, 0, 0],
  [0, 255, 0],
  [0, 0, 255],
  [255, 255, 0],
  [255, 0, 255],
  [0, 255, 255],
]

// 复制原始画布到编辑器，并创建低分辨率编辑画布
const copyCanvasToEditor = () => {
  if (!props.canvasElement || !editableCanvas.value || !props.pixelArtResolution) return

  nextTick(() => {
    const sourceCanvas = props.canvasElement!
    const targetCanvas = editableCanvas.value!
    const resolution = props.pixelArtResolution!

    // 显示画布保持与源画布相同的尺寸
    targetCanvas.width = sourceCanvas.width
    targetCanvas.height = sourceCanvas.height

    const targetCtx = targetCanvas.getContext('2d')!
    targetCtx.imageSmoothingEnabled = false
    targetCtx.drawImage(sourceCanvas, 0, 0)

    // 创建低分辨率编辑画布
    if (!lowResCanvas.value) {
      lowResCanvas.value = document.createElement('canvas')
    }

    const lowResCtx = lowResCanvas.value.getContext('2d')!
    lowResCanvas.value.width = resolution.width
    lowResCanvas.value.height = resolution.height
    lowResCtx.imageSmoothingEnabled = false

    // 从显示画布中提取低分辨率像素数据
    // 通过将显示画布缩小到逻辑分辨率来获取像素艺术数据
    lowResCtx.drawImage(sourceCanvas, 0, 0, resolution.width, resolution.height)
  })
}

// 监听画布变化
watch(
  () => props.canvasElement,
  (newCanvas) => {
    if (newCanvas) {
      copyCanvasToEditor()
    }
  },
  { immediate: true }
)

// 监听调色板变化
watch(
  () => props.palette,
  (newPalette) => {
    if (newPalette && newPalette.length > 0) {
      currentPalette.value = newPalette
      selectedColor.value = newPalette[0]
    } else {
      currentPalette.value = defaultPalette
      selectedColor.value = defaultPalette[0]
    }
  },
  { immediate: true }
)

// 切换编辑模式
const toggleEditMode = () => {
  isEditMode.value = !isEditMode.value
  if (isEditMode.value) {
    copyCanvasToEditor()
  }
}

// 重置画布
const resetCanvas = () => {
  copyCanvasToEditor()
}

// 选择颜色
const selectColor = (color: number[]) => {
  selectedColor.value = color
}

// 设置画笔大小
const setBrushSize = (size: number) => {
  brushSize.value = size
}

// 开始绘制
const startDrawing = (event: MouseEvent) => {
  if (!isEditMode.value) return
  isDrawing.value = true
  draw(event)
}

// 绘制 - 核心修复：在逻辑像素上绘制，而不是显示像素上
const draw = (event: MouseEvent) => {
  if (
    !isDrawing.value ||
    !isEditMode.value ||
    !lowResCanvas.value ||
    !editableCanvas.value ||
    !props.pixelArtResolution
  )
    return

  const displayCanvas = editableCanvas.value
  const rect = displayCanvas.getBoundingClientRect()

  // 计算鼠标在显示画布上的位置
  const displayX = (event.clientX - rect.left) * (displayCanvas.width / rect.width)
  const displayY = (event.clientY - rect.top) * (displayCanvas.height / rect.height)

  // 将显示坐标转换为逻辑像素坐标
  const logicalX = Math.floor((displayX / displayCanvas.width) * props.pixelArtResolution.width)
  const logicalY = Math.floor((displayY / displayCanvas.height) * props.pixelArtResolution.height)

  const lowResCtx = lowResCanvas.value.getContext('2d')!
  lowResCtx.imageSmoothingEnabled = false

  // 在低分辨率画布上绘制（逻辑像素）
  for (let i = 0; i < brushSize.value; i++) {
    for (let j = 0; j < brushSize.value; j++) {
      const pixelX = logicalX + i
      const pixelY = logicalY + j

      if (
        pixelX >= 0 &&
        pixelX < props.pixelArtResolution.width &&
        pixelY >= 0 &&
        pixelY < props.pixelArtResolution.height
      ) {
        lowResCtx.fillStyle = `rgb(${selectedColor.value[0]}, ${selectedColor.value[1]}, ${selectedColor.value[2]})`
        lowResCtx.fillRect(pixelX, pixelY, 1, 1)
      }
    }
  }

  // 将低分辨率画布缩放绘制到显示画布上
  const displayCtx = displayCanvas.getContext('2d')!
  displayCtx.imageSmoothingEnabled = false
  displayCtx.clearRect(0, 0, displayCanvas.width, displayCanvas.height)
  displayCtx.drawImage(lowResCanvas.value, 0, 0, displayCanvas.width, displayCanvas.height)

  // 发射更新事件，传递低分辨率画布
  emit('canvas-updated', lowResCanvas.value)
}

// 停止绘制
const stopDrawing = () => {
  isDrawing.value = false
}

// 组件挂载时初始化
onMounted(() => {
  if (props.canvasElement) {
    copyCanvasToEditor()
  }
})
</script>

<style scoped>
.pixel-editor {
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.9) 0%, rgba(248, 250, 252, 0.9) 100%);
  border-radius: 20px;
  padding: 28px;
  margin-top: 24px;
  border: 1px solid rgba(255, 255, 255, 0.3);
  backdrop-filter: blur(10px);
  box-shadow: 0 8px 32px rgba(102, 126, 234, 0.1);
}

.editor-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
  padding-bottom: 16px;
  border-bottom: 2px solid rgba(102, 126, 234, 0.1);
}

.editor-title {
  margin: 0;
  font-size: 20px;
  font-weight: 700;
  color: #475569;
  letter-spacing: 0.5px;
}

.editor-controls {
  display: flex;
  gap: 10px;
}

.edit-btn,
.reset-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 20px;
  border: none;
  border-radius: 12px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  position: relative;
  overflow: hidden;
}

.edit-btn {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.edit-btn:hover {
  background: linear-gradient(135deg, #5a6fd8 0%, #6d28d9 100%);
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgba(102, 126, 234, 0.3);
}

.edit-btn.active {
  background: linear-gradient(135deg, #10b981 0%, #059669 100%);
  box-shadow: 0 4px 16px rgba(16, 185, 129, 0.3);
}

.edit-btn.active:hover {
  background: linear-gradient(135deg, #059669 0%, #047857 100%);
}

.reset-btn {
  background: linear-gradient(135deg, #6b7280 0%, #4b5563 100%);
  color: white;
}

.reset-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #4b5563 0%, #374151 100%);
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgba(107, 114, 128, 0.3);
}

.reset-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.editor-content {
  display: flex;
  gap: 30px;
  margin-bottom: 20px;
}

.color-palette h4,
.brush-settings .setting-label {
  margin: 0 0 12px 0;
  font-size: 14px;
  font-weight: 600;
  color: #495057;
}

.palette-colors {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 8px;
}

.color-swatch {
  width: 36px;
  height: 36px;
  border-radius: 8px;
  cursor: pointer;
  border: 3px solid transparent;
  transition: all 0.3s ease;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  position: relative;
}

.color-swatch:hover {
  transform: scale(1.15);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.2);
}

.color-swatch.selected {
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.3), 0 4px 16px rgba(102, 126, 234, 0.2);
  transform: scale(1.1);
}

.color-swatch.selected::after {
  content: '✓';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: white;
  font-weight: bold;
  font-size: 14px;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.5);
}

.brush-size-controls {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.brush-size-btn {
  padding: 8px 14px;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  background: linear-gradient(135deg, #ffffff 0%, #f8fafc 100%);
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  min-width: 40px;
  text-align: center;
}

.brush-size-btn:hover {
  border-color: #667eea;
  background: linear-gradient(135deg, #f0f8ff 0%, #e6f3ff 100%);
  transform: translateY(-1px);
}

.brush-size-btn.active {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-color: #667eea;
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.3);
}

.canvas-container {
  display: flex;
  justify-content: center;
  background: white;
  border-radius: 8px;
  padding: 20px;
  border: 2px solid #dee2e6;
}

.editable-canvas {
  max-width: 100%;
  height: auto;
  border-radius: 4px;
  image-rendering: pixelated;
  image-rendering: -moz-crisp-edges;
  image-rendering: crisp-edges;
}

.editable-canvas.edit-mode {
  cursor: crosshair;
  border: 2px solid #667eea;
}

.btn-icon {
  font-size: 14px;
}
</style>
