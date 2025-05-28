<template>
  <div class="image-uploader">
    <div
      class="upload-area"
      :class="{ 'drag-over': isDragOver }"
      @click="triggerFileInput"
      @dragover.prevent="handleDragOver"
      @dragleave.prevent="handleDragLeave"
      @drop.prevent="handleDrop"
    >
      <input
        ref="fileInput"
        type="file"
        accept="image/jpeg,image/jpg,image/png,image/gif,image/webp,image/bmp,image/svg+xml"
        @change="handleFileSelect"
        class="file-input"
      />

      <div class="upload-content">
        <div class="upload-icon">📁</div>
        <h3 class="upload-title">上传图片</h3>
        <p class="upload-description">点击选择文件或拖拽图片到此区域</p>
        <p class="upload-formats">支持 JPG、PNG、GIF、WebP、BMP、SVG 格式</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

// 定义事件
const emit = defineEmits(['image-uploaded'])

// 响应式数据
const fileInput = ref<HTMLInputElement | null>(null)
const isDragOver = ref(false)

// 触发文件选择
const triggerFileInput = () => {
  fileInput.value?.click()
}

// 支持的图片格式
const supportedFormats = [
  'image/jpeg',
  'image/jpg',
  'image/png',
  'image/gif',
  'image/webp',
  'image/bmp',
  'image/svg+xml',
]

// 验证文件格式
const isValidImageFile = (file: File): boolean => {
  return supportedFormats.includes(file.type) || file.type.startsWith('image/')
}

// 处理文件选择
const handleFileSelect = (event: Event) => {
  const target = event.target as HTMLInputElement
  const file = target.files?.[0]
  if (file && isValidImageFile(file)) {
    // 检查文件大小（限制为10MB）
    if (file.size > 10 * 1024 * 1024) {
      alert('文件大小不能超过10MB')
      return
    }
    emit('image-uploaded', file)
  } else if (file) {
    alert('请选择有效的图片文件（支持 JPG、PNG、GIF、WebP、BMP、SVG 格式）')
  }
}

// 处理拖拽悬停
const handleDragOver = (event: DragEvent) => {
  event.preventDefault()
  isDragOver.value = true
}

// 处理拖拽离开
const handleDragLeave = (event: DragEvent) => {
  event.preventDefault()
  isDragOver.value = false
}

// 处理文件拖放
const handleDrop = (event: DragEvent) => {
  event.preventDefault()
  isDragOver.value = false

  const files = event.dataTransfer?.files
  if (files && files.length > 0) {
    const file = files[0]
    if (isValidImageFile(file)) {
      // 检查文件大小（限制为10MB）
      if (file.size > 10 * 1024 * 1024) {
        alert('文件大小不能超过10MB')
        return
      }
      emit('image-uploaded', file)
    } else {
      alert('请选择有效的图片文件（支持 JPG、PNG、GIF、WebP、BMP、SVG 格式）')
    }
  }
}
</script>

<style scoped>
.image-uploader {
  width: 100%;
}

.upload-area {
  border: 2px dashed #e2e8f0;
  border-radius: 16px;
  padding: 48px 24px;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s ease;
  background: linear-gradient(135deg, #f8fafc 0%, #f1f5f9 100%);
  position: relative;
  overflow: hidden;
  box-shadow: 0 4px 16px rgba(102, 126, 234, 0.05);
}

.upload-area:hover {
  border-color: #667eea;
  background: linear-gradient(135deg, #f0f8ff 0%, #e6f3ff 100%);
  transform: translateY(-3px);
  box-shadow: 0 12px 32px rgba(102, 126, 234, 0.15);
}

.upload-area.drag-over {
  border-color: #667eea;
  background: linear-gradient(135deg, #e6f3ff 0%, #dbeafe 100%);
  transform: scale(1.02);
  box-shadow: 0 16px 40px rgba(102, 126, 234, 0.25);
  border-style: solid;
}

.file-input {
  display: none;
}

.upload-content {
  position: relative;
  z-index: 1;
}

.upload-icon {
  font-size: 56px;
  margin-bottom: 20px;
  display: block;
  opacity: 0.8;
  transition: all 0.3s ease;
  filter: drop-shadow(0 2px 4px rgba(102, 126, 234, 0.1));
}

.upload-area:hover .upload-icon {
  opacity: 1;
  transform: scale(1.15) rotate(5deg);
  filter: drop-shadow(0 4px 8px rgba(102, 126, 234, 0.2));
}

.upload-title {
  margin: 0 0 16px 0;
  color: #475569;
  font-size: 22px;
  font-weight: 700;
  letter-spacing: 0.5px;
}

.upload-description {
  margin: 0 0 12px 0;
  color: #64748b;
  font-size: 15px;
  line-height: 1.6;
  font-weight: 500;
}

.upload-formats {
  margin: 0;
  color: #94a3b8;
  font-size: 13px;
  font-weight: 500;
  opacity: 0.9;
}

/* 添加一些视觉效果 */
.upload-area::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(45deg, transparent 30%, rgba(0, 122, 204, 0.05) 50%, transparent 70%);
  opacity: 0;
  transition: opacity 0.3s ease;
}

.upload-area:hover::before {
  opacity: 1;
}

/* 响应式设计 */
@media (max-width: 480px) {
  .upload-area {
    padding: 30px 15px;
  }

  .upload-icon {
    font-size: 36px;
    margin-bottom: 12px;
  }

  .upload-title {
    font-size: 18px;
  }

  .upload-description {
    font-size: 13px;
  }
}
</style>
