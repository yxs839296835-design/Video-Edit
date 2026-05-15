<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from 'vue'
import { videoEditLocale } from '../locales/video-edit'

type UploadCard = {
  icon: string
  title: string
  description: string
}

type ToolCategoryId = 'hot' | 'edit' | 'format' | 'audio' | 'publish' | 'ai'

type EditorTool = {
  id: string
  category: Exclude<ToolCategoryId, 'hot'>
  icon: string
  title: string
  description: string
  badge?: string
  featured?: boolean
  external?: boolean
  externalUrl?: string
}

type ExportSetting = {
  id: string
  label: string
  options: string[]
  hint?: string
}

type MergeFileItem = {
  id: string
  file: File
  url: string
}

defineProps<{
  uploadCards: UploadCard[]
}>()

const emit = defineEmits<{
  workspaceChange: [active: boolean]
}>()

const t = videoEditLocale.zh
const DB_NAME = 'video-edit-browser-store'
const STORE_NAME = 'files'
const FILE_KEY = 'last-uploaded-video'

const fileInputRef = ref<HTMLInputElement>()
const replaceInputRef = ref<HTMLInputElement>()
const mergeInputRef = ref<HTMLInputElement>()
const mediaRef = ref<HTMLVideoElement | HTMLAudioElement>()
const cropMediaRef = ref<HTMLVideoElement>()
const cropStageRef = ref<HTMLElement>()
const trimMediaRef = ref<HTMLVideoElement>()
const trimTrackRef = ref<HTMLElement>()
const uploadedFile = ref<File | null>(null)
const mergeFiles = ref<MergeFileItem[]>([])
const objectUrl = ref('')
const isDragging = ref(false)
const isPlaying = ref(false)
const isCropPlaying = ref(false)
const isTrimPlaying = ref(false)
const currentTime = ref(0)
const cropCurrentTime = ref(0)
const trimCurrentTime = ref(0)
const duration = ref(0)
const cropDuration = ref(0)
const trimDuration = ref(0)
const activeTool = ref('compress')
const outputRatio = ref('Original')
const outputQuality = ref('1080p')
const videoWidth = ref(0)
const videoHeight = ref(0)
const compressionSizeMode = ref<'percent' | 'size'>('percent')
const compressionPercent = ref(30)
const compressionTargetSizeMb = ref(0)
const convertResolution = ref('source')
const isResolutionDropdownOpen = ref(false)
const gifSeconds = ref(10)
const gifRangeMode = ref<'first' | 'custom'>('first')
const gifFrameRate = ref('10')
const gifSize = ref('原始')
const gifSpeed = ref('原始')
const cropRatio = ref('original')
const cropBox = ref({ x: 20, y: 14, width: 60, height: 72 })
const cropDragStart = ref<{
  clientX: number
  clientY: number
  x: number
  y: number
} | null>(null)
const isTrimHandleDragging = ref(false)
const selectedSettingOptions = ref<Record<string, string>>({})
const isProcessing = ref(false)

const editorTools: EditorTool[] = [
  {
    id: 'compress',
    category: 'format',
    icon: '↘',
    title: '视频压缩',
    description: '压缩视频体积，便于上传和分享',
  },
  {
    id: 'convert',
    category: 'format',
    icon: '⇄',
    title: '格式转换',
    description: '转换常见视频格式',
  },
  {
    id: 'gif-maker',
    category: 'format',
    icon: 'GIF',
    title: 'Gif制作',
    description: '截取视频片段制作 GIF',
  },
  {
    id: 'mp3-converter',
    category: 'audio',
    icon: 'MP3',
    title: 'mp3 converter',
    description: '将音频文件转换为 MP3',
  },
  {
    id: 'mp4-to-mp3',
    category: 'audio',
    icon: '♪',
    title: 'mp4 to mp3',
    description: '从 MP4 视频中提取 MP3',
  },
  {
    id: 'video-to-gif',
    category: 'format',
    icon: 'GIF',
    title: 'video to gif',
    description: '将视频片段转换为 GIF',
  },
  {
    id: 'merge',
    category: 'edit',
    icon: '+',
    title: '视频合并',
    description: '合并多个视频素材',
  },
  {
    id: 'trim',
    category: 'edit',
    icon: '✂',
    title: '视频剪切',
    description: '剪掉片头片尾和多余片段',
  },
  {
    id: 'crop',
    category: 'edit',
    icon: '▣',
    title: '视频裁剪',
    description: '裁剪画面范围和比例',
  },
  {
    id: 'ai-subtitle',
    category: 'ai',
    icon: 'AI',
    title: 'AI字幕',
    description: '自动识别语音并生成字幕',
    badge: 'AI',
    external: true,
    externalUrl: 'https://reccloud.cn/ai-subtitle?v=product',
  },
  {
    id: 'ai-video-translate',
    category: 'ai',
    icon: '译',
    title: 'AI视频翻译',
    description: '翻译视频语音和字幕内容',
    badge: 'AI',
    external: true,
    externalUrl: 'https://reccloud.cn/video-translator',
  },
  {
    id: 'ai-speech-to-text',
    category: 'ai',
    icon: 'TXT',
    title: 'AI语音转文字',
    description: '将视频或音频转成文字稿',
    badge: 'AI',
    external: true,
    externalUrl: 'https://reccloud.cn/speech-to-text-online',
  },
  {
    id: 'ai-remove-watermark',
    category: 'ai',
    icon: 'WM',
    title: 'AI视频去水印',
    description: '智能移除视频水印和遮挡',
    badge: 'AI',
    external: true,
    externalUrl: 'https://reccloud.cn/remove-watermark-from-video',
  },
  {
    id: 'ai-video-generate',
    category: 'ai',
    icon: '生',
    title: 'AI视频生成',
    description: '用文字或素材生成视频内容',
    badge: 'AI',
    external: true,
    externalUrl: 'https://reccloud.cn/text-to-video',
  },
]

const fileSize = computed(() => {
  if (!uploadedFile.value) return ''
  const size = uploadedFile.value.size
  if (size >= 1024 * 1024 * 1024) return `${(size / 1024 / 1024 / 1024).toFixed(2)} GB`
  return `${(size / 1024 / 1024).toFixed(2)} MB`
})

const fileType = computed(() => uploadedFile.value?.type || '')
const isAudio = computed(() => fileType.value.startsWith('audio/'))
const durationLabel = computed(() => formatTime(duration.value))
const currentTimeLabel = computed(() => formatTime(currentTime.value))
const cropDurationLabel = computed(() => formatTime(cropDuration.value || duration.value))
const cropCurrentTimeLabel = computed(() => formatTime(cropCurrentTime.value))
const trimDurationLabel = computed(() => formatTime(trimDuration.value || duration.value))
const trimCurrentTimeLabel = computed(() => formatTime(trimCurrentTime.value))
const cropProgressPercent = computed(() => {
  const total = cropDuration.value || duration.value
  if (!total) return 0
  return Math.min(100, Math.max(0, (cropCurrentTime.value / total) * 100))
})
const trimProgressPercent = computed(() => {
  const total = trimDuration.value || duration.value
  if (!total) return 0
  return Math.min(100, Math.max(0, (trimCurrentTime.value / total) * 100))
})
const cropBoxStyle = computed(() => ({
  left: `${cropBox.value.x}%`,
  top: `${cropBox.value.y}%`,
  width: `${cropBox.value.width}%`,
  height: `${cropBox.value.height}%`,
}))
const sourceResolutionLabel = computed(() => {
  if (!videoWidth.value || !videoHeight.value) return '同源文件分辨率'
  return `同源文件分辨率（${videoWidth.value}*${videoHeight.value}）`
})
const convertResolutionOptions = computed(() => [
  { value: 'source', label: sourceResolutionLabel.value },
  { value: '1080', label: '1080（1920*1080）' },
  { value: '720', label: '720（1280*720）' },
  { value: '640', label: '640（960*640）' },
  { value: '576', label: '576（720*576）' },
])
const selectedConvertResolutionLabel = computed(() => {
  return convertResolutionOptions.value.find((option) => option.value === convertResolution.value)?.label || sourceResolutionLabel.value
})
const mergeFileItems = computed(() => {
  return mergeFiles.value
})
const progressPercent = computed(() => {
  if (!duration.value) return 0
  return Math.min(100, Math.max(0, (currentTime.value / duration.value) * 100))
})

const visibleTools = computed(() => {
  return editorTools
})

const activeToolItem = computed(() => {
  return editorTools.find((tool) => tool.id === activeTool.value) || editorTools[0]
})

const activeExportSettings = computed<ExportSetting[]>(() => {
  switch (activeTool.value) {
    case 'compress':
      return [
        { id: 'compressMode', label: '压缩模式', options: ['智能压缩', '优先清晰', '最小体积'], hint: '适合减小文件体积并保持画面观感' },
      ]
    case 'convert':
      return [
        { id: 'format', label: '目标格式', options: ['MP4', 'AVI', 'MOV', 'MPEG', 'WMV', '3GP', 'ASF', 'MTS', 'MKV', 'FLV', 'M4V', 'XVID', 'VOB', 'DIVX', 'MXF'] },
        { id: 'codec', label: '编码方式', options: ['同源编码', 'H.264', 'H.265', 'MPEG-4', 'VP9'] },
      ]
    case 'trim':
      return [
        { id: 'range', label: '剪辑范围', options: ['保留选区', '删除选区', '自动去片头片尾'] },
        { id: 'joinMode', label: '片段处理', options: ['单段导出', '多段合并', '批量导出'] },
      ]
    case 'crop':
      return [
        { id: 'ratio', label: '画面比例', options: ['Original', '16:9', '9:16', '1:1'] },
        { id: 'fit', label: '适配方式', options: ['裁剪填满', '完整留边', '智能居中'] },
        { id: 'platform', label: '发布平台', options: ['通用', '抖音/视频号', '小红书', 'YouTube'] },
      ]
    case 'gif-maker':
    case 'video-to-gif':
      return []
    case 'merge':
      return [
        { id: 'mergeOrder', label: '合并顺序', options: ['当前顺序', '按文件名', '按上传时间'] },
        { id: 'transition', label: '转场', options: ['无转场', '淡入淡出', '快速切换'] },
        { id: 'format', label: '导出格式', options: ['MP4', 'MOV', 'WEBM'] },
      ]
    case 'mp3-converter':
      return [
        { id: 'audioFormat', label: '导出格式', options: ['MP3', 'M4A', 'WAV', 'AAC'] },
        { id: 'audioQuality', label: '音频质量', options: ['标准', '高质量', '无损'] },
      ]
    case 'mp4-to-mp3':
      return [
        { id: 'audioFormat', label: '导出格式', options: ['MP3', 'M4A', 'WAV', 'AAC'] },
        { id: 'audioQuality', label: '音频质量', options: ['标准', '高质量', '无损'] },
      ]
    case 'ai-subtitle':
    case 'ai-video-translate':
    case 'ai-speech-to-text':
    case 'ai-remove-watermark':
    case 'ai-video-generate':
      return [
        { id: 'aiMode', label: 'AI 处理方式', options: ['快速体验', '高清处理', '批量任务'], hint: 'AI 功能可跳转到独立处理页或桌面端任务流程' },
        { id: 'delivery', label: '完成后', options: ['当前页预览', '跳转详情页', '保存到任务列表'] },
      ]
    default:
      return [
        { id: 'preset', label: '处理预设', options: ['智能推荐', '保持原质量', '适合发布'] },
        { id: 'format', label: '导出格式', options: ['MP4', 'MOV', 'WEBM'] },
      ]
  }
})

onMounted(() => {
  restoreStoredFile().catch(() => undefined)
})

onBeforeUnmount(() => {
  window.removeEventListener('pointermove', handleCropDragMove)
  window.removeEventListener('pointerup', stopCropDrag)
  window.removeEventListener('pointermove', handleTrimHandleMove)
  window.removeEventListener('pointerup', stopTrimHandleDrag)
  clearMergeFiles()
  revokeObjectUrl()
})

function openFilePicker() {
  fileInputRef.value?.click()
}

function openReplacePicker() {
  replaceInputRef.value?.click()
}

function openMergePicker() {
  mergeInputRef.value?.click()
}

async function handleFileInput(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file) return
  await setUploadedFile(file)
  input.value = ''
}

function handleMergeFileInput(event: Event) {
  const input = event.target as HTMLInputElement
  const files = Array.from(input.files || [])
  if (!files.length) return
  mergeFiles.value = [...mergeFiles.value, ...files.map(createMergeFileItem)]
  input.value = ''
}

async function handleDrop(event: DragEvent) {
  isDragging.value = false
  const file = event.dataTransfer?.files?.[0]
  if (!file) return
  await setUploadedFile(file)
}

async function setUploadedFile(file: File) {
  revokeObjectUrl()
  clearMergeFiles()
  uploadedFile.value = file
  objectUrl.value = URL.createObjectURL(file)
  mergeFiles.value = [createMergeFileItem(file)]
  currentTime.value = 0
  duration.value = 0
  cropCurrentTime.value = 0
  cropDuration.value = 0
  trimCurrentTime.value = 0
  trimDuration.value = 0
  videoWidth.value = 0
  videoHeight.value = 0
  compressionTargetSizeMb.value = getDefaultCompressionSizeMb(file)
  isPlaying.value = false
  isCropPlaying.value = false
  isTrimPlaying.value = false
  await saveFileToBrowser(file)
  emit('workspaceChange', true)
  await nextTick()
  window.scrollTo({ top: 0, left: 0, behavior: 'instant' })
  mediaRef.value?.load()
}

function revokeObjectUrl() {
  if (objectUrl.value) URL.revokeObjectURL(objectUrl.value)
  objectUrl.value = ''
}

function createMergeFileItem(file: File): MergeFileItem {
  return {
    id: `${file.name}-${file.size}-${file.lastModified}-${crypto.randomUUID()}`,
    file,
    url: URL.createObjectURL(file),
  }
}

function clearMergeFiles() {
  mergeFiles.value.forEach((item) => URL.revokeObjectURL(item.url))
  mergeFiles.value = []
}

function formatTime(seconds: number) {
  if (!Number.isFinite(seconds) || seconds < 0) return '00:00'
  const total = Math.floor(seconds)
  const minutes = Math.floor(total / 60)
  const remain = total % 60
  return `${String(minutes).padStart(2, '0')}:${String(remain).padStart(2, '0')}`
}

function getDefaultCompressionSizeMb(file: File) {
  return Math.max(1, Math.round((file.size / 1024 / 1024) * 0.3))
}

function formatFileSizeValue(file: File) {
  if (file.size >= 1024 * 1024 * 1024) return `${(file.size / 1024 / 1024 / 1024).toFixed(2)} GB`
  return `${(file.size / 1024 / 1024).toFixed(2)} MB`
}

function handleLoadedMetadata() {
  const media = mediaRef.value
  duration.value = media?.duration || 0
  if (media instanceof HTMLVideoElement) {
    videoWidth.value = media.videoWidth || 0
    videoHeight.value = media.videoHeight || 0
  }
}

function handleCropLoadedMetadata() {
  const media = cropMediaRef.value
  cropDuration.value = media?.duration || 0
  if (media) {
    videoWidth.value = media.videoWidth || videoWidth.value
    videoHeight.value = media.videoHeight || videoHeight.value
  }
}

function handleTrimLoadedMetadata() {
  const media = trimMediaRef.value
  trimDuration.value = media?.duration || 0
  if (media) {
    videoWidth.value = media.videoWidth || videoWidth.value
    videoHeight.value = media.videoHeight || videoHeight.value
  }
}

function handleTimeUpdate() {
  currentTime.value = mediaRef.value?.currentTime || 0
}

function handleCropTimeUpdate() {
  cropCurrentTime.value = cropMediaRef.value?.currentTime || 0
}

function handleTrimTimeUpdate() {
  trimCurrentTime.value = trimMediaRef.value?.currentTime || 0
}

async function togglePlay() {
  const media = mediaRef.value
  if (!media) return
  if (media.paused) {
    await media.play()
    isPlaying.value = true
  } else {
    media.pause()
    isPlaying.value = false
  }
}

function handleEnded() {
  isPlaying.value = false
}

async function toggleCropPlay() {
  const media = cropMediaRef.value
  if (!media) return
  if (media.paused) {
    await media.play()
    isCropPlaying.value = true
  } else {
    media.pause()
    isCropPlaying.value = false
  }
}

function handleCropEnded() {
  isCropPlaying.value = false
}

async function toggleTrimPlay() {
  const media = trimMediaRef.value
  if (!media) return
  if (media.paused) {
    await media.play()
    isTrimPlaying.value = true
  } else {
    media.pause()
    isTrimPlaying.value = false
  }
}

function handleTrimEnded() {
  isTrimPlaying.value = false
}

function seekCropFromBar(event: MouseEvent) {
  const media = cropMediaRef.value
  const total = cropDuration.value || duration.value
  if (!media || !total) return
  const rect = (event.currentTarget as HTMLElement).getBoundingClientRect()
  const ratio = (event.clientX - rect.left) / rect.width
  media.currentTime = Math.min(total, Math.max(0, ratio * total))
}

function seekTrimFromTrack(event: MouseEvent | PointerEvent) {
  const media = trimMediaRef.value
  const track = trimTrackRef.value
  const total = trimDuration.value || duration.value
  if (!media || !track || !total) return
  const rect = track.getBoundingClientRect()
  const ratio = Math.min(1, Math.max(0, (event.clientX - rect.left) / rect.width))
  media.currentTime = total * ratio
  trimCurrentTime.value = media.currentTime
}

function startTrimHandleDrag(event: PointerEvent) {
  isTrimHandleDragging.value = true
  seekTrimFromTrack(event)
  window.addEventListener('pointermove', handleTrimHandleMove)
  window.addEventListener('pointerup', stopTrimHandleDrag)
}

function handleTrimHandleMove(event: PointerEvent) {
  if (!isTrimHandleDragging.value) return
  seekTrimFromTrack(event)
}

function stopTrimHandleDrag() {
  isTrimHandleDragging.value = false
  window.removeEventListener('pointermove', handleTrimHandleMove)
  window.removeEventListener('pointerup', stopTrimHandleDrag)
}

function splitTrimAtCurrentTime() {
  selectedSettingOptions.value = {
    ...selectedSettingOptions.value,
    trimAction: `split:${trimCurrentTime.value.toFixed(2)}`,
  }
}

function deleteTrimSegment() {
  selectedSettingOptions.value = {
    ...selectedSettingOptions.value,
    trimAction: `delete:${trimCurrentTime.value.toFixed(2)}`,
  }
}

function seekFromBar(event: MouseEvent) {
  const media = mediaRef.value
  if (!media || !duration.value) return
  const rect = (event.currentTarget as HTMLElement).getBoundingClientRect()
  const ratio = (event.clientX - rect.left) / rect.width
  media.currentTime = Math.min(duration.value, Math.max(0, ratio * duration.value))
}

function downloadOriginal() {
  if (!uploadedFile.value || !objectUrl.value) return
  const link = document.createElement('a')
  link.href = objectUrl.value
  link.download = uploadedFile.value.name
  link.click()
}

function removeMergeFile(index: number) {
  if (mergeFiles.value.length <= 1) return
  const item = mergeFiles.value[index]
  if (item) URL.revokeObjectURL(item.url)
  mergeFiles.value = mergeFiles.value.filter((_, itemIndex) => itemIndex !== index)
}

function selectTool(tool: EditorTool) {
  if (tool.external) {
    if (tool.externalUrl) window.location.href = tool.externalUrl
    return
  }
  activeTool.value = tool.id
  isResolutionDropdownOpen.value = false
}

function selectedOption(setting: ExportSetting) {
  return selectedSettingOptions.value[setting.id] || setting.options[0]
}

function selectSettingOption(setting: ExportSetting, option: string) {
  selectedSettingOptions.value = {
    ...selectedSettingOptions.value,
    [setting.id]: option,
  }
}

function toggleResolutionDropdown() {
  isResolutionDropdownOpen.value = !isResolutionDropdownOpen.value
}

function selectConvertResolution(value: string) {
  convertResolution.value = value
  isResolutionDropdownOpen.value = false
}

function setCropRatio(value: string) {
  cropRatio.value = value
  if (value === 'free') return

  const stage = cropStageRef.value
  const rect = stage?.getBoundingClientRect()
  const stageAspect = rect?.width && rect.height ? rect.width / rect.height : 16 / 9
  const ratioMap: Record<string, number> = {
    wide: 16 / 9,
    square: 1,
    vertical: 9 / 16,
    classic: 4 / 3,
    original: videoWidth.value && videoHeight.value ? videoWidth.value / videoHeight.value : 16 / 9,
  }
  const targetRatio = ratioMap[value] || ratioMap.original
  let width = 72
  let height = (width / targetRatio) * stageAspect
  if (height > 78) {
    height = 78
    width = (height * targetRatio) / stageAspect
  }
  cropBox.value = {
    x: (100 - width) / 2,
    y: (100 - height) / 2,
    width,
    height,
  }
}

function startCropDrag(event: PointerEvent) {
  cropDragStart.value = {
    clientX: event.clientX,
    clientY: event.clientY,
    x: cropBox.value.x,
    y: cropBox.value.y,
  }
  window.addEventListener('pointermove', handleCropDragMove)
  window.addEventListener('pointerup', stopCropDrag)
}

function handleCropDragMove(event: PointerEvent) {
  const start = cropDragStart.value
  const stage = cropStageRef.value
  if (!start || !stage) return
  const rect = stage.getBoundingClientRect()
  const dx = ((event.clientX - start.clientX) / rect.width) * 100
  const dy = ((event.clientY - start.clientY) / rect.height) * 100
  cropBox.value = {
    ...cropBox.value,
    x: Math.min(100 - cropBox.value.width, Math.max(0, start.x + dx)),
    y: Math.min(100 - cropBox.value.height, Math.max(0, start.y + dy)),
  }
}

function stopCropDrag() {
  cropDragStart.value = null
  window.removeEventListener('pointermove', handleCropDragMove)
  window.removeEventListener('pointerup', stopCropDrag)
}

async function resetEditor() {
  uploadedFile.value = null
  currentTime.value = 0
  duration.value = 0
  isPlaying.value = false
  clearMergeFiles()
  revokeObjectUrl()
  await deleteStoredFile()
  emit('workspaceChange', false)
}

function startProcess() {
  isProcessing.value = true
  window.setTimeout(() => {
    isProcessing.value = false
  }, 900)
}

function openDb() {
  return new Promise<IDBDatabase>((resolve, reject) => {
    const request = indexedDB.open(DB_NAME, 1)
    request.onupgradeneeded = () => {
      const db = request.result
      if (!db.objectStoreNames.contains(STORE_NAME)) db.createObjectStore(STORE_NAME)
    }
    request.onsuccess = () => resolve(request.result)
    request.onerror = () => reject(request.error)
  })
}

async function saveFileToBrowser(file: File) {
  const db = await openDb()
  await new Promise<void>((resolve, reject) => {
    const transaction = db.transaction(STORE_NAME, 'readwrite')
    transaction.objectStore(STORE_NAME).put(file, FILE_KEY)
    transaction.oncomplete = () => resolve()
    transaction.onerror = () => reject(transaction.error)
  })
  db.close()
}

async function restoreStoredFile() {
  const db = await openDb()
  const file = await new Promise<File | undefined>((resolve, reject) => {
    const transaction = db.transaction(STORE_NAME, 'readonly')
    const request = transaction.objectStore(STORE_NAME).get(FILE_KEY)
    request.onsuccess = () => resolve(request.result as File | undefined)
    request.onerror = () => reject(request.error)
  })
  db.close()
  if (file) await setUploadedFile(file)
}

async function deleteStoredFile() {
  const db = await openDb()
  await new Promise<void>((resolve, reject) => {
    const transaction = db.transaction(STORE_NAME, 'readwrite')
    transaction.objectStore(STORE_NAME).delete(FILE_KEY)
    transaction.oncomplete = () => resolve()
    transaction.onerror = () => reject(transaction.error)
  })
  db.close()
}
</script>

<template>
  <section class="editor-shell" :class="{ 'workspace-mode': uploadedFile }" aria-label="Video editor uploader">
    <template v-if="!uploadedFile">
      <div
        class="upload-panel"
        :class="{ dragging: isDragging }"
        @dragenter.prevent="isDragging = true"
        @dragover.prevent="isDragging = true"
        @dragleave.prevent="isDragging = false"
        @drop.prevent="handleDrop"
      >
        <div class="upload-icon">
          <span />
        </div>
        <h2>{{ t.uploadTitle }}</h2>
        <p>{{ t.uploadDescription }}</p>
        <div class="upload-actions">
          <button class="gradient-button" type="button" @click="openFilePicker">
            {{ t.uploadButton }}
          </button>
          <button class="secondary-button" type="button" @click="openFilePicker">{{ t.cloudButton }}</button>
        </div>
        <p class="format-text">{{ t.supportedFormats }}</p>
      </div>

      <ul class="upload-card-list">
        <li v-for="item in uploadCards" :key="item.title">
          <img :src="item.icon" :alt="item.title" draggable="false">
          <div>
            <h3>{{ item.title }}</h3>
            <p>{{ item.description }}</p>
          </div>
        </li>
      </ul>
    </template>

    <template v-else>
      <div class="workspace-grid">
        <section class="library-panel" aria-label="Video tools">
          <div class="file-summary-row">
            <div class="thumb-window">
              <video
                v-if="!isAudio"
                ref="mediaRef"
                class="thumb-video"
                :src="objectUrl"
                muted
                playsinline
                preload="metadata"
                @loadedmetadata="handleLoadedMetadata"
                @ended="handleEnded"
              />
              <div v-else class="audio-thumb">
                <span>♪</span>
                <audio
                  ref="mediaRef"
                  :src="objectUrl"
                  preload="metadata"
                  @loadedmetadata="handleLoadedMetadata"
                  @ended="handleEnded"
                />
              </div>
            </div>

            <div class="file-meta">
              <span>当前文件</span>
              <h2>{{ uploadedFile.name }}</h2>
              <p>{{ fileSize }} · {{ uploadedFile.type || 'media file' }} · {{ durationLabel }}</p>
            </div>

            <div class="file-row-actions">
              <button class="pill-button" type="button" @click="openReplacePicker">替换文件</button>
              <button type="button" class="back-button" @click="resetEditor">返回上传</button>
            </div>
          </div>

          <div class="library-scroll">
            <section class="tool-section">
              <div class="tool-board">
                <button
                  v-for="tool in visibleTools"
                  :key="tool.id"
                  type="button"
                  class="tool-card"
                  :class="{ active: !tool.external && activeTool === tool.id, external: tool.external }"
                  @click="selectTool(tool)"
                >
                  <span class="tool-icon">{{ tool.icon }}</span>
                  <span class="tool-copy">
                    <strong>{{ tool.title }}</strong>
                    <small>{{ tool.description }}</small>
                  </span>
                  <em v-if="tool.badge">{{ tool.badge }}</em>
                </button>
              </div>
            </section>

            <div class="windows-download-card">
              <div>
                <strong>下载 Windows 桌面端</strong>
                <small><b>基础功能 Windows 端全部永久免费</b>，支持批量处理，速度更快、更稳定，可处理超大文件。</small>
              </div>
              <a class="download-button" href="https://download.aoscdn.com/down.php?softid=reccloud">立即下载</a>
            </div>
          </div>
        </section>

        <aside class="export-panel" aria-label="Video export settings">
          <div class="export-scroll">
            <section class="export-settings">
              <h3>导出设置</h3>

              <template v-if="activeTool === 'compress'">
                <div
                  v-for="setting in activeExportSettings"
                  :key="setting.id"
                  class="compact-control"
                >
                  <span>{{ setting.label }}</span>
                  <div class="segmented-row">
                    <button
                      v-for="option in setting.options"
                      :key="option"
                      type="button"
                      :class="{ active: selectedOption(setting) === option }"
                      @click="selectSettingOption(setting, option)"
                    >
                      {{ option }}
                    </button>
                  </div>
                  <small v-if="setting.hint">{{ setting.hint }}</small>
                </div>

                <div class="compact-control">
                  <span>导出方式</span>
                  <div class="segmented-row two-column">
                    <button
                      type="button"
                      :class="{ active: compressionSizeMode === 'percent' }"
                      @click="compressionSizeMode = 'percent'"
                    >
                      导出体积
                    </button>
                    <button
                      type="button"
                      :class="{ active: compressionSizeMode === 'size' }"
                      @click="compressionSizeMode = 'size'"
                    >
                      导出大小
                    </button>
                  </div>

                  <div v-if="compressionSizeMode === 'percent'" class="size-control">
                    <div class="range-head">
                      <strong>{{ compressionPercent }}%</strong>
                      <span>按当前百分比压缩视频体积</span>
                    </div>
                    <input v-model.number="compressionPercent" type="range" min="1" max="100" step="1">
                  </div>

                  <label v-else class="number-control">
                    <span>目标大小</span>
                    <input v-model.number="compressionTargetSizeMb" type="number" min="1" step="1">
                    <strong>MB</strong>
                  </label>
                  <small v-if="compressionSizeMode === 'size'">默认按当前文件 30% 取整数，可自行修改。</small>
                </div>
              </template>

              <template v-else-if="activeTool === 'convert'">
                <div
                  v-for="setting in activeExportSettings"
                  :key="setting.id"
                  class="compact-control"
                >
                  <span>{{ setting.label }}</span>
                  <div class="segmented-row format-grid">
                    <button
                      v-for="option in setting.options"
                      :key="option"
                      type="button"
                      :class="{ active: selectedOption(setting) === option }"
                      @click="selectSettingOption(setting, option)"
                    >
                      {{ option }}
                    </button>
                  </div>
                </div>

                <div class="compact-control select-control">
                  <span>导出质量</span>
                  <div class="custom-select" :class="{ open: isResolutionDropdownOpen }">
                    <button class="select-trigger" type="button" @click="toggleResolutionDropdown">
                      <span>{{ selectedConvertResolutionLabel }}</span>
                      <i>⌄</i>
                    </button>

                    <div v-if="isResolutionDropdownOpen" class="select-menu">
                      <div class="select-options">
                        <button
                          v-for="option in convertResolutionOptions"
                          :key="option.value"
                          type="button"
                          :class="{ active: convertResolution === option.value }"
                          @click="selectConvertResolution(option.value)"
                        >
                          {{ option.label }}
                        </button>
                      </div>
                    </div>
                  </div>
                </div>
              </template>

              <template v-else-if="activeTool === 'gif-maker' || activeTool === 'video-to-gif'">
                <div class="compact-control">
                  <span>GIF片段</span>
                  <div class="gif-range-control">
                    <button
                      type="button"
                      :class="{ active: gifRangeMode === 'first' }"
                      @click="gifRangeMode = 'first'"
                    >
                      前
                      <input
                        v-model.number="gifSeconds"
                        type="number"
                        min="1"
                        step="1"
                        @click.stop
                      >
                      秒
                    </button>
                    <a
                      class="client-link-button"
                      href="https://download.aoscdn.com/down.php?softid=reccloud"
                      :class="{ active: gifRangeMode === 'custom' }"
                      @click="gifRangeMode = 'custom'"
                    >
                      自定义
                    </a>
                  </div>
                  <small>自定义片段需要下载 Windows 桌面端处理。</small>
                </div>

                <div class="compact-control">
                  <span>帧率</span>
                  <div class="segmented-row">
                    <button
                      v-for="rate in ['8', '10', '15', '24']"
                      :key="rate"
                      type="button"
                      :class="{ active: gifFrameRate === rate }"
                      @click="gifFrameRate = rate"
                    >
                      {{ rate }}
                    </button>
                  </div>
                </div>

                <div class="compact-control">
                  <span>尺寸</span>
                  <div class="segmented-row">
                    <button
                      v-for="size in ['原始', '400', '280', '180']"
                      :key="size"
                      type="button"
                      :class="{ active: gifSize === size }"
                      @click="gifSize = size"
                    >
                      {{ size }}
                    </button>
                  </div>
                </div>

                <div class="compact-control">
                  <span>播放速度</span>
                  <div class="segmented-row">
                    <button
                      v-for="speed in ['原始', '0.5倍', '0.75倍', '1.5倍', '2倍']"
                      :key="speed"
                      type="button"
                      :class="{ active: gifSpeed === speed }"
                      @click="gifSpeed = speed"
                    >
                      {{ speed }}
                    </button>
                  </div>
                </div>
              </template>

              <template v-else-if="activeTool === 'merge'">
                <div class="merge-file-panel">
                  <div class="merge-file-list">
                    <div
                      v-for="(item, index) in mergeFileItems"
                      :key="item.id"
                      class="merge-file-row"
                    >
                      <div class="merge-thumb-window">
                        <video
                          v-if="item.file.type.startsWith('video/')"
                          class="merge-thumb-video"
                          :src="item.url"
                          muted
                          playsinline
                          preload="metadata"
                        />
                        <span v-else>♪</span>
                      </div>
                      <div class="merge-file-copy">
                        <strong>{{ item.file.name }}</strong>
                        <small>
                          {{ formatFileSizeValue(item.file) }} · {{ item.file.type || 'media file' }}
                          <span v-if="index === 0 && duration"> · {{ durationLabel }}</span>
                        </small>
                      </div>
                      <button
                        class="merge-remove-button"
                        type="button"
                        :disabled="mergeFileItems.length <= 1"
                        @click="removeMergeFile(index)"
                      >
                        ×
                      </button>
                    </div>
                  </div>
                </div>
              </template>

              <template v-else-if="activeTool === 'trim'">
                <div class="trim-editor-panel">
                  <div class="trim-preview-stage">
                    <video
                      v-if="!isAudio"
                      ref="trimMediaRef"
                      class="trim-video"
                      :src="objectUrl"
                      playsinline
                      @loadedmetadata="handleTrimLoadedMetadata"
                      @timeupdate="handleTrimTimeUpdate"
                      @ended="handleTrimEnded"
                    />
                    <div v-else class="trim-audio-placeholder">
                      <span>♪</span>
                    </div>
                  </div>

                  <div class="trim-track-row">
                    <button class="trim-play-button" type="button" @click="toggleTrimPlay">
                      {{ isTrimPlaying ? 'Ⅱ' : '▶' }}
                    </button>
                    <div ref="trimTrackRef" class="trim-track" @click="seekTrimFromTrack">
                      <span class="trim-track-fill" :style="{ width: `${trimProgressPercent}%` }" />
                      <button
                        class="trim-cut-handle"
                        type="button"
                        :style="{ left: `${trimProgressPercent}%` }"
                        @pointerdown.stop.prevent="startTrimHandleDrag"
                      >
                        <i />
                      </button>
                    </div>
                    <span class="trim-time">{{ trimCurrentTimeLabel }} / {{ trimDurationLabel }}</span>
                  </div>
                </div>

                <div class="trim-actions">
                  <button class="trim-action-button split" type="button" @click="splitTrimAtCurrentTime">
                    分割
                  </button>
                  <button class="trim-action-button delete" type="button" @click="deleteTrimSegment">
                    删除
                  </button>
                </div>
              </template>

              <template v-else-if="activeTool === 'crop'">
                <div class="crop-editor-panel">
                  <div ref="cropStageRef" class="crop-stage">
                    <video
                      v-if="!isAudio"
                      ref="cropMediaRef"
                      class="crop-video"
                      :src="objectUrl"
                      playsinline
                      @loadedmetadata="handleCropLoadedMetadata"
                      @timeupdate="handleCropTimeUpdate"
                      @ended="handleCropEnded"
                    />
                    <div v-else class="crop-audio-placeholder">
                      <span>♪</span>
                    </div>
                    <div
                      class="crop-frame"
                      :style="cropBoxStyle"
                      @pointerdown.prevent="startCropDrag"
                    >
                      <i class="corner tl" />
                      <i class="corner tr" />
                      <i class="corner bl" />
                      <i class="corner br" />
                    </div>
                  </div>

                  <div class="crop-player-bar">
                    <button class="crop-play-button" type="button" @click="toggleCropPlay">
                      {{ isCropPlaying ? 'Ⅱ' : '▶' }}
                    </button>
                    <div class="crop-progress" @click="seekCropFromBar">
                      <span :style="{ width: `${cropProgressPercent}%` }" />
                    </div>
                    <span>{{ cropCurrentTimeLabel }} / {{ cropDurationLabel }}</span>
                  </div>
                </div>

                <div class="compact-control">
                  <span>裁剪比例</span>
                  <div class="ratio-options">
                    <button
                      v-for="ratio in [
                        { value: 'wide', label: '16:9(宽屏)' },
                        { value: 'square', label: '1:1(正方形)' },
                        { value: 'vertical', label: '9:16(手机竖屏)' },
                        { value: 'classic', label: '4:3(旧屏幕)' },
                        { value: 'original', label: '原始比例' },
                        { value: 'free', label: '自由比例' },
                      ]"
                      :key="ratio.value"
                      type="button"
                      :class="{ active: cropRatio === ratio.value }"
                      @click="setCropRatio(ratio.value)"
                    >
                      <span class="radio-dot" />
                      {{ ratio.label }}
                    </button>
                  </div>
                </div>
              </template>

              <template v-else>
                <div
                  v-for="setting in activeExportSettings"
                  :key="setting.id"
                  class="compact-control"
                >
                  <span>{{ setting.label }}</span>
                  <div class="segmented-row">
                    <button
                      v-for="option in setting.options"
                      :key="option"
                      type="button"
                      :class="{ active: selectedOption(setting) === option }"
                      @click="selectSettingOption(setting, option)"
                    >
                      {{ option }}
                    </button>
                  </div>
                  <small v-if="setting.hint">{{ setting.hint }}</small>
                </div>

                <div v-if="activeTool !== 'mp4-to-mp3' && activeTool !== 'mp3-converter'" class="compact-control">
                  <span>导出质量</span>
                  <div class="segmented-row">
                    <button
                      v-for="quality in ['720p', '1080p', '4K']"
                      :key="quality"
                      type="button"
                      :class="{ active: outputQuality === quality }"
                      @click="outputQuality = quality"
                    >
                      {{ quality }}
                    </button>
                  </div>
                </div>
              </template>
            </section>
          </div>

          <div class="settings-footer">
            <div class="panel-actions">
              <button class="gradient-button" type="button" :disabled="isProcessing" @click="startProcess">
                {{ isProcessing ? '导出处理中...' : '开始导出' }}
              </button>
              <button
                v-if="activeTool === 'merge'"
                class="outline-button"
                type="button"
                @click="openMergePicker"
              >
                添加文件
              </button>
              <button v-else class="outline-button" type="button" @click="downloadOriginal">下载原文件</button>
            </div>
          </div>
        </aside>
      </div>
    </template>

    <input ref="fileInputRef" type="file" accept="video/*,audio/*" hidden @change="handleFileInput">
    <input ref="replaceInputRef" type="file" accept="video/*,audio/*" hidden @change="handleFileInput">
    <input ref="mergeInputRef" type="file" accept="video/*" multiple hidden @change="handleMergeFileInput">
  </section>
</template>

<style scoped>
.editor-shell {
  width: min(1120px, 100%);
  overflow: hidden;
  border: 1px solid rgb(255 255 255 / 16%);
  border-radius: 28px;
  background: rgb(59 71 115 / 62%);
  box-shadow: 0 28px 100px rgb(0 0 0 / 35%);
  backdrop-filter: blur(40px);
}

.editor-shell.workspace-mode {
  width: min(96vw, 1880px);
  height: 100%;
  border: 0;
  background: transparent;
  box-shadow: none;
  backdrop-filter: none;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

button {
  font: inherit;
}

.gradient-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 188px;
  min-height: 56px;
  padding: 0 24px;
  border: 0;
  border-radius: 999px;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  color: #091018;
  cursor: pointer;
  font-weight: 800;
  text-decoration: none;
  box-shadow: 0 18px 46px rgb(128 250 163 / 22%);
}

.secondary-button,
.outline-button,
.pill-button,
.back-button {
  border: 1px solid rgb(255 255 255 / 16%);
  background: rgb(255 255 255 / 8%);
  color: #fff;
  cursor: pointer;
}

.upload-panel {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 24px;
  padding: clamp(34px, 6vw, 70px) 24px;
  border: 1px dashed rgb(255 255 255 / 24%);
  border-radius: 24px;
  background:
    radial-gradient(circle at center top, rgb(118 249 177 / 16%), transparent 42%),
    rgb(15 22 45 / 58%);
  text-align: center;
  transition: border-color 0.2s ease, background 0.2s ease;
}

.upload-panel.dragging {
  border-color: #76f9b1;
  background:
    radial-gradient(circle at center top, rgb(118 249 177 / 24%), transparent 44%),
    rgb(15 22 45 / 70%);
}

.upload-icon {
  display: grid;
  width: 86px;
  height: 86px;
  margin-bottom: 24px;
  place-items: center;
  border-radius: 28px;
  background: rgb(255 255 255 / 10%);
}

.upload-icon span {
  position: relative;
  width: 42px;
  height: 34px;
  border: 3px solid #76f9b1;
  border-radius: 10px;
}

.upload-icon span::before {
  position: absolute;
  top: 9px;
  left: 13px;
  width: 0;
  height: 0;
  border-top: 8px solid transparent;
  border-bottom: 8px solid transparent;
  border-left: 13px solid #d6fb72;
  content: "";
}

.upload-panel h2 {
  margin: 0;
  font-size: clamp(24px, 3vw, 36px);
  line-height: 1.25;
}

.upload-panel p {
  max-width: 680px;
  margin: 14px 0 0;
  color: rgb(255 255 255 / 68%);
  font-size: 16px;
  line-height: 1.65;
}

.upload-actions {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 16px;
  margin-top: 30px;
}

.secondary-button {
  min-width: 160px;
  min-height: 56px;
  padding: 0 24px;
  border-radius: 999px;
  font-weight: 700;
}

.format-text {
  font-size: 14px;
}

.upload-card-list {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 18px;
  padding: 0 24px 24px;
  margin: 0;
  list-style: none;
}

.upload-card-list li {
  display: flex;
  gap: 14px;
  padding: 18px;
  border-radius: 18px;
  background: rgb(16 24 48 / 72%);
}

.upload-card-list img {
  width: 42px;
  height: 42px;
}

.upload-card-list h3 {
  margin: 0;
  font-size: 16px;
}

.upload-card-list p {
  margin: 8px 0 0;
  color: rgb(255 255 255 / 58%);
  font-size: 13px;
  line-height: 1.55;
}

.workspace-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex: 0 0 auto;
  margin-bottom: 12px;
  color: rgb(255 255 255 / 72%);
}

.back-button {
  min-height: 38px;
  padding: 0 16px;
  border-radius: 999px;
}

.workspace-grid {
  display: grid;
  grid-template-columns: minmax(0, 1.45fr) minmax(360px, 0.72fr);
  gap: 20px;
  flex: 1;
  min-height: 0;
}

.library-panel,
.export-panel,
.preview-panel,
.settings-panel {
  border-radius: 12px;
  background: #1b1c21;
}

.library-panel,
.export-panel {
  display: flex;
  min-height: 0;
  overflow: hidden;
  flex-direction: column;
  border: 1px solid rgb(255 255 255 / 8%);
}

.file-summary-row {
  display: grid;
  grid-template-columns: 132px minmax(0, 1fr) auto;
  align-items: center;
  gap: 18px;
  flex: 0 0 auto;
  min-height: 126px;
  padding: 20px 24px;
  border-bottom: 1px solid rgb(255 255 255 / 10%);
  background:
    radial-gradient(circle at left center, rgb(118 249 177 / 12%), transparent 38%),
    rgb(255 255 255 / 4%);
}

.thumb-window {
  display: grid;
  width: 132px;
  height: 82px;
  place-items: center;
  overflow: hidden;
  border: 1px solid rgb(255 255 255 / 14%);
  border-radius: 10px;
  background: #101217;
}

.thumb-video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.audio-thumb {
  display: grid;
  width: 100%;
  height: 100%;
  place-items: center;
  background: radial-gradient(circle at center, rgb(118 249 177 / 20%), transparent 44%), #101217;
}

.audio-thumb span {
  display: grid;
  width: 42px;
  height: 42px;
  place-items: center;
  border-radius: 999px;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  color: #091018;
  font-weight: 900;
}

.file-meta {
  display: grid;
  min-width: 0;
  gap: 8px;
}

.file-meta span,
.active-tool-head div > span {
  color: #8dfb87;
  font-size: 12px;
  font-weight: 900;
}

.file-meta h2 {
  overflow: hidden;
  margin: 0;
  font-size: 22px;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.file-meta p {
  margin: 0;
  color: rgb(255 255 255 / 58%);
  font-size: 13px;
}

.file-row-actions {
  display: flex;
  flex-wrap: wrap;
  justify-content: flex-end;
  gap: 10px;
}

.library-scroll,
.export-scroll {
  display: flex;
  min-height: 0;
  flex: 1;
  flex-direction: column;
  gap: 14px;
  overflow-y: auto;
  padding: 22px 24px;
  scrollbar-width: thin;
  scrollbar-color: rgb(118 249 177 / 48%) transparent;
}

.preview-panel {
  display: flex;
  min-height: 0;
  overflow: hidden;
  flex-direction: column;
  border: 1px solid rgb(255 255 255 / 8%);
}

.media-stage {
  position: relative;
  display: grid;
  flex: 1;
  min-height: 0;
  place-items: center;
  overflow: hidden;
}

.media-preview {
  width: 100%;
  height: 100%;
  max-height: 72vh;
  object-fit: contain;
  background: #191a20;
}

.audio-preview {
  display: grid;
  width: 100%;
  height: 100%;
  place-items: center;
  background: radial-gradient(circle at center, rgb(118 249 177 / 18%), transparent 34%), #191a20;
}

.audio-disc {
  display: grid;
  width: 190px;
  height: 190px;
  place-items: center;
  border-radius: 999px;
  background: rgb(255 255 255 / 8%);
}

.audio-disc span {
  width: 72px;
  height: 72px;
  border-radius: 999px;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
}

.selection-box {
  position: absolute;
  right: 18%;
  bottom: 16%;
  width: 180px;
  height: 74px;
  border: 2px dashed rgb(255 255 255 / 78%);
}

.selection-label {
  position: absolute;
  top: -28px;
  left: 8px;
  padding: 4px 10px;
  border-radius: 999px;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  color: #081018;
  font-size: 13px;
  font-weight: 800;
}

.handle {
  position: absolute;
  width: 10px;
  height: 10px;
  border-radius: 999px;
  background: #8dfb87;
  box-shadow: 0 0 0 2px rgb(141 251 135 / 35%);
}

.tl {
  top: -6px;
  left: -6px;
}

.tr {
  top: -6px;
  right: -6px;
}

.bl {
  bottom: -6px;
  left: -6px;
}

.br {
  right: -6px;
  bottom: -6px;
}

.tm {
  top: -6px;
  left: 50%;
  transform: translateX(-50%);
}

.bm {
  bottom: -6px;
  left: 50%;
  transform: translateX(-50%);
}

.player-bar {
  display: flex;
  align-items: center;
  gap: 14px;
  min-height: 60px;
  padding: 0 18px;
  border-top: 1px solid rgb(255 255 255 / 28%);
}

.play-button {
  border: 0;
  background: transparent;
  color: #fff;
  cursor: pointer;
  font-size: 22px;
}

.time-text {
  min-width: 108px;
  color: #fff;
  font-weight: 800;
}

.time-text small {
  color: rgb(255 255 255 / 48%);
}

.progress-track {
  position: relative;
  height: 6px;
  flex: 1;
  overflow: hidden;
  border-radius: 999px;
  background: rgb(255 255 255 / 16%);
  cursor: pointer;
}

.progress-track span {
  position: absolute;
  inset: 0 auto 0 0;
  border-radius: inherit;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
}

.pill-button {
  min-height: 34px;
  padding: 0 20px;
  border: 0;
  border-radius: 999px;
  background: rgb(255 255 255 / 9%);
  font-weight: 800;
}

.settings-panel {
  display: flex;
  min-height: 0;
  overflow: hidden;
  flex-direction: column;
  padding: 0;
  color: #fff;
}

.settings-scroll {
  display: flex;
  min-height: 0;
  flex: 1;
  flex-direction: column;
  gap: 14px;
  overflow-y: auto;
  padding: 22px 30px 16px;
  scrollbar-width: thin;
  scrollbar-color: rgb(118 249 177 / 48%) transparent;
}

.panel-title {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 16px;
}

.panel-title span {
  color: #8dfb87;
  font-size: 13px;
  font-weight: 800;
}

.panel-title h2 {
  margin: 4px 0 0;
  font-size: 22px;
  line-height: 1.2;
}

.panel-title strong {
  display: grid;
  min-width: 34px;
  height: 34px;
  place-items: center;
  border-radius: 999px;
  background: rgb(118 249 177 / 14%);
  color: #8dfb87;
  font-size: 14px;
}

.segmented-row button {
  border: 1px solid rgb(255 255 255 / 16%);
  background: rgb(255 255 255 / 6%);
  color: rgb(255 255 255 / 72%);
  cursor: pointer;
  font-weight: 800;
}

.segmented-row button.active {
  border-color: #76f9b1;
  background: rgb(118 249 177 / 12%);
  color: #fff;
  box-shadow: inset 0 0 0 1px rgb(118 249 177 / 22%);
}

.tool-section h3,
.active-tool-panel h3 {
  margin: 0;
  font-size: 15px;
}

.tool-board {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 10px;
  margin-top: 10px;
}

.tool-card {
  position: relative;
  display: grid;
  grid-template-columns: 34px 1fr;
  align-items: center;
  gap: 10px;
  min-height: 78px;
  padding: 12px;
  border: 1px solid rgb(255 255 255 / 14%);
  border-radius: 8px;
  background: rgb(255 255 255 / 5%);
  color: #fff;
  cursor: pointer;
  text-align: left;
}

.tool-card.active {
  border-color: #76f9b1;
  background: linear-gradient(135deg, rgb(214 251 114 / 14%), rgb(118 249 177 / 9%));
}

.tool-card.external {
  border-color: rgb(214 251 114 / 34%);
}

.tool-icon {
  display: grid;
  width: 34px;
  height: 34px;
  place-items: center;
  border-radius: 8px;
  background: rgb(118 249 177 / 13%);
  color: #8dfb87;
  font-size: 13px;
  font-weight: 900;
}

.tool-copy {
  display: grid;
  min-width: 0;
  gap: 4px;
}

.tool-copy strong {
  overflow: hidden;
  font-size: 14px;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.tool-copy small,
.active-tool-panel p,
.windows-download-card small,
.file-info span {
  color: rgb(255 255 255 / 58%);
  font-size: 12px;
  line-height: 1.45;
}

.tool-card em {
  position: absolute;
  top: 8px;
  right: 8px;
  padding: 2px 6px;
  border-radius: 999px;
  background: rgb(214 251 114 / 16%);
  color: #d6fb72;
  font-size: 11px;
  font-style: normal;
  font-weight: 800;
}

.active-tool-panel {
  display: grid;
  gap: 14px;
  padding: 14px;
  border: 1px solid rgb(255 255 255 / 12%);
  border-radius: 10px;
  background: rgb(0 0 0 / 16%);
}

.active-tool-head {
  display: grid;
  grid-template-columns: 38px 1fr;
  gap: 12px;
}

.active-tool-head .tool-icon {
  width: 38px;
  height: 38px;
}

.active-tool-panel p {
  margin: 4px 0 0;
}

.export-settings {
  display: grid;
  gap: 14px;
}

.export-settings h3 {
  margin: 0;
  font-size: 18px;
}

.export-settings .compact-control {
  padding: 14px;
  border: 1px solid rgb(255 255 255 / 10%);
  border-radius: 10px;
  background: rgb(255 255 255 / 4%);
}

.compact-control small {
  color: rgb(255 255 255 / 48%);
  font-size: 12px;
  line-height: 1.5;
}

.compact-control {
  display: grid;
  gap: 8px;
}

.compact-control > span {
  color: rgb(255 255 255 / 72%);
  font-size: 13px;
  font-weight: 800;
}

.segmented-row {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(86px, 1fr));
  gap: 8px;
}

.segmented-row.two-column {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.segmented-row.format-grid {
  grid-template-columns: repeat(auto-fit, minmax(72px, 1fr));
}

.segmented-row button {
  min-height: 34px;
  padding: 0 8px;
  border-radius: 999px;
  font-size: 13px;
}

.size-control {
  display: grid;
  gap: 12px;
  padding: 12px;
  border-radius: 10px;
  background: rgb(0 0 0 / 18%);
}

.range-head {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 14px;
}

.range-head strong {
  color: #8dfb87;
  font-size: 24px;
  line-height: 1;
}

.range-head span {
  color: rgb(255 255 255 / 54%);
  font-size: 12px;
}

.size-control input[type="range"] {
  width: 100%;
  accent-color: #76f9b1;
}

.number-control {
  display: grid;
  grid-template-columns: 1fr 116px 36px;
  align-items: center;
  gap: 10px;
  padding: 12px;
  border-radius: 10px;
  background: rgb(0 0 0 / 18%);
}

.number-control span {
  color: rgb(255 255 255 / 72%);
  font-size: 13px;
  font-weight: 800;
}

.number-control input,
.select-control select {
  width: 100%;
  min-height: 38px;
  border: 1px solid rgb(255 255 255 / 16%);
  border-radius: 999px;
  outline: 0;
  background: rgb(255 255 255 / 7%);
  color: #fff;
  font: inherit;
  font-weight: 800;
}

.number-control input {
  padding: 0 14px;
  text-align: center;
}

.number-control strong {
  color: rgb(255 255 255 / 64%);
  font-size: 13px;
}

.select-control select {
  padding: 0 16px;
  cursor: pointer;
}

.merge-file-panel {
  display: grid;
  gap: 10px;
  padding: 14px;
  border: 1px solid rgb(255 255 255 / 10%);
  border-radius: 12px;
  background: rgb(118 249 177 / 7%);
}

.merge-file-list {
  display: grid;
  gap: 10px;
}

.merge-file-row {
  display: grid;
  grid-template-columns: 104px minmax(0, 1fr) 32px;
  align-items: center;
  gap: 12px;
  min-height: 78px;
  padding: 12px;
  border: 1px solid rgb(255 255 255 / 10%);
  border-radius: 10px;
  background: rgb(255 255 255 / 6%);
}

.merge-thumb-window {
  display: grid;
  width: 104px;
  height: 64px;
  place-items: center;
  overflow: hidden;
  border: 1px solid rgb(255 255 255 / 12%);
  border-radius: 8px;
  background: #101217;
  color: #76f9b1;
  font-weight: 900;
}

.merge-thumb-video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.merge-file-copy {
  display: grid;
  min-width: 0;
  gap: 5px;
}

.merge-file-copy strong,
.merge-file-copy small {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.merge-file-copy strong {
  color: #fff;
  font-size: 14px;
}

.merge-file-copy small {
  color: rgb(255 255 255 / 58%);
  font-size: 12px;
}

.merge-remove-button {
  display: grid;
  width: 28px;
  height: 28px;
  place-items: center;
  border: 1px solid rgb(255 255 255 / 24%);
  border-radius: 999px;
  background: rgb(255 255 255 / 6%);
  color: rgb(255 255 255 / 72%);
  cursor: pointer;
  font-size: 18px;
  line-height: 1;
}

.merge-remove-button:disabled {
  cursor: default;
  opacity: 0.42;
}

.trim-editor-panel {
  overflow: hidden;
  border: 1px solid rgb(255 255 255 / 10%);
  border-radius: 12px;
  background: rgb(0 0 0 / 22%);
}

.trim-preview-stage {
  display: grid;
  width: 100%;
  aspect-ratio: 16 / 9;
  place-items: center;
  overflow: hidden;
  background: #101217;
}

.trim-video {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.trim-audio-placeholder {
  display: grid;
  width: 100%;
  height: 100%;
  place-items: center;
  background: radial-gradient(circle at center, rgb(118 249 177 / 18%), transparent 42%), #101217;
}

.trim-audio-placeholder span {
  display: grid;
  width: 52px;
  height: 52px;
  place-items: center;
  border-radius: 999px;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  color: #091018;
  font-weight: 900;
}

.trim-track-row {
  display: grid;
  grid-template-columns: 40px minmax(0, 1fr) auto;
  align-items: center;
  gap: 12px;
  min-height: 68px;
  padding: 0 14px;
  border-top: 1px solid rgb(255 255 255 / 12%);
}

.trim-play-button {
  display: grid;
  width: 34px;
  height: 34px;
  place-items: center;
  border: 0;
  border-radius: 999px;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  color: #091018;
  cursor: pointer;
  font-size: 14px;
  font-weight: 900;
}

.trim-track {
  position: relative;
  height: 30px;
  border-radius: 7px;
  background: rgb(53 232 94 / 36%);
  box-shadow: inset 0 0 0 1px rgb(255 255 255 / 8%);
  cursor: pointer;
}

.trim-track-fill {
  position: absolute;
  inset: 0 auto 0 0;
  border-radius: inherit;
  background: linear-gradient(90deg, #1ebc3e, #76f9b1);
}

.trim-cut-handle {
  position: absolute;
  top: 50%;
  display: grid;
  width: 28px;
  height: 42px;
  place-items: center;
  border: 0;
  border-radius: 999px;
  background: #202228;
  color: #76f9b1;
  cursor: ew-resize;
  font-weight: 900;
  transform: translate(-50%, -50%);
  touch-action: none;
  box-shadow:
    0 0 0 2px rgb(118 249 177 / 24%),
    0 8px 18px rgb(0 0 0 / 28%);
}

.trim-cut-handle::before {
  position: absolute;
  top: -10px;
  width: 4px;
  height: 16px;
  border-radius: 999px;
  background: #76f9b1;
  content: "";
}

.trim-cut-handle::after {
  position: absolute;
  bottom: -10px;
  width: 4px;
  height: 16px;
  border-radius: 999px;
  background: #76f9b1;
  content: "";
}

.trim-cut-handle i {
  width: 15px;
  height: 15px;
  background:
    linear-gradient(45deg, transparent 41%, currentcolor 42% 58%, transparent 59%),
    linear-gradient(-45deg, transparent 41%, currentcolor 42% 58%, transparent 59%);
}

.trim-time {
  color: rgb(255 255 255 / 72%);
  font-size: 12px;
  white-space: nowrap;
}

.trim-actions {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}

.trim-action-button {
  min-height: 42px;
  border-radius: 999px;
  cursor: pointer;
  font-weight: 900;
}

.trim-action-button.split {
  border: 0;
  background: linear-gradient(90deg, #35e85e, #76f9b1);
  color: #091018;
}

.trim-action-button.delete {
  border: 1px solid rgb(255 126 126 / 64%);
  background: rgb(255 126 126 / 8%);
  color: #ffb0b0;
}

.crop-editor-panel {
  overflow: hidden;
  border: 1px solid rgb(255 255 255 / 10%);
  border-radius: 12px;
  background: rgb(0 0 0 / 22%);
}

.crop-stage {
  position: relative;
  display: grid;
  width: 100%;
  aspect-ratio: 16 / 9;
  place-items: center;
  overflow: hidden;
  background: #101217;
}

.crop-video {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.crop-audio-placeholder {
  display: grid;
  width: 100%;
  height: 100%;
  place-items: center;
  background: radial-gradient(circle at center, rgb(118 249 177 / 18%), transparent 42%), #101217;
}

.crop-audio-placeholder span {
  display: grid;
  width: 52px;
  height: 52px;
  place-items: center;
  border-radius: 999px;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  color: #091018;
  font-weight: 900;
}

.crop-frame {
  position: absolute;
  border: 2px solid #35e85e;
  cursor: move;
  box-shadow:
    0 0 0 999px rgb(0 0 0 / 35%),
    0 0 18px rgb(53 232 94 / 26%);
  touch-action: none;
}

.crop-frame::before,
.crop-frame::after {
  position: absolute;
  inset: 33.33% 0 auto;
  height: 1px;
  background: rgb(255 255 255 / 36%);
  content: "";
}

.crop-frame::after {
  inset: 66.66% 0 auto;
}

.corner {
  position: absolute;
  width: 10px;
  height: 10px;
  border-radius: 999px;
  background: #7cff88;
  box-shadow: 0 0 0 2px rgb(124 255 136 / 30%);
}

.corner.tl {
  top: -6px;
  left: -6px;
}

.corner.tr {
  top: -6px;
  right: -6px;
}

.corner.bl {
  bottom: -6px;
  left: -6px;
}

.corner.br {
  right: -6px;
  bottom: -6px;
}

.crop-player-bar {
  display: grid;
  grid-template-columns: 40px minmax(0, 1fr) auto;
  align-items: center;
  gap: 12px;
  min-height: 52px;
  padding: 0 14px;
  border-top: 1px solid rgb(255 255 255 / 12%);
}

.crop-play-button {
  display: grid;
  width: 34px;
  height: 34px;
  place-items: center;
  border: 0;
  border-radius: 999px;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  color: #091018;
  cursor: pointer;
  font-size: 14px;
  font-weight: 900;
}

.crop-progress {
  position: relative;
  height: 5px;
  overflow: hidden;
  border-radius: 999px;
  background: rgb(255 255 255 / 18%);
  cursor: pointer;
}

.crop-progress span {
  position: absolute;
  inset: 0 auto 0 0;
  border-radius: inherit;
  background: #35e85e;
}

.crop-player-bar > span {
  color: rgb(255 255 255 / 72%);
  font-size: 12px;
  white-space: nowrap;
}

.ratio-options {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
}

.ratio-options button {
  display: flex;
  align-items: center;
  gap: 8px;
  min-height: 38px;
  padding: 0 12px;
  border: 1px solid rgb(255 255 255 / 16%);
  border-radius: 999px;
  background: rgb(255 255 255 / 6%);
  color: rgb(255 255 255 / 72%);
  cursor: pointer;
  font-weight: 800;
}

.ratio-options button.active {
  border-color: #76f9b1;
  color: #fff;
}

.radio-dot {
  display: inline-block;
  width: 16px;
  height: 16px;
  flex: 0 0 auto;
  border: 1px solid rgb(255 255 255 / 42%);
  border-radius: 999px;
}

.ratio-options button.active .radio-dot {
  border: 5px solid #76f9b1;
}

.gif-range-control {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(120px, 0.7fr);
  gap: 10px;
}

.gif-range-control button,
.client-link-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  min-height: 40px;
  border: 1px solid rgb(255 255 255 / 16%);
  border-radius: 999px;
  background: rgb(255 255 255 / 6%);
  color: rgb(255 255 255 / 72%);
  cursor: pointer;
  font-weight: 900;
  text-decoration: none;
}

.gif-range-control button.active,
.client-link-button.active {
  border-color: #76f9b1;
  background: rgb(118 249 177 / 12%);
  color: #fff;
  box-shadow: inset 0 0 0 1px rgb(118 249 177 / 22%);
}

.gif-range-control input {
  width: 54px;
  min-height: 28px;
  border: 1px solid rgb(255 255 255 / 16%);
  border-radius: 999px;
  outline: 0;
  background: rgb(0 0 0 / 18%);
  color: #fff;
  font: inherit;
  font-weight: 900;
  text-align: center;
}

.custom-select {
  position: relative;
  z-index: 5;
}

.select-trigger {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 18px;
  align-items: center;
  width: 100%;
  min-height: 48px;
  padding: 0 16px;
  border: 1px solid rgb(255 255 255 / 14%);
  border-radius: 8px;
  background: #2b2c33;
  color: #fff;
  cursor: pointer;
  font-weight: 800;
  text-align: left;
}

.select-trigger span {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.select-trigger i {
  color: rgb(255 255 255 / 72%);
  font-style: normal;
  text-align: right;
  transition: transform 0.18s ease;
}

.custom-select.open .select-trigger {
  border-color: rgb(118 249 177 / 48%);
  box-shadow: inset 0 0 0 1px rgb(118 249 177 / 14%);
}

.custom-select.open .select-trigger i {
  transform: rotate(180deg);
}

.select-menu {
  position: absolute;
  z-index: 20;
  top: calc(100% + 8px);
  right: 0;
  left: 0;
  overflow: hidden;
  border: 1px solid rgb(255 255 255 / 14%);
  border-radius: 8px;
  background: #2b2c33;
  box-shadow: 0 18px 48px rgb(0 0 0 / 36%);
}

.select-options {
  max-height: 220px;
  overflow-y: auto;
  padding: 8px 0;
  scrollbar-width: thin;
  scrollbar-color: rgb(118 249 177 / 40%) transparent;
}

.select-options button {
  display: block;
  width: 100%;
  min-height: 38px;
  padding: 0 16px;
  border: 0;
  background: transparent;
  color: #fff;
  cursor: pointer;
  font-weight: 700;
  text-align: left;
}

.select-options button:hover,
.select-options button.active {
  background: rgb(255 255 255 / 5%);
  color: #a8ff7a;
}

.select-options p {
  margin: 0;
  padding: 12px 16px 16px;
  color: rgb(255 255 255 / 54%);
  font-size: 13px;
}

.windows-download-card {
  display: grid;
  grid-template-columns: minmax(0, 1fr) auto;
  align-items: end;
  gap: 20px;
  padding: 18px 22px;
  overflow: hidden;
  border: 1px solid rgb(214 251 114 / 28%);
  border-radius: 12px;
  background:
    radial-gradient(circle at right top, rgb(118 249 177 / 26%), transparent 38%),
    rgb(255 255 255 / 6%);
  color: #fff;
}

.windows-download-card strong {
  display: block;
  font-size: 18px;
  line-height: 1.35;
}

.windows-download-card small {
  display: block;
  margin-top: 8px;
}

.windows-download-card b {
  color: #8dfb87;
  font-weight: 900;
}

.download-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 132px;
  min-height: 44px;
  padding: 0 22px;
  border-radius: 999px;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  color: #091018;
  font-weight: 900;
  text-decoration: none;
  box-shadow: 0 16px 38px rgb(128 250 163 / 18%);
}

.settings-footer {
  display: grid;
  gap: 14px;
  flex: 0 0 auto;
  padding: 14px 30px 22px;
  border-top: 1px solid rgb(255 255 255 / 10%);
  background: linear-gradient(180deg, rgb(27 28 33 / 92%), #1b1c21);
}

.file-info {
  display: grid;
  gap: 6px;
  padding: 12px 14px;
  border-radius: 10px;
  background: rgb(255 255 255 / 6%);
}

.file-info strong {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.panel-actions {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(0, 0.85fr);
  gap: 12px;
}

.panel-actions .gradient-button,
.panel-actions .outline-button {
  min-width: 0;
  min-height: 42px;
}

@media (max-height: 820px) and (min-width: 1181px) {
  .library-scroll,
  .export-scroll {
    gap: 10px;
    padding: 16px 24px 12px;
  }

  .file-summary-row {
    min-height: 104px;
    padding: 14px 22px;
  }

  .thumb-window {
    width: 116px;
    height: 70px;
  }

  .file-summary-row {
    grid-template-columns: 116px minmax(0, 1fr) auto;
  }

  .panel-title h2 {
    font-size: 19px;
  }

  .tool-board {
    gap: 8px;
  }

  .tool-card {
    min-height: 66px;
    padding: 10px;
  }

  .active-tool-panel {
    gap: 10px;
    padding: 12px;
  }

  .file-info {
    padding: 10px 12px;
  }

  .tool-copy small,
  .active-tool-panel p,
  .windows-download-card small,
  .file-info span {
    font-size: 12px;
  }

  .settings-footer {
    gap: 10px;
    padding: 12px 24px 16px;
  }
}

.outline-button {
  border-radius: 999px;
  font-weight: 800;
}

.gradient-button:disabled {
  cursor: default;
  opacity: 0.7;
}

@media (max-width: 1180px) {
  .workspace-grid {
    grid-template-columns: 1fr;
  }

  .library-panel,
  .export-panel,
  .preview-panel,
  .settings-panel {
    min-height: auto;
  }

  .media-stage,
  .audio-preview {
    min-height: 420px;
  }
}

@media (max-width: 1023px) {
  .upload-card-list {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 640px) {
  .editor-shell {
    border-radius: 22px;
  }

  .upload-actions,
  .panel-actions,
  .tool-board,
  .segmented-row,
  .ratio-options,
  .gif-range-control,
  .trim-actions,
  .windows-download-card {
    grid-template-columns: 1fr;
  }

  .trim-track-row {
    grid-template-columns: 40px minmax(0, 1fr);
  }

  .trim-time {
    grid-column: 2;
  }

  .file-summary-row {
    grid-template-columns: 92px minmax(0, 1fr);
  }

  .thumb-window {
    width: 92px;
    height: 64px;
  }

  .file-row-actions {
    grid-column: 1 / -1;
    justify-content: flex-start;
  }

  .upload-actions {
    display: grid;
    width: min(100%, 280px);
  }

  .gradient-button,
  .secondary-button {
    width: 100%;
  }

  .upload-panel {
    margin: 14px;
    padding: 32px 16px;
  }

  .upload-card-list {
    padding: 0 14px 14px;
  }

  .player-bar {
    flex-wrap: wrap;
    padding: 12px 14px;
  }

  .progress-track {
    order: 5;
    flex-basis: 100%;
  }

  .library-scroll,
  .export-scroll {
    padding: 18px;
  }

  .settings-footer {
    padding: 14px 18px 18px;
  }
}
</style>
