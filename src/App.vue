<template>
  <div id="app-container">
    <header class="app-header">
      <h1>🎨 像素画转换器</h1>
      <p class="app-subtitle">将普通图片转换为精美的像素艺术作品</p>
    </header>
    <div class="main-layout">
      <!-- 左侧操作区 -->
      <aside class="sidebar">
        <ImageUploader @image-uploaded="handleImageUpload" />
        <ControlsPanel
          :initialPixelSize="pixelSize"
          :initialPalette="selectedPalette"
          :initialDithering="enableDithering"
          :initialTransparency="preserveTransparency"
          :initialContrast="contrast"
          :initialBrightness="brightness"
          @settings-changed="handleSettingsChange"
        />
        <div class="action-buttons">
          <button
            class="pixelate-btn"
            @click="pixelateImage"
            :disabled="!originalImageUrl || isLoading"
          >
            <span class="btn-icon">✨</span>
            {{ isLoading ? '处理中...' : '生成像素化图片' }}
          </button>
          <DownloadButton
            :canvasElement="pixelatedCanvas"
            :pixelArtResolution="pixelArtResolution"
          />
        </div>
      </aside>
      <!-- 右侧图片与编辑区 -->
      <section class="content-area">
        <div class="image-panels">
          <div class="image-panel original">
            <h2 class="image-title">📷 原始图片</h2>
            <div class="image-container">
              <img
                v-if="originalImageUrl"
                :src="originalImageUrl"
                alt="原始图片"
                class="original-image"
              />
              <div v-else class="placeholder">
                <span class="placeholder-icon">🖼️</span>
                <p>请上传一张图片开始转换</p>
              </div>
            </div>
          </div>
          <div class="image-panel pixelated">
            <h2 class="image-title">🎨 像素化结果</h2>
            <div class="image-container">
              <canvas
                v-show="pixelatedCanvas"
                ref="pixelatedCanvasRef"
                class="pixelated-canvas"
              ></canvas>
              <div v-if="!pixelatedCanvas" class="placeholder">
                <span class="placeholder-icon">✨</span>
                <p>像素化结果将在这里显示</p>
              </div>
            </div>
            <!-- 仅当 pixelatedCanvas 和 pixelArtResolution 都存在时才渲染 PixelEditor -->
            <PixelEditor
              v-if="pixelatedCanvas && pixelArtResolution"
              :canvasElement="pixelatedCanvas"
              :palette="selectedPalette ? palettes[selectedPalette] : []"
              :pixelArtResolution="pixelArtResolution"
              @canvas-updated="handleCanvasUpdate"
            />
          </div>
        </div>
      </section>
    </div>
    <div v-if="isLoading" class="loading-overlay">
      <div class="loading-content">
        <div class="loading-spinner"></div>
        <p>正在处理图片...</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick } from 'vue'
import ImageUploader from './components/ImageUploader.vue'
import ControlsPanel from './components/ControlsPanel.vue'
import DownloadButton from './components/DownloadButton.vue'
import PixelEditor from './components/PixelEditor.vue'

const originalImageUrl = ref<string | null>(null)
const pixelatedCanvas = ref<HTMLCanvasElement | undefined>(undefined) // 实际画布元素引用，用于传递给 DownloadButton/PixelEditor
const pixelatedCanvasRef = ref<HTMLCanvasElement | null>(null) // 模板引用，用于显示画布
const isLoading = ref(false)
const pixelSize = ref(16)
const selectedPalette = ref<keyof typeof palettes | ''>('')
const enableDithering = ref(false)
const preserveTransparency = ref(true)
const contrast = ref(1)
const brightness = ref(1)
// 新增：存储像素画的实际（逻辑）分辨率 (例如，对于 600x450 的图片，像素大小为 10px 时，逻辑分辨率为 60x45)
const pixelArtResolution = ref<{ width: number; height: number } | null>(null)

const palettes: Record<string, number[][]> = {
  gameboy: [
    [15, 56, 15],
    [48, 98, 48],
    [139, 172, 15],
    [155, 188, 15],
  ],
  nes: [
    [124, 124, 124],
    [0, 0, 252],
    [0, 0, 188],
    [68, 40, 188],
    [148, 0, 132],
    [168, 0, 32],
    [168, 16, 0],
    [136, 20, 0],
    [80, 48, 0],
    [0, 120, 0],
    [0, 104, 0],
    [0, 88, 0],
    [0, 64, 88],
    [0, 0, 0],
    [188, 188, 188],
    [0, 120, 248],
    [0, 88, 248],
    [104, 68, 252],
    [216, 0, 204],
    [228, 0, 88],
    [248, 56, 0],
    [228, 92, 16],
    [172, 124, 0],
    [0, 184, 0],
    [0, 168, 0],
    [0, 168, 68],
    [0, 136, 136],
    [248, 248, 248],
    [60, 188, 252],
    [104, 136, 252],
    [152, 120, 248],
    [248, 120, 248],
    [248, 88, 152],
    [248, 120, 88],
    [252, 160, 68],
    [248, 184, 0],
    [184, 248, 24],
    [88, 216, 84],
    [88, 248, 152],
    [0, 232, 216],
    [120, 120, 120],
    [252, 252, 252],
    [164, 228, 252],
    [184, 184, 248],
    [216, 184, 248],
    [248, 184, 248],
    [248, 164, 192],
    [248, 184, 172],
    [252, 216, 168],
    [248, 216, 120],
    [216, 248, 120],
    [184, 248, 184],
    [184, 248, 216],
    [0, 252, 252],
    [248, 216, 248],
  ],
  cga: [
    [0, 0, 0],
    [85, 255, 255],
    [255, 85, 255],
    [255, 255, 255],
  ],
  grayscale: [
    [0, 0, 0],
    [36, 36, 36],
    [72, 72, 72],
    [108, 108, 108],
    [144, 144, 144],
    [180, 180, 180],
    [216, 216, 216],
    [255, 255, 255],
  ],
  sepia: [
    [112, 66, 20],
    [149, 102, 49],
    [196, 143, 78],
    [222, 184, 135],
    [245, 222, 179],
    [255, 248, 220],
  ],
  neon: [
    [255, 0, 255],
    [0, 255, 255],
    [255, 255, 0],
    [255, 0, 0],
    [0, 255, 0],
    [0, 0, 255],
    [255, 128, 0],
    [128, 255, 0],
  ],
  pastel: [
    [255, 179, 186],
    [255, 223, 186],
    [255, 255, 186],
    [186, 255, 201],
    [186, 225, 255],
    [186, 186, 255],
    [255, 186, 255],
    [255, 255, 255],
  ],
}

const handleImageUpload = (file: File | null) => {
  if (file) {
    // 释放旧的 Blob URL 内存
    if (originalImageUrl.value) {
      URL.revokeObjectURL(originalImageUrl.value)
    }
    originalImageUrl.value = URL.createObjectURL(file)
    clearPixelatedCanvas()
    pixelArtResolution.value = null // 上传新图片时重置逻辑分辨率
  } else {
    // 如果没有文件（例如取消上传），则清空所有状态
    if (originalImageUrl.value) {
      URL.revokeObjectURL(originalImageUrl.value)
    }
    originalImageUrl.value = null
    clearPixelatedCanvas()
    pixelArtResolution.value = null
  }
}

const handleSettingsChange = (settings: {
  pixelSize: number
  palette: keyof typeof palettes | ''
  dithering: boolean
  transparency: boolean
  contrast: number
  brightness: number
}) => {
  pixelSize.value = settings.pixelSize
  selectedPalette.value = settings.palette
  enableDithering.value = settings.dithering
  preserveTransparency.value = settings.transparency
  contrast.value = settings.contrast
  brightness.value = settings.brightness
  // 设置改变后，如果已上传图片且不在处理中，则重新生成像素化图片
  if (originalImageUrl.value && !isLoading.value) {
    pixelateImage()
  }
}

// 处理 PixelEditor 返回的低分辨率画布更新
const handleCanvasUpdate = (updatedLowResCanvas: HTMLCanvasElement) => {
  if (pixelatedCanvasRef.value && updatedLowResCanvas && pixelArtResolution.value) {
    const ctx = pixelatedCanvasRef.value.getContext('2d')
    if (ctx) {
      // 清空当前的显示画布
      ctx.clearRect(0, 0, pixelatedCanvasRef.value.width, pixelatedCanvasRef.value.height)

      // 将 PixelEditor 返回的低分辨率画布缩放绘制到显示画布上
      // pixelatedCanvasRef.value 的尺寸已经在 processImage 中正确设置
      ctx.imageSmoothingEnabled = false // 确保缩放时保持像素化效果
      ctx.drawImage(
        updatedLowResCanvas,
        0,
        0,
        updatedLowResCanvas.width,
        updatedLowResCanvas.height, // 源图像 (低分辨率)
        0,
        0,
        pixelatedCanvasRef.value.width,
        pixelatedCanvasRef.value.height // 目标显示画布
      )
      // 重要：更新 pixelatedCanvas 引用，使其指向当前的显示画布
      // DownloadButton 和 PixelEditor 会接收到这个引用
      pixelatedCanvas.value = pixelatedCanvasRef.value
    }
  }
}

const clearPixelatedCanvas = () => {
  pixelatedCanvas.value = undefined // 取消引用，触发模板中的 v-if
  pixelArtResolution.value = null // 清除逻辑分辨率
  if (pixelatedCanvasRef.value) {
    const ctx = pixelatedCanvasRef.value.getContext('2d')
    if (ctx) {
      ctx.clearRect(0, 0, pixelatedCanvasRef.value.width, pixelatedCanvasRef.value.height)
      // 将画布尺寸重置为 0，防止前一张图片残留
      pixelatedCanvasRef.value.width = 0
      pixelatedCanvasRef.value.height = 0
    }
  }
}

// --- 新增图片加载 Promise 封装 ---
const loadImage = (url: string): Promise<HTMLImageElement> => {
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.crossOrigin = 'anonymous' // 启用跨域，以防图片来自不同源
    img.onload = () => resolve(img)
    img.onerror = () => reject(new Error('图片加载失败，请重试或检查图片URL。'))
    img.src = url
  })
}
// --- 结束新增 ---

const pixelateImage = async () => {
  if (!originalImageUrl.value) {
    alert('请先上传图片')
    return
  }
  console.log('开始像素化处理，设置 isLoading = true')
  isLoading.value = true // 设置加载状态为 true

  // 添加超时保护，确保 isLoading 最终会被重置
  const timeoutId = setTimeout(() => {
    console.warn('处理超时，强制重置 isLoading')
    isLoading.value = false
    alert('处理超时，请重试')
  }, 10000) // 10秒超时

  try {
    console.log('开始加载图片:', originalImageUrl.value)
    // 使用 Promise 封装的 loadImage 函数来加载图片
    const img = await loadImage(originalImageUrl.value)
    console.log('图片加载成功，尺寸:', img.width, 'x', img.height)
    console.log('开始处理图片')
    await processImage(img) // 等待处理完成
    clearTimeout(timeoutId) // 清除超时定时器
  } catch (error) {
    // 捕获图片加载或处理过程中的任何错误
    console.error('像素化处理错误:', error)
    clearTimeout(timeoutId) // 清除超时定时器
    isLoading.value = false // 确保在错误时重置加载状态
    alert(`处理失败，请重试。错误信息: ${error instanceof Error ? error.message : String(error)}`)
  }
}

const processImage = (img: HTMLImageElement): Promise<void> => {
  return new Promise((resolve, reject) => {
    try {
      console.log('processImage 开始执行')
      const maxWidth = 600
      const maxHeight = 450
      let outputWidth = img.width
      let outputHeight = img.height
      const aspectRatio = img.width / img.height

      console.log('原始图片尺寸:', img.width, 'x', img.height)
      console.log('当前像素大小设置:', pixelSize.value)

      // 根据最大宽度/高度等比例缩放图片，以适应显示区域
      if (outputWidth > maxWidth) {
        outputWidth = maxWidth
        outputHeight = outputWidth / aspectRatio
      }
      if (outputHeight > maxHeight) {
        outputHeight = maxHeight
        outputWidth = outputHeight * aspectRatio
      }

      // 四舍五入到整数像素，用于显示尺寸
      outputWidth = Math.round(outputWidth)
      outputHeight = Math.round(outputHeight)

      // 计算实际的逻辑像素艺术分辨率 (像素块数量)
      // 确保 blocksX 和 blocksY 至少为 1
      const blocksX = Math.max(1, Math.floor(outputWidth / pixelSize.value))
      const blocksY = Math.max(1, Math.floor(outputHeight / pixelSize.value))

      console.log('计算后的显示尺寸:', outputWidth, 'x', outputHeight)
      console.log('计算后的逻辑分辨率:', blocksX, 'x', blocksY)

      // 存储此逻辑分辨率，供 PixelEditor 使用
      pixelArtResolution.value = { width: blocksX, height: blocksY }
      console.log('pixelArtResolution 已设置')

      nextTick(() => {
        try {
          console.log('开始处理画布，nextTick 回调执行')
          const canvas = pixelatedCanvasRef.value
          if (!canvas) {
            isLoading.value = false
            console.error('Canvas 元素未准备好。')
            return
          }
          console.log('Canvas 元素已准备好，尺寸:', outputWidth, 'x', outputHeight)

          // 设置显示画布的尺寸
          canvas.width = outputWidth
          canvas.height = outputHeight
          const ctx = canvas.getContext('2d')
          if (!ctx) {
            isLoading.value = false
            console.error('无法获取 Canvas 2D 上下文。')
            return
          }
          console.log('Canvas 2D 上下文获取成功')

          ctx.imageSmoothingEnabled = false // 像素艺术显示的关键

          console.log('开始应用像素化效果，逻辑分辨率:', blocksX, 'x', blocksY)
          // 应用像素化效果 (绘制到一个 blocksX/blocksY 的临时画布，然后缩放绘制到 outputWidth/outputHeight 的显示画布上)
          applyPixelEffect(ctx, img, outputWidth, outputHeight, blocksX, blocksY)
          console.log('像素化效果应用完成')

          // 关键：处理完成后，将 pixelatedCanvas 更新为当前的画布元素
          // 这样 DownloadButton 和 PixelEditor 就能接收到最新的画布内容
          pixelatedCanvas.value = canvas
          console.log('pixelatedCanvas 更新完成，准备重置 isLoading')
          isLoading.value = false // 成功完成处理，重置加载状态
          console.log('isLoading 已重置为 false，处理完成')
          resolve() // 处理完成，解析 Promise
        } catch (error) {
          console.error('nextTick 回调中发生错误:', error)
          isLoading.value = false
          reject(error) // 处理失败，拒绝 Promise
        }
      })
    } catch (error) {
      isLoading.value = false // 确保在处理函数内部的错误中重置加载状态
      console.error('图片处理函数内部错误:', error)
      reject(error) // 处理失败，拒绝 Promise
    }
  })
}

const applyPixelEffect = (
  ctx: CanvasRenderingContext2D,
  img: HTMLImageElement,
  outputWidth: number, // 显示宽度
  outputHeight: number, // 显示高度
  blocksX: number, // 逻辑像素艺术宽度 (块数)
  blocksY: number // 逻辑像素艺术高度 (块数)
) => {
  const tempCanvas = document.createElement('canvas')
  const tempCtx = tempCanvas.getContext('2d')
  if (!tempCtx) return

  // 临时画布的尺寸是实际的低分辨率像素艺术网格
  tempCanvas.width = blocksX
  tempCanvas.height = blocksY

  tempCtx.imageSmoothingEnabled = false // 初始像素化步骤的关键

  // 将原始图片缩放绘制到逻辑像素艺术分辨率的临时画布上
  tempCtx.drawImage(img, 0, 0, blocksX, blocksY)

  // 对低分辨率图像数据应用亮度/对比度
  if (brightness.value !== 1 || contrast.value !== 1) {
    applyBrightnessContrast(tempCtx, blocksX, blocksY)
  }

  // 对低分辨率图像数据应用调色板和抖动
  if (selectedPalette.value && palettes[selectedPalette.value]) {
    applyPalette(tempCtx, blocksX, blocksY, palettes[selectedPalette.value])
  }

  // 将低分辨率的临时画布绘制到主显示画布上，并进行放大
  // 主画布已禁用平滑处理。
  ctx.drawImage(tempCanvas, 0, 0, outputWidth, outputHeight)
}

const applyBrightnessContrast = (ctx: CanvasRenderingContext2D, width: number, height: number) => {
  const imageData = ctx.getImageData(0, 0, width, height)
  const data = imageData.data
  for (let i = 0; i < data.length; i += 4) {
    let r = ((data[i] / 255 - 0.5) * contrast.value + 0.5) * 255
    let g = ((data[i + 1] / 255 - 0.5) * contrast.value + 0.5) * 255
    let b = ((data[i + 2] / 255 - 0.5) * contrast.value + 0.5) * 255
    r = r * brightness.value
    g = g * brightness.value
    b = b * brightness.value
    data[i] = Math.max(0, Math.min(255, r))
    data[i + 1] = Math.max(0, Math.min(255, g))
    data[i + 2] = Math.max(0, Math.min(255, b))
  }
  ctx.putImageData(imageData, 0, 0)
}

const applyPalette = (
  ctx: CanvasRenderingContext2D,
  width: number,
  height: number,
  palette: number[][]
) => {
  const imageData = ctx.getImageData(0, 0, width, height)
  const data = imageData.data
  for (let i = 0; i < data.length; i += 4) {
    const r = data[i]
    const g = data[i + 1]
    const b = data[i + 2]
    const a = data[i + 3]
    if (preserveTransparency.value && a < 128) {
      continue
    }
    let closestColor: number[]
    // 传递像素索引而非字节索引 (i / 4)
    if (enableDithering.value) {
      closestColor = findClosestColorWithDithering(r, g, b, palette, i / 4, width)
    } else {
      closestColor = findClosestColor(r, g, b, palette)
    }
    data[i] = closestColor[0]
    data[i + 1] = closestColor[1]
    data[i + 2] = closestColor[2]
  }
  ctx.putImageData(imageData, 0, 0)
}

const findClosestColor = (r: number, g: number, b: number, palette: number[][]): number[] => {
  let minDistance = Infinity
  let closestColor = palette[0] // 默认第一个颜色
  if (!palette || palette.length === 0) return [r, g, b] // 如果调色板为空，则返回原始颜色

  for (const color of palette) {
    const distance = Math.sqrt(
      Math.pow(r - color[0], 2) + Math.pow(g - color[1], 2) + Math.pow(b - color[2], 2)
    )
    if (distance < minDistance) {
      minDistance = distance
      closestColor = color
    }
  }
  return closestColor
}

const findClosestColorWithDithering = (
  r: number,
  g: number,
  b: number,
  palette: number[][],
  pixelIndex: number, // 当前像素的索引
  width: number // 图像宽度 (像素数)
): number[] => {
  const x = pixelIndex % width
  const y = Math.floor(pixelIndex / width)
  // 简单的有序抖动 (类似 Bayer 算法)
  const noise = ((x + y) % 2) * 16 - 8
  const adjustedR = Math.max(0, Math.min(255, r + noise))
  const adjustedG = Math.max(0, Math.min(255, g + noise))
  const adjustedB = Math.max(0, Math.min(255, b + noise))
  return findClosestColor(adjustedR, adjustedG, adjustedB, palette)
}
</script>

<style>
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
  margin: 0;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: #333;
  min-height: 100vh;
  box-sizing: border-box;
}

* {
  box-sizing: border-box;
}

#app-container {
  width: 100%;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.app-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  text-align: center;
  padding: 30px 20px;
  box-shadow: 0 8px 32px rgba(102, 126, 234, 0.12);
  position: relative;
  overflow: hidden;
  flex-shrink: 0;
}

.app-header::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: url('data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 100 100"><defs><pattern id="grain" width="100" height="100" patternUnits="userSpaceOnUse"><circle cx="25" cy="25" r="1" fill="white" opacity="0.1"/><circle cx="75" cy="75" r="1" fill="white" opacity="0.1"/><circle cx="50" cy="10" r="0.5" fill="white" opacity="0.15"/><circle cx="10" cy="60" r="0.5" fill="white" opacity="0.15"/><circle cx="90" cy="40" r="0.5" fill="white" opacity="0.15"/></pattern></defs><rect width="100" height="100" fill="url(%23grain)"/></svg>');
  pointer-events: none;
}

.app-header h1 {
  margin: 0 0 10px 0;
  font-size: 2.8em;
  font-weight: 800;
  letter-spacing: 2px;
  position: relative;
  z-index: 1;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.app-subtitle {
  margin: 0;
  font-size: 1.2em;
  opacity: 0.92;
  font-weight: 500;
  position: relative;
  z-index: 1;
}

.main-layout {
  display: flex;
  flex-direction: column;
  gap: 24px;
  padding: 24px;
  flex: 1;
  overflow-y: auto;
  min-height: 0;
}

.sidebar {
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 20px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 20px;
  padding: 20px;
  box-shadow: 0 8px 32px rgba(102, 126, 234, 0.15);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.3);
  order: 1;
}

.content-area {
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 24px;
  order: 2;
}

.image-panels {
  display: flex;
  flex-direction: column;
  gap: 20px;
  width: 100%;
}

.image-panel {
  width: 100%;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 20px;
  padding: 20px;
  box-shadow: 0 8px 32px rgba(102, 126, 234, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.3);
  display: flex;
  flex-direction: column;
  backdrop-filter: blur(10px);
  transition: all 0.3s ease;
}

.image-panel:hover {
  transform: translateY(-2px);
  box-shadow: 0 12px 40px rgba(102, 126, 234, 0.15);
}

.image-title {
  margin: 0 0 20px 0;
  color: #495057;
  font-size: 1.2em;
  font-weight: 700;
  letter-spacing: 1px;
  text-align: center;
  padding-bottom: 12px;
  border-bottom: 2px solid #f1f5f9;
}

.image-container {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 400px;
  border-radius: 16px;
  background: linear-gradient(135deg, #f8fafc 0%, #f1f5f9 100%);
  border: 2px dashed #e2e8f0;
  position: relative;
  overflow: hidden;
  transition: all 0.3s ease;
}
.image-container:hover {
  border-color: #667eea;
  background: linear-gradient(135deg, #f0f8ff 0%, #e6f3ff 100%);
  transform: scale(1.01);
}

.image-display,
.canvas-display {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.original-image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.08);
}

.pixelated-canvas {
  max-width: 100%;
  max-height: 100%;
  border-radius: 8px;
  image-rendering: pixelated;
  image-rendering: -moz-crisp-edges;
  image-rendering: crisp-edges;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.1);
  background: white;
}

.action-buttons {
  margin: 28px 0 16px 0;
  text-align: center;
}

.pixelate-btn {
  background: linear-gradient(135deg, #667eea 0%, #20c997 100%);
  color: white;
  border: none;
  padding: 16px 36px;
  border-radius: 12px;
  font-size: 1.1em;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.3s ease;
  display: inline-flex;
  align-items: center;
  gap: 12px;
  min-width: 200px;
  box-shadow: 0 4px 16px rgba(102, 126, 234, 0.12);
  position: relative;
  overflow: hidden;
}

.pixelate-btn::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  transition: left 0.5s;
}

.pixelate-btn:hover:not(:disabled)::before {
  left: 100%;
}

.pixelate-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #5a6fd8 0%, #1ea085 100%);
  transform: translateY(-3px) scale(1.05);
  box-shadow: 0 12px 32px rgba(102, 126, 234, 0.25);
}
.pixelate-btn:disabled {
  background: linear-gradient(135deg, #bfc8e6 0%, #a8b5d1 100%);
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.btn-icon {
  font-size: 1.2em;
}

.placeholder {
  text-align: center;
  color: #94a3b8;
  padding: 60px 20px;
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.05) 0%, rgba(118, 75, 162, 0.05) 100%);
  border-radius: 12px;
  border: 1px dashed #cbd5e1;
}
.placeholder-icon {
  font-size: 64px;
  display: block;
  margin-bottom: 20px;
  opacity: 0.6;
  animation: float 3s ease-in-out infinite;
}
.placeholder p {
  margin: 0;
  font-size: 1.1em;
  font-weight: 500;
  color: #64748b;
}

@keyframes float {
  0%,
  100% {
    transform: translateY(0px);
  }
  50% {
    transform: translateY(-10px);
  }
}

.loading-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.85);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  backdrop-filter: blur(12px);
  animation: fadeIn 0.3s ease-out;
}
.loading-content {
  text-align: center;
  color: white;
  background: rgba(255, 255, 255, 0.15);
  padding: 60px 48px;
  border-radius: 24px;
  border: 2px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(20px);
}
.loading-spinner {
  width: 56px;
  height: 56px;
  border: 6px solid rgba(255, 255, 255, 0.2);
  border-top: 6px solid #667eea;
  border-right: 6px solid #20c997;
  border-radius: 50%;
  animation: spin 1.2s linear infinite;
  margin: 0 auto 28px;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

/* 移除 .editor-section，因为 PixelEditor 组件自身会管理其样式 */

/* 大屏幕布局 */
@media (min-width: 1200px) {
  .main-layout {
    flex-direction: row;
    gap: 32px;
    padding: 32px;
  }

  .sidebar {
    width: 320px;
    flex: 0 0 320px;
    order: 0;
  }

  .content-area {
    flex: 1;
    order: 1;
  }

  .image-panels {
    flex-direction: row;
    gap: 24px;
  }

  .image-panel {
    flex: 1;
  }

  .image-container {
    min-height: 500px;
  }
}

/* 中等屏幕布局 */
@media (min-width: 768px) and (max-width: 1199px) {
  .main-layout {
    padding: 24px 20px;
    gap: 24px;
  }

  .image-panels {
    flex-direction: row;
    gap: 20px;
  }

  .image-panel {
    flex: 1;
  }

  .image-container {
    min-height: 400px;
  }
}

/* 小屏幕布局 */
@media (max-width: 767px) {
  .app-header {
    padding: 20px 16px;
  }

  .app-header h1 {
    font-size: 2em;
    margin-bottom: 8px;
  }

  .app-subtitle {
    font-size: 1em;
  }

  .main-layout {
    padding: 16px 12px;
    gap: 16px;
  }

  .sidebar {
    padding: 16px;
    gap: 16px;
    border-radius: 16px;
  }

  .image-panel {
    padding: 16px;
    border-radius: 16px;
  }

  .image-container {
    min-height: 300px;
  }

  .pixelate-btn {
    width: 100%;
    padding: 14px 24px;
    min-width: auto;
  }
}

/* 超小屏幕布局 */
@media (max-width: 480px) {
  .app-header {
    padding: 16px 12px;
  }

  .app-header h1 {
    font-size: 1.8em;
  }

  .main-layout {
    padding: 12px 8px;
    gap: 12px;
  }

  .sidebar {
    padding: 12px;
    gap: 12px;
  }

  .image-panel {
    padding: 12px;
  }

  .image-container {
    min-height: 250px;
  }

  .placeholder {
    padding: 30px 12px;
  }

  .placeholder-icon {
    font-size: 40px;
  }
}
</style>
