## 像素画转换器 (Pixelator App) 后续开发方案

### 阶段一：核心优化与基础增强 (优先级：高)

此阶段的目标是提升代码质量、引入更好的状态管理、优化性能并完善用户体验的基础功能。

#### 1.1 状态管理优化 (Pinia 引入)

- **当前问题**: 目前所有状态都集中在 `App.vue` 中，导致组件逻辑复杂，数据流不清晰。`package.json` 中已包含 `pinia` 依赖，但未实际使用。
- **解决方案**: 引入 Pinia 作为全局状态管理方案。

  - **任务 1.1.1**: 创建 Pinia store，例如 `settingsStore` (用于存储 `pixelSize`, `selectedPalette`, `enableDithering`, `preserveTransparency`, `contrast`, `brightness` 等设置)。

    - 文件提议:

    ```typescript
    // src/stores/settings.ts
    import { defineStore } from 'pinia'

    interface SettingsState {
      pixelSize: number
      selectedPalette: string
      enableDithering: boolean
      preserveTransparency: boolean
      contrast: number
      brightness: number
      // ...
    }

    export const useSettingsStore = defineStore('settings', {
      state: (): SettingsState => ({
        pixelSize: 16,
        selectedPalette: '',
        enableDithering: false,
        preserveTransparency: true,
        contrast: 1,
        brightness: 1,
      }),
      actions: {
        updateSettings(newSettings: Partial<SettingsState>) {
          Object.assign(this, newSettings)
        },
        resetSettings() {
          this.pixelSize = 16
          this.selectedPalette = ''
          this.enableDithering = false
          this.preserveTransparency = true
          this.contrast = 1
          this.brightness = 1
        },
      },
    })
    ```

  - **任务 1.1.2**: 创建 `imageStore` (用于存储 `originalImageUrl`, `pixelatedCanvas` 等图片相关状态)。

    - 文件提议:

    ```typescript
    // src/stores/image.ts
    import { defineStore } from 'pinia'
    import { ref } from 'vue'

    export const useImageStore = defineStore('image', () => {
      const originalImageUrl = ref<string | null>(null)
      const pixelatedCanvas = ref<HTMLCanvasElement | undefined>(undefined)
      const pixelArtResolution = ref<{ width: number; height: number } | null>(null)
      const isLoading = ref(false)

      function setOriginalImage(url: string | null) {
        if (originalImageUrl.value) {
          URL.revokeObjectURL(originalImageUrl.value) // Revoke previous URL
        }
        originalImageUrl.value = url
      }

      function setPixelatedCanvas(canvas: HTMLCanvasElement | undefined) {
        pixelatedCanvas.value = canvas
      }

      function setPixelArtResolution(resolution: { width: number; height: number } | null) {
        pixelArtResolution.value = resolution
      }

      function setLoading(loading: boolean) {
        isLoading.value = loading
      }

      function resetImageState() {
        setOriginalImage(null)
        setPixelatedCanvas(undefined)
        setPixelArtResolution(null)
        setLoading(false)
      }

      return {
        originalImageUrl,
        pixelatedCanvas,
        pixelArtResolution,
        isLoading,
        setOriginalImage,
        setPixelatedCanvas,
        setPixelArtResolution,
        setLoading,
        resetImageState,
      }
    })
    ```

  - **任务 1.1.3**: 修改 `main.ts` 以初始化 Pinia。

    - 文件提议:

    ```typescript
    // src/main.ts
    import './assets/main.css'

    import { createApp } from 'vue'
    import { createPinia } from 'pinia' // Import createPinia
    import App from './App.vue'

    const app = createApp(App)
    app.use(createPinia()) // Use Pinia
    app.mount('#app')
    ```

  - **任务 1.1.4**: 重构 `App.vue` 和相关组件，使用 Pinia store 管理状态。
    - `ImageUploader` 调用 `imageStore.setOriginalImage`。
    - `ControlsPanel` 读取 `settingsStore` 的状态，并通过 `settingsStore.updateSettings` 更新。
    - `App.vue` 中的 `pixelateImage` 和 `processImage` 逻辑将读取 `settingsStore` 中的参数，更新 `imageStore` 中的画布和加载状态。
    - `DownloadButton` 和 `PixelEditor` 从 `imageStore` 获取 `pixelatedCanvas` 和 `pixelArtResolution`。

#### 1.2 Canvas 操作封装

- **当前问题**: `App.vue` 中包含大量的 Canvas 绘制和像素处理逻辑，这部分代码可以进一步抽象和封装。
- **解决方案**: 创建独立的 Canvas 工具函数或 Composition API。

  - **任务 1.2.1**: 封装 `applyPixelEffect`, `applyBrightnessContrast`, `applyPalette`, `findClosestColor`, `findClosestColorWithDithering` 等为独立的工具函数。

    - 文件提议:

    ```typescript
    // src/utils/canvasEffects.ts
    // Reusable functions for image processing on canvas
    interface RgbColor {
      r: number
      g: number
      b: number
    }

    export const findClosestColor = (
      r: number,
      g: number,
      b: number,
      palette: number[][],
    ): number[] => {
      /* ... existing logic ... */
    }

    export const findClosestColorWithDithering = (
      r: number,
      g: number,
      b: number,
      palette: number[][],
      pixelIndex: number,
      width: number,
    ): number[] => {
      /* ... existing logic ... */
    }

    export const applyBrightnessContrast = (
      ctx: CanvasRenderingContext2D,
      width: number,
      height: number,
      contrast: number,
      brightness: number,
    ) => {
      /* ... existing logic ... */
    }

    export const applyPalette = (
      ctx: CanvasRenderingContext2D,
      width: number,
      height: number,
      palette: number[][],
      enableDithering: boolean,
      preserveTransparency: boolean,
    ) => {
      /* ... existing logic ... */
    }

    export const pixelateAndApplyEffects = (
      img: HTMLImageElement,
      outputWidth: number,
      outputHeight: number,
      blocksX: number,
      blocksY: number,
      settings: {
        pixelSize: number
        selectedPalette: string
        enableDithering: boolean
        preserveTransparency: boolean
        contrast: number
        brightness: number
      },
      palettes: Record<string, number[][]>, // Pass palettes here
    ): HTMLCanvasElement => {
      // This function will create a new canvas, apply all effects, and return it.
      // It will encapsulate the logic currently in App.vue's processImage and applyPixelEffect.
      const tempCanvas = document.createElement('canvas')
      const tempCtx = tempCanvas.getContext('2d')!
      tempCanvas.width = blocksX
      tempCanvas.height = blocksY
      tempCtx.imageSmoothingEnabled = false
      tempCtx.drawImage(img, 0, 0, blocksX, blocksY)

      applyBrightnessContrast(tempCtx, blocksX, blocksY, settings.contrast, settings.brightness)

      if (settings.selectedPalette && palettes[settings.selectedPalette]) {
        applyPalette(
          tempCtx,
          blocksX,
          blocksY,
          palettes[settings.selectedPalette],
          settings.enableDithering,
          settings.preserveTransparency,
        )
      }

      const finalCanvas = document.createElement('canvas')
      const finalCtx = finalCanvas.getContext('2d')!
      finalCanvas.width = outputWidth
      finalCanvas.height = outputHeight
      finalCtx.imageSmoothingEnabled = false
      finalCtx.drawImage(tempCanvas, 0, 0, outputWidth, outputHeight)

      return finalCanvas
    }
    ```

  - **任务 1.2.2**: 在 `App.vue` 中导入并使用这些工具函数。

#### 1.3 调色板数据独立管理

- **当前问题**: 调色板数据硬编码在 `App.vue` 中。
- **解决方案**: 将调色板数据提取到单独的文件中。

  - **任务 1.3.1**: 创建 `src/data/palettes.ts` 文件。

    - 文件提议:

    ```typescript
    // src/data/palettes.ts
    export const palettes: Record<string, number[][]> = {
      gameboy: [
        /* ... colors ... */
      ],
      nes: [
        /* ... colors ... */
      ],
      // ... all other palettes
    }

    export type PaletteKey = keyof typeof palettes | ''
    ```

  - **任务 1.3.2**: 在 `App.vue` 和 `ControlsPanel.vue` 中导入使用。

#### 1.4 性能优化 (Web Workers)

- **当前问题**: 图片像素化处理是 CPU 密集型任务，可能导致主线程阻塞，尤其对于大图。
- **解决方案**: 将图片处理的核心逻辑放入 Web Worker 中执行，避免阻塞 UI。

  - **任务 1.4.1**: 创建 Web Worker 文件 (`src/workers/pixelatorWorker.ts`)。

    - 文件提议:

    ```typescript
    // src/workers/pixelatorWorker.ts
    import { pixelateAndApplyEffects } from '@/utils/canvasEffects' // Assume this is extracted
    import { palettes } from '@/data/palettes' // Assume this is extracted

    self.onmessage = async (event) => {
      const { imageDataUrl, settings, originalImageWidth, originalImageHeight } = event.data

      try {
        const img = await new Promise<HTMLImageElement>((resolve, reject) => {
          const image = new Image()
          image.crossOrigin = 'anonymous'
          image.onload = () => resolve(image)
          image.onerror = () => reject(new Error('Image load failed in worker.'))
          image.src = imageDataUrl
        })

        // Re-calculate output dimensions based on max width/height if necessary,
        // or pass them directly from main thread.
        const maxWidth = 600
        const maxHeight = 450
        let outputWidth = originalImageWidth
        let outputHeight = originalImageHeight
        const aspectRatio = originalImageWidth / originalImageHeight

        if (outputWidth > maxWidth) {
          outputWidth = maxWidth
          outputHeight = outputWidth / aspectRatio
        }
        if (outputHeight > maxHeight) {
          outputHeight = maxHeight
          outputWidth = outputHeight * aspectRatio
        }

        outputWidth = Math.round(outputWidth)
        outputHeight = Math.round(outputHeight)

        const blocksX = Math.max(1, Math.floor(outputWidth / settings.pixelSize))
        const blocksY = Math.max(1, Math.floor(outputHeight / settings.pixelSize))

        const processedCanvas = pixelateAndApplyEffects(
          img,
          outputWidth,
          outputHeight,
          blocksX,
          blocksY,
          settings,
          palettes,
        )

        // Transfer the canvas bitmap or its data URL back to the main thread
        const bitmap = processedCanvas.transferToImageBitmap()
        self.postMessage(
          { type: 'success', bitmap, pixelArtResolution: { width: blocksX, height: blocksY } },
          [bitmap],
        )
      } catch (error: any) {
        self.postMessage({ type: 'error', message: error.message || 'Unknown error' })
      }
    }
    ```

  - **任务 1.4.2**: 修改 `App.vue`，在 `pixelateImage` 函数中创建并使用 Web Worker。

    - 将图片转换为 `data URL` 传递给 Worker。
    - 接收 Worker 返回的 `ImageBitmap`，绘制到主画布上。
    - 文件提议 (App.vue 部分修改):

    ```typescript
    // src/App.vue (Partial change)
    <script setup lang="ts">
    // ... imports
    import { useImageStore } from './stores/image'
    import { useSettingsStore } from './stores/settings'
    // ... other imports

    const imageStore = useImageStore()
    const settingsStore = useSettingsStore()

    // Replace ref(null) for originalImageUrl with store's value
    // Replace ref(undefined) for pixelatedCanvas with store's value
    // Replace ref(false) for isLoading with store's value
    // Replace ref(16), ref(''), etc. for settings with store's values

    const pixelatedCanvasRef = ref<HTMLCanvasElement | null>(null) // Still needed for template ref

    // Create the worker instance outside functions or using a ref
    let pixelatorWorker: Worker | null = null;
    if (typeof Worker !== 'undefined') {
      pixelatorWorker = new Worker(new URL('./workers/pixelatorWorker', import.meta.url), { type: 'module' });
      pixelatorWorker.onmessage = (event) => {
        imageStore.setLoading(false);
        if (event.data.type === 'success') {
          const { bitmap, pixelArtResolution } = event.data;
          if (pixelatedCanvasRef.value) {
            const ctx = pixelatedCanvasRef.value.getContext('2d')!;
            pixelatedCanvasRef.value.width = bitmap.width;
            pixelatedCanvasRef.value.height = bitmap.height;
            ctx.imageSmoothingEnabled = false;
            ctx.drawImage(bitmap, 0, 0);
            imageStore.setPixelatedCanvas(pixelatedCanvasRef.value); // Update store with display canvas
            imageStore.setPixelArtResolution(pixelArtResolution); // Update store with logical resolution
          }
          bitmap.close(); // Release bitmap memory
        } else if (event.data.type === 'error') {
          console.error('Worker error:', event.data.message);
          alert(`处理失败：${event.data.message}`);
        }
      };
      pixelatorWorker.onerror = (error) => {
        imageStore.setLoading(false);
        console.error('Web Worker error:', error);
        alert('图片处理过程中发生未知错误，请重试。');
      };
    } else {
      console.warn('Web Workers are not supported in this browser. Image processing will run on the main thread.');
      // Fallback to main thread processing if workers are not supported
      // You might keep the existing processImage function as a fallback here.
    }

    const pixelateImage = async () => {
      if (!imageStore.originalImageUrl) {
        alert('请先上传图片');
        return;
      }
      if (imageStore.isLoading) return;

      imageStore.setLoading(true);

      try {
        const img = await loadImage(imageStore.originalImageUrl); // loadImage still helpful for initial load

        // Pass original image dimensions and settings to the worker
        if (pixelatorWorker) {
          pixelatorWorker.postMessage({
            imageDataUrl: imageStore.originalImageUrl, // Use data URL or transfer Blob if available
            settings: {
              pixelSize: settingsStore.pixelSize,
              selectedPalette: settingsStore.selectedPalette,
              enableDithering: settingsStore.enableDithering,
              preserveTransparency: settingsStore.preserveTransparency,
              contrast: settingsStore.contrast,
              brightness: settingsStore.brightness,
            },
            originalImageWidth: img.width,
            originalImageHeight: img.height,
          });
        } else {
          // Fallback to main thread processing
          // Your existing processImage function, adapted to use Pinia stores
          // await processImage(img);
        }
      } catch (error) {
        console.error('像素化处理错误:', error);
        imageStore.setLoading(false);
        alert(`处理失败，请重试。错误信息: ${error instanceof Error ? error.message : String(error)}`);
      }
    };

    // ... other functions like handleImageUpload, handleSettingsChange (now use store actions)
    // ... handleCanvasUpdate now receives the low-res canvas from PixelEditor and draws it to pixelatedCanvasRef
    // ... pixelatedCanvas.value will be updated from imageStore now
    </script>
    ```

#### 1.5 错误处理与用户反馈

- **当前问题**: 错误提示主要通过 `alert()`，不够友好。
- **解决方案**: 引入更友好的通知系统，并细化错误类型。
  - **任务 1.5.1**: 替换 `alert()` 为 Toast 通知或模态框。
  - **任务 1.5.2**: 捕获更多潜在错误（如 Canvas 上下文获取失败、图片格式不受支持等）。
  - **任务 1.5.3**: 优化加载状态的视觉反馈，例如在加载层中显示具体处理步骤（可选）。
