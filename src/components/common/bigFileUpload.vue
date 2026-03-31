<template>
  <div class="upload-container">
    <input type="file" @change="selectFile" ref="fileInput" />
    <button @click="startUpload" :disabled="!selectedFile">上传文件</button>

    <div v-if="uploading" class="progress-container">
      <div class="progress-bar" :style="{ width: progress + '%' }"></div>
      <span>{{ progress }}%</span>
      <button @click="pauseUpload">暂停</button>
      <button @click="cancelUpload">取消</button>
    </div>

    <div v-if="uploadedChunks.length > 0">
      <p>已上传分片: {{ uploadedChunks.length }}/{{ totalChunks }}</p>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted } from 'vue'
import axios from 'axios'

const fileInput = ref(null)
const selectedFile = ref(null)
const fileId = ref('')
const uploading = ref(false)
const paused = ref(false)
const progress = ref(0)
const chunkSize = ref(1024 * 1024) // 1MB
const uploadedChunks = ref([])
const totalChunks = ref(0)
const controller = ref(null)

// 选择文件
const selectFile = (e) => {
  if (e.target.files.length > 0) {
    selectedFile.value = e.target.files[0]
    fileId.value = generateFileId()
    console.log('文件ID:', fileId.value)
  }
}

// 生成唯一文件ID
const generateFileId = () => {
  return `${Date.now()}-${Math.random().toString(36).substring(2, 15)}`
}

// 开始上传
const startUpload = async () => {
  if (!selectedFile.value || uploading.value) return

  uploading.value = true
  paused.value = false
  controller.value = new AbortController()

  // 检查已上传的分片
  await checkExistingChunks()

  // 计算总片数
  totalChunks.value = Math.ceil(selectedFile.value.size / chunkSize.value)

  // 上传剩余分片
  await uploadChunks()
}

// 检查已上传的分片
const checkExistingChunks = async () => {
  try {
    const response = await axios.get(`/api/check-chunks/${fileId.value}`)
    uploadedChunks.value = response.data.existingChunks || []
    updateProgress()
  } catch (error) {
    console.error('检查分片失败:', error)
    uploadedChunks.value = []
  }
}

// 上传所有分片
const uploadChunks = async () => {
  const file = selectedFile.value
  let offset = 0
  let chunkIndex = 0

  while (offset < file.size && !paused.value) {
    // 跳过已上传的分片
    if (uploadedChunks.value.includes(chunkIndex)) {
      offset += chunkSize.value
      chunkIndex++
      continue
    }

    const chunk = file.slice(offset, offset + chunkSize.value)
    await uploadChunk(chunk, chunkIndex)

    offset += chunkSize.value
    chunkIndex++
  }

  // 所有分片上传完成后合并
  if (uploadedChunks.value.length === totalChunks.value && !paused.value) {
    await mergeChunks()
  }
}

// 上传单个分片
const uploadChunk = async (chunk, chunkIndex) => {
  const formData = new FormData()
  formData.append('fileId', fileId.value)
  formData.append('chunk', chunk)
  formData.append('chunkIndex', chunkIndex)
  formData.append('totalChunks', totalChunks.value)
  formData.append('fileName', selectedFile.value.name)

  try {
    await axios.post('/api/upload-chunk', formData, {
      signal: controller.value.signal,
      onUploadProgress: (progressEvent) => {
        // 计算当前分片的上传进度
        const percentCompleted = Math.round((progressEvent.loaded * 100) / progressEvent.total)
        // 更新整体进度（加权平均）
        const overallProgress = (uploadedChunks.value.length / totalChunks.value) * 100 + (1 / totalChunks.value) * percentCompleted
        progress.value = Math.min(100, Math.round(overallProgress))
      }
    })

    // 记录已上传的分片
    uploadedChunks.value.push(chunkIndex)
    updateProgress()
  } catch (error) {
    if (error.name !== 'AbortError') {
      console.error('上传分片失败:', error)
      // 可以添加重试逻辑
    }
  }
}

// 更新进度显示
const updateProgress = () => {
  if (totalChunks.value > 0) {
    progress.value = Math.round((uploadedChunks.value.length / totalChunks.value) * 100)
  }
}

// 合并分片
const mergeChunks = async () => {
  try {
    await axios.post('/api/merge-chunks', {
      fileId: fileId.value,
      fileName: selectedFile.value.name,
      totalChunks: totalChunks.value
    })

    alert('文件上传完成！')
    resetUpload()
  } catch (error) {
    console.error('合并分片失败:', error)
    alert('合并分片失败，请重试')
  }
}

// 暂停上传
const pauseUpload = () => {
  paused.value = true
  controller.value.abort()
}

// 取消上传
const cancelUpload = () => {
  paused.value = true
  controller.value.abort()
  resetUpload()
}

// 重置上传状态
const resetUpload = () => {
  uploading.value = false
  progress.value = 0
  uploadedChunks.value = []
  totalChunks.value = 0
  fileInput.value.value = ''
  selectedFile.value = null
}

// 组件卸载时清理
onUnmounted(() => {
  if (controller.value) {
    controller.value.abort()
  }
})
</script>

<style scoped>
.upload-container {
  max-width: 600px;
  margin: 20px auto;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 8px;
}

.progress-container {
  margin-top: 15px;
}

.progress-bar {
  height: 20px;
  background-color: #4caf50;
  transition: width 0.3s;
  margin-bottom: 10px;
}

button {
  margin: 5px;
  padding: 8px 15px;
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}
</style>
