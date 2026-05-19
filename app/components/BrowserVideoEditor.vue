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

type MergeMediaInfo = {
  duration: number
  width: number
  height: number
  hasAudio: boolean
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
const MB = 1024 * 1024
const MAX_WEB_COMPRESS_SIZE = 500 * MB
const MAX_WEB_COMPRESS_DURATION = 30 * 60
const WINDOWS_DOWNLOAD_URL = 'https://download.aoscdn.com/down.php?softid=reccloud'
const FFMPEG_LOAD_TIMEOUT = 45_000
const FFMPEG_CORE_BASE_URL = 'https://unpkg.com/@ffmpeg/core@0.12.10/dist/umd'

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
const gifStartSecond = ref(0)
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
  width: number
  height: number
  mode: 'move' | 'resize'
  handle?: 'tl' | 'tr' | 'bl' | 'br'
} | null>(null)
const isTrimHandleDragging = ref(false)
const trimRangeStart = ref(0)
const trimRangeEnd = ref(0)
const selectedSettingOptions = ref<Record<string, string>>({})
const isProcessing = ref(false)
const compressionProgress = ref(0)
const compressionStatus = ref('')
const compressionError = ref('')
const compressionDebugLog = ref<string[]>([])
const compressionOutputUrl = ref('')
const compressionOutputName = ref('')
const compressionOutputSize = ref(0)
const showWindowsCompressPrompt = ref(false)
const pendingTaskAction = ref<'replace' | 'reset' | null>(null)
let processTimerId: number | null = null
let taskToken = 0
let ffmpegInstance: {
  loaded: boolean
  load: (config?: Record<string, string>) => Promise<boolean>
  on: {
    (event: 'progress', callback: (payload: { progress: number }) => void): void
    (event: 'log', callback: (payload: { type: string; message: string }) => void): void
  }
  writeFile: (path: string, data: Uint8Array) => Promise<boolean>
  exec: (args: string[]) => Promise<number>
  readFile: (path: string) => Promise<Uint8Array | string>
  deleteFile: (path: string) => Promise<boolean>
  terminate: () => void
} | null = null
let activeProcessingFfmpeg: typeof ffmpegInstance | null = null

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
const trimSelectionStartPercent = computed(() => {
  const total = trimDuration.value || duration.value
  if (!total) return 0
  return Math.min(100, Math.max(0, (getTrimRange().start / total) * 100))
})
const trimSelectionEndPercent = computed(() => {
  const total = trimDuration.value || duration.value
  if (!total) return 100
  return Math.min(100, Math.max(0, (getTrimRange().end / total) * 100))
})
const trimSelectionStyle = computed(() => ({
  left: `${trimSelectionStartPercent.value}%`,
  width: `${Math.max(0, trimSelectionEndPercent.value - trimSelectionStartPercent.value)}%`,
}))
const trimRangeLabel = computed(() => {
  const range = getTrimRange()
  return `${formatTime(range.start)} - ${formatTime(range.end)}`
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
const compressionButtonText = computed(() => {
  if (activeTool.value === 'compress') {
    if (!isProcessing.value) return '开始压缩'
    if (compressionProgress.value > 0) return `压缩中 ${compressionProgress.value}%`
    return compressionStatus.value || '准备压缩...'
  }

  if (activeTool.value === 'convert') {
    if (!isProcessing.value) return '开始转换'
    if (compressionProgress.value > 0) return `转换中 ${compressionProgress.value}%`
    return compressionStatus.value || '准备转换...'
  }

  if (activeTool.value === 'gif-maker' || activeTool.value === 'video-to-gif') {
    if (!isProcessing.value) return '开始制作 GIF'
    if (compressionProgress.value > 0) return `GIF 制作中 ${compressionProgress.value}%`
    return compressionStatus.value || '准备制作 GIF...'
  }

  if (activeTool.value === 'mp3-converter' || activeTool.value === 'mp4-to-mp3') {
    if (!isProcessing.value) return '开始转换音频'
    if (compressionProgress.value > 0) return `音频转换中 ${compressionProgress.value}%`
    return compressionStatus.value || '准备转换音频...'
  }

  if (activeTool.value === 'merge') {
    if (!isProcessing.value) return '开始合并'
    if (compressionProgress.value > 0) return `合并中 ${compressionProgress.value}%`
    return compressionStatus.value || '准备合并...'
  }

  if (activeTool.value === 'trim') {
    if (!isProcessing.value) return '开始剪切'
    if (compressionProgress.value > 0) return `剪切中 ${compressionProgress.value}%`
    return compressionStatus.value || '准备剪切...'
  }

  if (activeTool.value === 'crop') {
    if (!isProcessing.value) return '开始裁剪'
    if (compressionProgress.value > 0) return `裁剪中 ${compressionProgress.value}%`
    return compressionStatus.value || '准备裁剪...'
  }

  return isProcessing.value ? '导出处理中...' : '开始导出'
})
const compressionResultSizeLabel = computed(() => {
  if (!compressionOutputSize.value) return ''
  if (compressionOutputSize.value >= 1024 * MB) return `${(compressionOutputSize.value / 1024 / MB).toFixed(2)} GB`
  return `${(compressionOutputSize.value / MB).toFixed(2)} MB`
})
const processResultSizePrefix = computed(() => {
  if (activeTool.value === 'convert') return '转换后大小：'
  if (activeTool.value === 'gif-maker' || activeTool.value === 'video-to-gif') return 'GIF 大小：'
  if (activeTool.value === 'mp3-converter' || activeTool.value === 'mp4-to-mp3') return '音频大小：'
  if (activeTool.value === 'merge') return '合并后大小：'
  if (activeTool.value === 'trim') return '剪切后大小：'
  if (activeTool.value === 'crop') return '裁剪后大小：'
  return '压缩后大小：'
})
const processDownloadText = computed(() => {
  if (activeTool.value === 'convert') return '下载转换文件'
  if (activeTool.value === 'gif-maker' || activeTool.value === 'video-to-gif') return '下载 GIF 文件'
  if (activeTool.value === 'mp3-converter' || activeTool.value === 'mp4-to-mp3') return '下载音频文件'
  if (activeTool.value === 'merge') return '下载合并文件'
  if (activeTool.value === 'trim') return '下载剪切文件'
  if (activeTool.value === 'crop') return '下载裁剪文件'
  return '下载压缩文件'
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
  stopCurrentTask()
  clearMergeFiles()
  revokeObjectUrl()
})

function openFilePicker() {
  fileInputRef.value?.click()
}

function openReplacePicker() {
  if (isProcessing.value) {
    pendingTaskAction.value = 'replace'
    return
  }
  openReplacePickerDirect()
}

function openReplacePickerDirect() {
  replaceInputRef.value?.click()
}

function openMergePicker() {
  if (isProcessing.value) return
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
  resetCompressionState()
  uploadedFile.value = file
  objectUrl.value = URL.createObjectURL(file)
  mergeFiles.value = [createMergeFileItem(file)]
  currentTime.value = 0
  duration.value = 0
  gifStartSecond.value = 0
  cropCurrentTime.value = 0
  cropDuration.value = 0
  trimCurrentTime.value = 0
  trimDuration.value = 0
  trimRangeStart.value = 0
  trimRangeEnd.value = 0
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

function resetCompressionState() {
  if (compressionOutputUrl.value) URL.revokeObjectURL(compressionOutputUrl.value)
  compressionProgress.value = 0
  compressionStatus.value = ''
  compressionError.value = ''
  compressionDebugLog.value = []
  compressionOutputUrl.value = ''
  compressionOutputName.value = ''
  compressionOutputSize.value = 0
  showWindowsCompressPrompt.value = false
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

function getFileExtension(name: string) {
  return name.split('.').pop()?.toLowerCase().replace(/[^a-z0-9]/g, '') || 'mp4'
}

function getCompressedFileName(name: string) {
  const baseName = name.replace(/\.[^/.]+$/, '') || 'video'
  return `${baseName}-compressed.mp4`
}

function getConvertedFileName(name: string, format: string) {
  const baseName = name.replace(/\.[^/.]+$/, '') || 'video'
  return `${baseName}-converted.${getConvertOutputExtension(format)}`
}

function getSelectedConvertFormat() {
  return (selectedSettingOptions.value.format || 'MP4').toUpperCase()
}

function getSelectedConvertCodec() {
  return selectedSettingOptions.value.codec || '同源编码'
}

function getConvertOutputExtension(format: string) {
  const extensionMap: Record<string, string> = {
    MP4: 'mp4',
    AVI: 'avi',
    MOV: 'mov',
    MPEG: 'mpeg',
    WMV: 'wmv',
    '3GP': '3gp',
    ASF: 'asf',
    MTS: 'mts',
    MKV: 'mkv',
    FLV: 'flv',
    M4V: 'm4v',
    XVID: 'avi',
    VOB: 'vob',
    DIVX: 'avi',
    MXF: 'mxf',
  }
  return extensionMap[format] || format.toLowerCase()
}

function getConvertMimeType(format: string) {
  const mimeMap: Record<string, string> = {
    MP4: 'video/mp4',
    AVI: 'video/x-msvideo',
    MOV: 'video/quicktime',
    MPEG: 'video/mpeg',
    WMV: 'video/x-ms-wmv',
    '3GP': 'video/3gpp',
    ASF: 'video/x-ms-asf',
    MTS: 'video/mp2t',
    MKV: 'video/x-matroska',
    FLV: 'video/x-flv',
    M4V: 'video/x-m4v',
    XVID: 'video/x-msvideo',
    VOB: 'video/dvd',
    DIVX: 'video/x-msvideo',
    MXF: 'application/mxf',
  }
  return mimeMap[format] || 'video/mp4'
}

function getAudioOutputFormat() {
  return (selectedSettingOptions.value.audioFormat || 'MP3').toUpperCase()
}

function getAudioQuality() {
  return selectedSettingOptions.value.audioQuality || '标准'
}

function getAudioOutputExtension(format: string) {
  const extensionMap: Record<string, string> = {
    MP3: 'mp3',
    M4A: 'm4a',
    WAV: 'wav',
    AAC: 'aac',
  }
  return extensionMap[format] || format.toLowerCase()
}

function getAudioMimeType(format: string) {
  const mimeMap: Record<string, string> = {
    MP3: 'audio/mpeg',
    M4A: 'audio/mp4',
    WAV: 'audio/wav',
    AAC: 'audio/aac',
  }
  return mimeMap[format] || 'audio/mpeg'
}

function getAudioOutputFileName(name: string, format: string) {
  const baseName = name.replace(/\.[^/.]+$/, '') || 'audio'
  return `${baseName}-audio.${getAudioOutputExtension(format)}`
}

function getGifOutputFileName(name: string) {
  const baseName = name.replace(/\.[^/.]+$/, '') || 'video'
  return `${baseName}-gif.gif`
}

function getMergedOutputFileName(name: string, format: string) {
  const baseName = name.replace(/\.[^/.]+$/, '') || 'video'
  return `${baseName}-merged.${getMergeOutputExtension(format)}`
}

function getTrimmedOutputFileName(name: string) {
  const baseName = name.replace(/\.[^/.]+$/, '') || 'video'
  return `${baseName}-trimmed.mp4`
}

function getCroppedOutputFileName(name: string) {
  const baseName = name.replace(/\.[^/.]+$/, '') || 'video'
  return `${baseName}-cropped.mp4`
}

function getMergeOutputFormat() {
  return (selectedSettingOptions.value.format || 'MP4').toUpperCase()
}

function getMergeOutputExtension(format: string) {
  const extensionMap: Record<string, string> = {
    MP4: 'mp4',
    MOV: 'mov',
    WEBM: 'webm',
  }
  return extensionMap[format] || 'mp4'
}

function getMergeMimeType(format: string) {
  const mimeMap: Record<string, string> = {
    MP4: 'video/mp4',
    MOV: 'video/quicktime',
    WEBM: 'video/webm',
  }
  return mimeMap[format] || 'video/mp4'
}

function handleLoadedMetadata() {
  const media = mediaRef.value
  duration.value = media?.duration || 0
  if (duration.value && !trimRangeEnd.value) trimRangeEnd.value = duration.value
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
  if (trimDuration.value && !trimRangeEnd.value) trimRangeEnd.value = trimDuration.value
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
  trimRangeStart.value = Math.min(trimCurrentTime.value, getTrimRange().end - 0.1)
}

function deleteTrimSegment() {
  trimRangeEnd.value = Math.max(trimCurrentTime.value, getTrimRange().start + 0.1)
}

function resetTrimRange() {
  trimRangeStart.value = 0
  trimRangeEnd.value = trimDuration.value || duration.value
}

function getTrimRange() {
  const total = trimDuration.value || duration.value || 0
  const start = Math.min(total, Math.max(0, trimRangeStart.value || 0))
  const end = trimRangeEnd.value ? Math.min(total, Math.max(0, trimRangeEnd.value)) : total
  if (end <= start) return { start: 0, end: total }
  return { start, end }
}

function seekFromBar(event: MouseEvent) {
  const media = mediaRef.value
  if (!media || !duration.value) return
  const rect = (event.currentTarget as HTMLElement).getBoundingClientRect()
  const ratio = (event.clientX - rect.left) / rect.width
  media.currentTime = Math.min(duration.value, Math.max(0, ratio * duration.value))
}

function downloadCompressedVideo() {
  if (!compressionOutputUrl.value || !compressionOutputName.value) return
  const link = document.createElement('a')
  link.href = compressionOutputUrl.value
  link.download = compressionOutputName.value
  link.click()
}

function removeMergeFile(index: number) {
  if (mergeFiles.value.length <= 1) return
  const item = mergeFiles.value[index]
  if (item) URL.revokeObjectURL(item.url)
  mergeFiles.value = mergeFiles.value.filter((_, itemIndex) => itemIndex !== index)
}

function getOrderedMergeItems() {
  const order = selectedSettingOptions.value.mergeOrder || '当前顺序'
  const items = [...mergeFiles.value]
  if (order === '按文件名') {
    return items.sort((a, b) => a.file.name.localeCompare(b.file.name, 'zh-Hans-CN'))
  }
  if (order === '按上传时间') {
    return items.sort((a, b) => a.file.lastModified - b.file.lastModified)
  }
  return items
}

function selectTool(tool: EditorTool) {
  if (tool.external) {
    if (tool.externalUrl) window.location.href = tool.externalUrl
    return
  }
  if (!isProcessing.value && activeTool.value !== tool.id) resetCompressionState()
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

  const targetRatio = getCropTargetRatio(value)
  if (!targetRatio) return

  const stage = cropStageRef.value
  const rect = stage?.getBoundingClientRect()
  const stageAspect = getCropStageAspect(rect)
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

function getCropTargetRatio(value = cropRatio.value) {
  if (value === 'free') return null

  const ratioMap: Record<string, number> = {
    wide: 16 / 9,
    square: 1,
    vertical: 9 / 16,
    classic: 4 / 3,
    original: videoWidth.value && videoHeight.value ? videoWidth.value / videoHeight.value : 16 / 9,
  }
  return ratioMap[value] || ratioMap.original
}

function getCropStageAspect(rect = cropStageRef.value?.getBoundingClientRect()) {
  return rect?.width && rect.height ? rect.width / rect.height : 16 / 9
}

function startCropDrag(event: PointerEvent) {
  cropDragStart.value = {
    clientX: event.clientX,
    clientY: event.clientY,
    x: cropBox.value.x,
    y: cropBox.value.y,
    width: cropBox.value.width,
    height: cropBox.value.height,
    mode: 'move',
  }
  window.addEventListener('pointermove', handleCropDragMove)
  window.addEventListener('pointerup', stopCropDrag)
}

function startCropResize(event: PointerEvent, handle: 'tl' | 'tr' | 'bl' | 'br') {
  cropDragStart.value = {
    clientX: event.clientX,
    clientY: event.clientY,
    x: cropBox.value.x,
    y: cropBox.value.y,
    width: cropBox.value.width,
    height: cropBox.value.height,
    mode: 'resize',
    handle,
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

  if (start.mode === 'resize') {
    const next = resizeCropBox(start, dx, dy)
    cropBox.value = next
    return
  }

  cropBox.value = {
    ...cropBox.value,
    x: Math.min(100 - cropBox.value.width, Math.max(0, start.x + dx)),
    y: Math.min(100 - cropBox.value.height, Math.max(0, start.y + dy)),
  }
}

function resizeCropBox(
  start: NonNullable<typeof cropDragStart.value>,
  dx: number,
  dy: number,
) {
  const lockedBox = resizeLockedCropBox(start, dx, dy)
  if (lockedBox) return lockedBox

  let { x, y, width, height } = start

  if (start.handle?.includes('l')) {
    x = start.x + dx
    width = start.width - dx
  }
  if (start.handle?.includes('r')) {
    width = start.width + dx
  }
  if (start.handle?.includes('t')) {
    y = start.y + dy
    height = start.height - dy
  }
  if (start.handle?.includes('b')) {
    height = start.height + dy
  }

  return constrainCropBox({ x, y, width, height })
}

function resizeLockedCropBox(
  start: NonNullable<typeof cropDragStart.value>,
  dx: number,
  dy: number,
) {
  const targetRatio = getCropTargetRatio()
  if (!targetRatio || !start.handle) return null

  const boxHeightPerWidth = getCropStageAspect() / targetRatio
  const widthByPointer = start.handle.includes('l') ? start.width - dx : start.width + dx
  const heightByPointer = start.handle.includes('t') ? start.height - dy : start.height + dy
  const widthByHeight = heightByPointer / boxHeightPerWidth
  const proposedWidth = Math.abs(widthByPointer - start.width) >= Math.abs(widthByHeight - start.width)
    ? widthByPointer
    : widthByHeight

  const anchorX = start.handle.includes('l') ? start.x + start.width : start.x
  const anchorY = start.handle.includes('t') ? start.y + start.height : start.y
  const maxWidthFromX = start.handle.includes('l') ? anchorX : 100 - anchorX
  const maxHeightFromY = start.handle.includes('t') ? anchorY : 100 - anchorY
  const maxWidth = Math.max(0, Math.min(maxWidthFromX, maxHeightFromY / boxHeightPerWidth))
  const minWidth = Math.min(maxWidth, Math.max(12, 12 / boxHeightPerWidth))
  const width = Math.min(maxWidth, Math.max(minWidth, proposedWidth))
  const height = width * boxHeightPerWidth

  return {
    x: start.handle.includes('l') ? anchorX - width : anchorX,
    y: start.handle.includes('t') ? anchorY - height : anchorY,
    width,
    height,
  }
}

function constrainCropBox(box: { x: number; y: number; width: number; height: number }) {
  const minSize = 12
  const width = Math.min(100, Math.max(minSize, box.width))
  const height = Math.min(100, Math.max(minSize, box.height))
  return {
    width,
    height,
    x: Math.min(100 - width, Math.max(0, box.x)),
    y: Math.min(100 - height, Math.max(0, box.y)),
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
  resetCompressionState()
  clearMergeFiles()
  revokeObjectUrl()
  await deleteStoredFile()
  emit('workspaceChange', false)
}

function requestResetEditor() {
  if (isProcessing.value) {
    pendingTaskAction.value = 'reset'
    return
  }
  void resetEditor()
}

function closeTaskGuard() {
  pendingTaskAction.value = null
}

async function confirmTaskGuard() {
  const action = pendingTaskAction.value
  pendingTaskAction.value = null
  stopCurrentTask()

  if (action === 'reset') {
    await resetEditor()
    return
  }

  if (action === 'replace') {
    resetCompressionState()
    openReplacePickerDirect()
  }
}

function stopCurrentTask() {
  taskToken += 1

  clearProcessingTimeout()

  if (activeProcessingFfmpeg && activeProcessingFfmpeg !== ffmpegInstance) {
    activeProcessingFfmpeg.terminate()
  }

  if (ffmpegInstance) {
    ffmpegInstance.terminate()
    ffmpegInstance = null
  }
  activeProcessingFfmpeg = null

  isProcessing.value = false
  compressionStatus.value = ''
}

function clearProcessingTimeout() {
  if (processTimerId !== null) {
    window.clearTimeout(processTimerId)
    processTimerId = null
  }
}

function startProcessingTimeout(totalBytes: number, currentTaskToken: number) {
  clearProcessingTimeout()
  const timeoutMs = getProcessingTimeoutMs(totalBytes)
  processTimerId = window.setTimeout(() => {
    if (currentTaskToken !== taskToken) return
    stopCurrentTask()
    showWindowsCompressPrompt.value = true
    compressionError.value = '处理失败：当前任务超时，已终止处理。请下载客户端进行处理。'
  }, timeoutMs)
}

function getProcessingTimeoutMs(totalBytes: number) {
  const sizeMb = totalBytes / MB
  const minutes = sizeMb <= 100
    ? 10
    : sizeMb <= 200
      ? 15
      : sizeMb <= 300
        ? 20
        : sizeMb <= 400
          ? 25
          : 30
  return minutes * 60 * 1000
}

function finishProcessingTask(currentTaskToken: number) {
  if (currentTaskToken !== taskToken) return
  clearProcessingTimeout()
  activeProcessingFfmpeg = null
  isProcessing.value = false
}

async function startProcess() {
  if (activeTool.value === 'compress') {
    await compressCurrentVideo()
    return
  }
  if (activeTool.value === 'convert') {
    await convertCurrentVideo()
    return
  }
  if (activeTool.value === 'gif-maker' || activeTool.value === 'video-to-gif') {
    await createGifFromCurrentVideo()
    return
  }
  if (activeTool.value === 'mp3-converter' || activeTool.value === 'mp4-to-mp3') {
    await convertCurrentMediaToAudio()
    return
  }
  if (activeTool.value === 'merge') {
    await mergeCurrentVideos()
    return
  }
  if (activeTool.value === 'trim') {
    await trimCurrentVideo()
    return
  }
  if (activeTool.value === 'crop') {
    await cropCurrentVideo()
    return
  }
}

async function compressCurrentVideo() {
  if (!uploadedFile.value) return
  resetCompressionState()

  if (isAudio.value) {
    compressionError.value = '视频压缩仅支持视频文件，请上传视频后再处理。'
    return
  }

  const videoDuration = await ensureMediaDuration()
  if (!videoDuration) {
    compressionError.value = '暂时无法读取视频时长，请更换文件后重试。'
    return
  }

  if (uploadedFile.value.size > MAX_WEB_COMPRESS_SIZE || videoDuration > MAX_WEB_COMPRESS_DURATION) {
    showWindowsCompressPrompt.value = true
    compressionError.value = '当前文件超过网页端处理范围，请下载 Windows 桌面端处理。'
    return
  }

  isProcessing.value = true
  const currentTaskToken = ++taskToken
  startProcessingTimeout(uploadedFile.value.size, currentTaskToken)
  compressionStatus.value = '正在加载压缩引擎...'

  const inputName = `input.${getFileExtension(uploadedFile.value.name)}`
  const outputName = 'output.mp4'
  let ffmpeg: typeof ffmpegInstance = null

  try {
    ffmpeg = await getFfmpeg()
    ensureCurrentTask(currentTaskToken)
    const { fetchFile } = await import('@ffmpeg/util')
    ensureCurrentTask(currentTaskToken)
    compressionStatus.value = '正在读取文件...'
    await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(outputName)])
    ensureCurrentTask(currentTaskToken)
    await ffmpeg.writeFile(inputName, await fetchFile(uploadedFile.value))
    ensureCurrentTask(currentTaskToken)

    compressionStatus.value = '正在压缩视频...'
    const args = createCompressionArgs(inputName, outputName, uploadedFile.value, videoDuration)
    let exitCode = await ffmpeg.exec(args)
    ensureCurrentTask(currentTaskToken)
    if (exitCode !== 0) {
      exitCode = await ffmpeg.exec(createFallbackCompressionArgs(inputName, outputName, uploadedFile.value, videoDuration))
      ensureCurrentTask(currentTaskToken)
    }
    if (exitCode !== 0) throw new Error('FFmpeg compression failed')

    compressionStatus.value = '正在生成下载文件...'
    const data = await ffmpeg.readFile(outputName)
    ensureCurrentTask(currentTaskToken)
    const bytes = data instanceof Uint8Array ? data : new TextEncoder().encode(data)
    const blob = new Blob([bytes], { type: 'video/mp4' })

    compressionOutputUrl.value = URL.createObjectURL(blob)
    compressionOutputName.value = getCompressedFileName(uploadedFile.value.name)
    compressionOutputSize.value = blob.size
    compressionProgress.value = 100
    compressionStatus.value = '压缩完成，可下载结果文件。'

    await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(outputName)])
  } catch (error) {
    const reason = error instanceof Error ? error.message : String(error)
    if (reason === 'Task cancelled' || currentTaskToken !== taskToken) return
    console.error('[video-compress] failed', reason, compressionDebugLog.value.slice(-50).join('\n'))
    compressionError.value = '网页端压缩失败，请重试或下载 Windows 桌面端处理。'
    compressionStatus.value = ''
    if (ffmpeg) await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(outputName)])
  } finally {
    finishProcessingTask(currentTaskToken)
  }
}

async function convertCurrentVideo() {
  if (!uploadedFile.value) return
  resetCompressionState()

  if (isAudio.value) {
    compressionError.value = '格式转换仅支持视频文件，请上传视频后再处理。'
    return
  }

  const videoDuration = await ensureMediaDuration()
  if (!videoDuration) {
    compressionError.value = '暂时无法读取视频时长，请更换文件后重试。'
    return
  }

  if (uploadedFile.value.size > MAX_WEB_COMPRESS_SIZE || videoDuration > MAX_WEB_COMPRESS_DURATION) {
    showWindowsCompressPrompt.value = true
    compressionError.value = '当前文件超过网页端处理范围，请下载 Windows 桌面端处理。'
    return
  }

  isProcessing.value = true
  const currentTaskToken = ++taskToken
  startProcessingTimeout(uploadedFile.value.size, currentTaskToken)
  const format = getSelectedConvertFormat()
  const inputName = `input.${getFileExtension(uploadedFile.value.name)}`
  const outputName = `output.${getConvertOutputExtension(format)}`
  let ffmpeg: typeof ffmpegInstance = null

  compressionStatus.value = '正在加载转换引擎...'

  try {
    ffmpeg = await getFfmpeg()
    ensureCurrentTask(currentTaskToken)
    const { fetchFile } = await import('@ffmpeg/util')
    ensureCurrentTask(currentTaskToken)
    compressionStatus.value = '正在读取文件...'
    await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(outputName)])
    ensureCurrentTask(currentTaskToken)
    await ffmpeg.writeFile(inputName, await fetchFile(uploadedFile.value))
    ensureCurrentTask(currentTaskToken)

    compressionStatus.value = `正在转换为 ${format}...`
    let exitCode = await ffmpeg.exec(createConvertArgs(inputName, outputName, uploadedFile.value, false))
    ensureCurrentTask(currentTaskToken)
    if (exitCode !== 0) {
      await Promise.allSettled([ffmpeg.deleteFile(outputName)])
      compressionStatus.value = '正在使用兼容模式转换...'
      exitCode = await ffmpeg.exec(createConvertArgs(inputName, outputName, uploadedFile.value, true))
      ensureCurrentTask(currentTaskToken)
    }
    if (exitCode !== 0) throw new Error('FFmpeg conversion failed')

    compressionStatus.value = '正在生成下载文件...'
    const data = await ffmpeg.readFile(outputName)
    ensureCurrentTask(currentTaskToken)
    const bytes = data instanceof Uint8Array ? data : new TextEncoder().encode(data)
    const blob = new Blob([bytes], { type: getConvertMimeType(format) })

    compressionOutputUrl.value = URL.createObjectURL(blob)
    compressionOutputName.value = getConvertedFileName(uploadedFile.value.name, format)
    compressionOutputSize.value = blob.size
    compressionProgress.value = 100
    compressionStatus.value = '转换完成，可下载结果文件。'

    await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(outputName)])
  } catch (error) {
    const reason = error instanceof Error ? error.message : String(error)
    if (reason === 'Task cancelled' || currentTaskToken !== taskToken) return
    console.error('[video-convert] failed', reason, compressionDebugLog.value.slice(-50).join('\n'))
    compressionError.value = '网页端格式转换失败，请重试或下载 Windows 桌面端处理。'
    compressionStatus.value = ''
    if (ffmpeg) await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(outputName)])
  } finally {
    finishProcessingTask(currentTaskToken)
  }
}

async function createGifFromCurrentVideo() {
  if (!uploadedFile.value) return
  resetCompressionState()

  if (isAudio.value) {
    compressionError.value = 'GIF 制作仅支持视频文件，请上传视频后再处理。'
    return
  }

  const videoDuration = await ensureMediaDuration()
  if (!videoDuration) {
    compressionError.value = '暂时无法读取视频时长，请更换文件后重试。'
    return
  }

  if (uploadedFile.value.size > MAX_WEB_COMPRESS_SIZE || videoDuration > MAX_WEB_COMPRESS_DURATION) {
    showWindowsCompressPrompt.value = true
    compressionError.value = '当前文件超过网页端处理范围，请下载 Windows 桌面端处理。'
    return
  }

  isProcessing.value = true
  const currentTaskToken = ++taskToken
  startProcessingTimeout(uploadedFile.value.size, currentTaskToken)
  const inputName = `input.${getFileExtension(uploadedFile.value.name)}`
  const outputName = 'output.gif'
  let ffmpeg: typeof ffmpegInstance = null

  compressionStatus.value = '正在加载 GIF 引擎...'

  try {
    ffmpeg = await getFfmpeg()
    ensureCurrentTask(currentTaskToken)
    const { fetchFile } = await import('@ffmpeg/util')
    ensureCurrentTask(currentTaskToken)
    compressionStatus.value = '正在读取文件...'
    await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(outputName)])
    ensureCurrentTask(currentTaskToken)
    await ffmpeg.writeFile(inputName, await fetchFile(uploadedFile.value))
    ensureCurrentTask(currentTaskToken)

    compressionStatus.value = '正在生成 GIF...'
    let exitCode = await ffmpeg.exec(createGifArgs(inputName, outputName, videoDuration))
    ensureCurrentTask(currentTaskToken)
    if (exitCode !== 0) {
      await Promise.allSettled([ffmpeg.deleteFile(outputName)])
      compressionStatus.value = '正在使用兼容模式生成 GIF...'
      exitCode = await ffmpeg.exec(createGifFallbackArgs(inputName, outputName, videoDuration))
      ensureCurrentTask(currentTaskToken)
    }
    if (exitCode !== 0) throw new Error('FFmpeg gif failed')

    compressionStatus.value = '正在生成下载文件...'
    const data = await ffmpeg.readFile(outputName)
    ensureCurrentTask(currentTaskToken)
    const bytes = data instanceof Uint8Array ? data : new TextEncoder().encode(data)
    const blob = new Blob([bytes], { type: 'image/gif' })

    compressionOutputUrl.value = URL.createObjectURL(blob)
    compressionOutputName.value = getGifOutputFileName(uploadedFile.value.name)
    compressionOutputSize.value = blob.size
    compressionProgress.value = 100
    compressionStatus.value = 'GIF 制作完成，可下载结果文件。'

    await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(outputName)])
  } catch (error) {
    const reason = error instanceof Error ? error.message : String(error)
    if (reason === 'Task cancelled' || currentTaskToken !== taskToken) return
    console.error('[video-gif] failed', reason, compressionDebugLog.value.slice(-50).join('\n'))
    compressionError.value = '网页端 GIF 制作失败，请重试或下载 Windows 桌面端处理。'
    compressionStatus.value = ''
    if (ffmpeg) await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(outputName)])
  } finally {
    finishProcessingTask(currentTaskToken)
  }
}

async function convertCurrentMediaToAudio() {
  if (!uploadedFile.value) return
  resetCompressionState()

  const mediaDuration = await ensureMediaDuration()
  if (!mediaDuration) {
    compressionError.value = '暂时无法读取媒体时长，请更换文件后重试。'
    return
  }

  if (uploadedFile.value.size > MAX_WEB_COMPRESS_SIZE || mediaDuration > MAX_WEB_COMPRESS_DURATION) {
    showWindowsCompressPrompt.value = true
    compressionError.value = '当前文件超过网页端处理范围，请下载 Windows 桌面端处理。'
    return
  }

  if (!isAudio.value && !fileType.value.startsWith('video/')) {
    compressionError.value = '音频转换仅支持音频或视频文件，请重新上传后再处理。'
    return
  }

  isProcessing.value = true
  const currentTaskToken = ++taskToken
  startProcessingTimeout(uploadedFile.value.size, currentTaskToken)
  const format = getAudioOutputFormat()
  const inputName = `input.${getFileExtension(uploadedFile.value.name)}`
  const outputName = `output.${getAudioOutputExtension(format)}`
  let ffmpeg: typeof ffmpegInstance = null

  compressionStatus.value = '正在加载音频引擎...'

  try {
    ffmpeg = await getFfmpeg()
    ensureCurrentTask(currentTaskToken)
    const { fetchFile } = await import('@ffmpeg/util')
    ensureCurrentTask(currentTaskToken)
    compressionStatus.value = '正在读取文件...'
    await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(outputName)])
    ensureCurrentTask(currentTaskToken)
    await ffmpeg.writeFile(inputName, await fetchFile(uploadedFile.value))
    ensureCurrentTask(currentTaskToken)

    compressionStatus.value = `正在转换为 ${format}...`
    let exitCode = await ffmpeg.exec(createAudioArgs(inputName, outputName, format, false))
    ensureCurrentTask(currentTaskToken)
    if (exitCode !== 0) {
      await Promise.allSettled([ffmpeg.deleteFile(outputName)])
      compressionStatus.value = '正在使用兼容模式转换音频...'
      exitCode = await ffmpeg.exec(createAudioArgs(inputName, outputName, format, true))
      ensureCurrentTask(currentTaskToken)
    }
    if (exitCode !== 0) throw new Error('FFmpeg audio failed')

    compressionStatus.value = '正在生成下载文件...'
    const data = await ffmpeg.readFile(outputName)
    ensureCurrentTask(currentTaskToken)
    const bytes = data instanceof Uint8Array ? data : new TextEncoder().encode(data)
    const blob = new Blob([bytes], { type: getAudioMimeType(format) })

    compressionOutputUrl.value = URL.createObjectURL(blob)
    compressionOutputName.value = getAudioOutputFileName(uploadedFile.value.name, format)
    compressionOutputSize.value = blob.size
    compressionProgress.value = 100
    compressionStatus.value = '音频转换完成，可下载结果文件。'

    await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(outputName)])
  } catch (error) {
    const reason = error instanceof Error ? error.message : String(error)
    if (reason === 'Task cancelled' || currentTaskToken !== taskToken) return
    console.error('[audio-convert] failed', reason, compressionDebugLog.value.slice(-50).join('\n'))
    compressionError.value = '网页端音频转换失败，请重试或下载 Windows 桌面端处理。'
    compressionStatus.value = ''
    if (ffmpeg) await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(outputName)])
  } finally {
    finishProcessingTask(currentTaskToken)
  }
}

async function mergeCurrentVideos() {
  resetCompressionState()
  const orderedItems = getOrderedMergeItems()
  if (orderedItems.length < 2) {
    compressionError.value = '请至少添加 2 个视频文件后再合并。'
    return
  }

  if (orderedItems.some((item) => !item.file.type.startsWith('video/'))) {
    compressionError.value = '视频合并仅支持视频文件，请移除非视频文件后再处理。'
    return
  }

  const mediaInfos = await Promise.all(orderedItems.map((item) => getMergeMediaInfo(item.file, item.url)))
  const totalDuration = mediaInfos.reduce((sum, itemInfo) => sum + itemInfo.duration, 0)
  const totalSize = orderedItems.reduce((sum, item) => sum + item.file.size, 0)
  if (!totalDuration) {
    compressionError.value = '暂时无法读取视频时长，请更换文件后重试。'
    return
  }

  if (totalSize > MAX_WEB_COMPRESS_SIZE || totalDuration > MAX_WEB_COMPRESS_DURATION) {
    showWindowsCompressPrompt.value = true
    compressionError.value = '当前文件超过网页端处理范围，请下载 Windows 桌面端处理。'
    return
  }

  isProcessing.value = true
  const currentTaskToken = ++taskToken
  startProcessingTimeout(totalSize, currentTaskToken)
  const format = getMergeOutputFormat()
  const useCopyMode = shouldUseMergeCopy(orderedItems, mediaInfos, format)
  const inputNames = orderedItems.map((item, index) => `merge-${index}.${getFileExtension(item.file.name)}`)
  const listName = 'merge-list.txt'
  const outputName = `merged-output.${getMergeOutputExtension(format)}`
  let ffmpeg: typeof ffmpegInstance = null

  compressionStatus.value = useCopyMode ? '正在加载快速合并引擎...' : '正在加载兼容合并引擎...'

  try {
    ffmpeg = await getFfmpeg()
    ensureCurrentTask(currentTaskToken)
    const { fetchFile } = await import('@ffmpeg/util')
    ensureCurrentTask(currentTaskToken)
    compressionStatus.value = '正在读取合并文件...'
    await Promise.allSettled([...inputNames, listName, outputName].map((name) => ffmpeg!.deleteFile(name)))

    for (const [index, item] of orderedItems.entries()) {
      ensureCurrentTask(currentTaskToken)
      await ffmpeg.writeFile(inputNames[index], await fetchFile(item.file))
    }
    await ffmpeg.writeFile(listName, new TextEncoder().encode(inputNames.map((name) => `file '${name}'`).join('\n')))
    ensureCurrentTask(currentTaskToken)

    compressionStatus.value = useCopyMode
      ? '正在快速合并视频...'
      : '检测到素材分辨率或参数不同，正在重编码合并...'
    let exitCode = await ffmpeg.exec(createMergeArgs({
      listName,
      outputName,
      format,
      mode: useCopyMode ? 'copy' : 'reencode',
      items: orderedItems,
      mediaInfos,
    }))
    ensureCurrentTask(currentTaskToken)
    if (exitCode !== 0 && useCopyMode) {
      await Promise.allSettled([ffmpeg.deleteFile(outputName)])
      compressionStatus.value = '快速合并失败，正在切换为兼容重编码...'
      exitCode = await ffmpeg.exec(createMergeArgs({
        listName,
        outputName,
        format,
        mode: 'reencode',
        items: orderedItems,
        mediaInfos,
      }))
      ensureCurrentTask(currentTaskToken)
    }
    if (exitCode !== 0) throw new Error('FFmpeg merge failed')

    compressionStatus.value = '正在生成下载文件...'
    const data = await ffmpeg.readFile(outputName)
    ensureCurrentTask(currentTaskToken)
    const bytes = data instanceof Uint8Array ? data : new TextEncoder().encode(data)
    const blob = new Blob([bytes], { type: getMergeMimeType(format) })
    compressionOutputUrl.value = URL.createObjectURL(blob)
    compressionOutputName.value = getMergedOutputFileName(uploadedFile.value?.name || orderedItems[0].file.name, format)
    compressionOutputSize.value = blob.size
    compressionProgress.value = 100
    compressionStatus.value = '合并完成，可下载结果文件。'

    await Promise.allSettled([...inputNames, listName, outputName].map((name) => ffmpeg!.deleteFile(name)))
  } catch (error) {
    const reason = error instanceof Error ? error.message : String(error)
    if (reason === 'Task cancelled' || currentTaskToken !== taskToken) return
    console.error('[video-merge] failed', reason, compressionDebugLog.value.slice(-50).join('\n'))
    compressionError.value = '网页端视频合并失败，请重试或下载 Windows 桌面端处理。'
    compressionStatus.value = ''
    if (ffmpeg) await Promise.allSettled([...inputNames, listName, outputName].map((name) => ffmpeg!.deleteFile(name)))
  } finally {
    finishProcessingTask(currentTaskToken)
  }
}

async function trimCurrentVideo() {
  if (!uploadedFile.value) return
  resetCompressionState()

  if (isAudio.value) {
    compressionError.value = '视频剪切仅支持视频文件，请上传视频后再处理。'
    return
  }

  const videoDuration = await ensureMediaDuration()
  if (!videoDuration) {
    compressionError.value = '暂时无法读取视频时长，请更换文件后重试。'
    return
  }

  if (uploadedFile.value.size > MAX_WEB_COMPRESS_SIZE || videoDuration > MAX_WEB_COMPRESS_DURATION) {
    showWindowsCompressPrompt.value = true
    compressionError.value = '当前文件超过网页端处理范围，请下载 Windows 桌面端处理。'
    return
  }

  const range = getTrimRange()
  if (range.end - range.start < 0.2) {
    compressionError.value = '剪切片段太短，请重新设置起点和终点。'
    return
  }

  await runSingleVideoTask({
    currentTaskLabel: '剪切',
    loadingStatus: '正在加载剪切引擎...',
    processingStatus: '正在剪切视频...',
    doneStatus: '剪切完成，可下载结果文件。',
    failedMessage: '网页端视频剪切失败，请重试或下载 Windows 桌面端处理。',
    outputName: 'trimmed-output.mp4',
    outputFileName: getTrimmedOutputFileName(uploadedFile.value.name),
    totalBytes: uploadedFile.value.size,
    args: (inputName, outputName) => createTrimArgs(inputName, outputName, range),
  })
}

async function cropCurrentVideo() {
  if (!uploadedFile.value) return
  resetCompressionState()

  if (isAudio.value) {
    compressionError.value = '视频裁剪仅支持视频文件，请上传视频后再处理。'
    return
  }

  const videoDuration = await ensureMediaDuration()
  if (!videoDuration) {
    compressionError.value = '暂时无法读取视频时长，请更换文件后重试。'
    return
  }

  if (uploadedFile.value.size > MAX_WEB_COMPRESS_SIZE || videoDuration > MAX_WEB_COMPRESS_DURATION) {
    showWindowsCompressPrompt.value = true
    compressionError.value = '当前文件超过网页端处理范围，请下载 Windows 桌面端处理。'
    return
  }

  if (!videoWidth.value || !videoHeight.value) {
    compressionError.value = '暂时无法读取视频尺寸，请等待预览加载后再处理。'
    return
  }

  await runSingleVideoTask({
    currentTaskLabel: '裁剪',
    loadingStatus: '正在加载裁剪引擎...',
    processingStatus: '正在裁剪视频...',
    doneStatus: '裁剪完成，可下载结果文件。',
    failedMessage: '网页端视频裁剪失败，请重试或下载 Windows 桌面端处理。',
    outputName: 'cropped-output.mp4',
    outputFileName: getCroppedOutputFileName(uploadedFile.value.name),
    totalBytes: uploadedFile.value.size,
    args: (inputName, outputName) => createCropArgs(inputName, outputName),
  })
}

async function runSingleVideoTask(config: {
  currentTaskLabel: string
  loadingStatus: string
  processingStatus: string
  doneStatus: string
  failedMessage: string
  outputName: string
  outputFileName: string
  totalBytes: number
  args: (inputName: string, outputName: string) => string[]
}) {
  if (!uploadedFile.value) return
  isProcessing.value = true
  const currentTaskToken = ++taskToken
  startProcessingTimeout(config.totalBytes, currentTaskToken)
  const inputName = `input.${getFileExtension(uploadedFile.value.name)}`
  let ffmpeg: typeof ffmpegInstance = null

  compressionStatus.value = config.loadingStatus

  try {
    ffmpeg = await getFfmpeg()
    ensureCurrentTask(currentTaskToken)
    const { fetchFile } = await import('@ffmpeg/util')
    ensureCurrentTask(currentTaskToken)
    compressionStatus.value = '正在读取文件...'
    await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(config.outputName)])
    ensureCurrentTask(currentTaskToken)
    await ffmpeg.writeFile(inputName, await fetchFile(uploadedFile.value))
    ensureCurrentTask(currentTaskToken)

    compressionStatus.value = config.processingStatus
    const exitCode = await ffmpeg.exec(config.args(inputName, config.outputName))
    ensureCurrentTask(currentTaskToken)
    if (exitCode !== 0) throw new Error(`FFmpeg ${config.currentTaskLabel} failed`)

    compressionStatus.value = '正在生成下载文件...'
    const data = await ffmpeg.readFile(config.outputName)
    ensureCurrentTask(currentTaskToken)
    const bytes = data instanceof Uint8Array ? data : new TextEncoder().encode(data)
    const blob = new Blob([bytes], { type: 'video/mp4' })
    compressionOutputUrl.value = URL.createObjectURL(blob)
    compressionOutputName.value = config.outputFileName
    compressionOutputSize.value = blob.size
    compressionProgress.value = 100
    compressionStatus.value = config.doneStatus

    await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(config.outputName)])
  } catch (error) {
    const reason = error instanceof Error ? error.message : String(error)
    if (reason === 'Task cancelled' || currentTaskToken !== taskToken) return
    console.error(`[video-${config.currentTaskLabel}] failed`, reason, compressionDebugLog.value.slice(-50).join('\n'))
    compressionError.value = config.failedMessage
    compressionStatus.value = ''
    if (ffmpeg) await Promise.allSettled([ffmpeg.deleteFile(inputName), ffmpeg.deleteFile(config.outputName)])
  } finally {
    finishProcessingTask(currentTaskToken)
  }
}

function ensureCurrentTask(currentTaskToken: number) {
  if (currentTaskToken !== taskToken) throw new Error('Task cancelled')
}

async function getFfmpeg() {
  if (ffmpegInstance?.loaded) {
    activeProcessingFfmpeg = ffmpegInstance
    return ffmpegInstance
  }
  const { FFmpeg } = await import('@ffmpeg/ffmpeg')
  const ffmpeg = new FFmpeg()
  activeProcessingFfmpeg = ffmpeg
  ffmpeg.on('progress', ({ progress }) => {
    compressionProgress.value = Math.min(99, Math.max(1, Math.round(progress * 100)))
  })
  ffmpeg.on('log', ({ type, message }) => {
    const nextMessage = `[${type}] ${message}`
    compressionDebugLog.value = [...compressionDebugLog.value.slice(-80), nextMessage]
    console.debug('[ffmpeg]', nextMessage)
  })
  await loadFfmpegWithTimeout(ffmpeg)
  ffmpegInstance = ffmpeg
  return ffmpegInstance
}

async function loadFfmpegWithTimeout(ffmpeg: NonNullable<typeof ffmpegInstance>) {
  const { toBlobURL } = await import('@ffmpeg/util')
  const timeout = new Promise<never>((_, reject) => {
    window.setTimeout(() => reject(new Error('FFmpeg load timed out')), FFMPEG_LOAD_TIMEOUT)
  })
  await Promise.race([
    ffmpeg.load({
      coreURL: await toBlobURL(`${FFMPEG_CORE_BASE_URL}/ffmpeg-core.js`, 'text/javascript'),
      wasmURL: await toBlobURL(`${FFMPEG_CORE_BASE_URL}/ffmpeg-core.wasm`, 'application/wasm'),
    }),
    timeout,
  ])
}

async function ensureMediaDuration() {
  if (duration.value) return duration.value
  const mediaDuration = mediaRef.value?.duration
  if (mediaDuration && Number.isFinite(mediaDuration)) {
    duration.value = mediaDuration
    return mediaDuration
  }
  if (!objectUrl.value) return 0
  return new Promise<number>((resolve) => {
    const media = document.createElement(isAudio.value ? 'audio' : 'video')
    media.preload = 'metadata'
    media.onloadedmetadata = () => {
      const nextDuration = media.duration || 0
      duration.value = nextDuration
      media.removeAttribute('src')
      media.load()
      resolve(nextDuration)
    }
    media.onerror = () => resolve(0)
    media.src = objectUrl.value
  })
}

function getGifSegmentDuration(videoDuration: number) {
  if (gifRangeMode.value === 'custom') {
    const start = clamp(gifStartSecond.value || 0, 0, Math.max(0, videoDuration - 0.1))
    const remaining = Math.max(0.5, videoDuration - start)
    return { start, duration: Math.min(remaining, Math.max(1, gifSeconds.value || 1)) }
  }
  return { start: 0, duration: Math.min(Math.max(1, gifSeconds.value || 1), videoDuration) }
}

function getGifScaleArgs() {
  const sizeMap: Record<string, string> = {
    原始: '-1',
    '400': '400',
    '280': '280',
    '180': '180',
  }
  const width = sizeMap[gifSize.value] || '-1'
  return width === '-1' ? 'scale=trunc(iw/2)*2:trunc(ih/2)*2' : `scale=${width}:-1:flags=lanczos`
}

function getGifSpeedMultiplier() {
  const speedMap: Record<string, number> = {
    原始: 1,
    '0.5倍': 0.5,
    '0.75倍': 0.75,
    '1.5倍': 1.5,
    '2倍': 2,
  }
  return speedMap[gifSpeed.value] || 1
}

function createGifArgs(inputName: string, outputName: string, videoDuration: number) {
  const { start, duration: segmentDuration } = getGifSegmentDuration(videoDuration)
  const fps = Number(gifFrameRate.value) || 10
  const speed = getGifSpeedMultiplier()
  const filterParts = [
    `fps=${fps}`,
    getGifScaleArgs(),
  ]
  if (speed !== 1) filterParts.push(`setpts=PTS/${speed}`)
  const filter = filterParts.join(',')
  const paletteFilter = `[0:v]${filter},split[a][b];[a]palettegen[p];[b][p]paletteuse`
  return [
    '-y',
    '-ss',
    `${start}`,
    '-t',
    `${segmentDuration}`,
    '-i',
    inputName,
    '-filter_complex',
    paletteFilter,
    outputName,
  ]
}

function createGifFallbackArgs(inputName: string, outputName: string, videoDuration: number) {
  const { start, duration: segmentDuration } = getGifSegmentDuration(videoDuration)
  const fps = Number(gifFrameRate.value) || 10
  const speed = getGifSpeedMultiplier()
  const width = gifSize.value === '原始' ? '320' : gifSize.value
  const scaleFilter = gifSize.value === '原始'
    ? 'scale=trunc(iw/2)*2:trunc(ih/2)*2'
    : `scale=${width}:-1:flags=lanczos`
  const filterParts = [`fps=${Math.max(6, Math.min(15, fps))}`, scaleFilter]
  if (speed !== 1) filterParts.push(`setpts=PTS/${speed}`)
  const filter = filterParts.join(',')
  return [
    '-y',
    '-ss',
    `${start}`,
    '-t',
    `${segmentDuration}`,
    '-i',
    inputName,
    '-vf',
    filter,
    '-loop',
    '0',
    outputName,
  ]
}

function createAudioArgs(inputName: string, outputName: string, format: string, useFallback: boolean) {
  const quality = getAudioQuality()
  const bitrateMap: Record<string, Record<string, string>> = {
    标准: { MP3: '128k', M4A: '128k', AAC: '128k' },
    高质量: { MP3: '192k', M4A: '192k', AAC: '192k' },
    无损: { MP3: '320k', M4A: 'alac', AAC: '320k' },
  }
  const bitrate = bitrateMap[quality]?.[format] || '128k'

  if (format === 'WAV') {
    return [
      '-y',
      '-i',
      inputName,
      '-vn',
      '-map',
      '0:a:0',
      '-c:a',
      'pcm_s16le',
      outputName,
    ]
  }

  const codecMap: Record<string, string[]> = {
    MP3: useFallback ? ['-c:a', 'mp3'] : ['-c:a', 'libmp3lame'],
    M4A: useFallback ? ['-c:a', 'aac'] : (bitrate === 'alac' ? ['-c:a', 'alac'] : ['-c:a', 'aac']),
    AAC: ['-c:a', 'aac'],
  }
  const containerMap: Record<string, string> = {
    MP3: 'mp3',
    M4A: 'ipod',
    AAC: 'adts',
  }

  const args = [
    '-y',
    '-i',
    inputName,
    '-vn',
    '-map',
    '0:a:0',
      ...(codecMap[format] || codecMap.MP3),
  ]

  if (format === 'MP3') {
    args.push('-b:a', String(bitrate))
  } else if (format === 'M4A') {
    if (bitrate === 'alac') {
      args.push('-sample_fmt', 's16')
    } else {
      args.push('-b:a', String(bitrate))
    }
  } else {
    args.push('-b:a', String(bitrate))
  }

  args.push('-f', containerMap[format] || 'mp3', outputName)
  return args
}

function createMergeArgs(config: {
  listName: string
  outputName: string
  format: string
  mode: 'copy' | 'reencode'
  items: MergeFileItem[]
  mediaInfos: MergeMediaInfo[]
}) {
  if (config.mode === 'copy') {
    return [
      '-y',
      '-f',
      'concat',
      '-safe',
      '0',
      '-i',
      config.listName,
      '-c',
      'copy',
      ...getMergeContainerArgs(config.format, true),
      config.outputName,
    ]
  }

  return createMergeReencodeArgs(config.outputName, config.format, config.items, config.mediaInfos)
}

function createMergeReencodeArgs(outputName: string, format: string, items: MergeFileItem[], mediaInfos: MergeMediaInfo[]) {
  const { width, height } = getMergeTargetSize(mediaInfos)
  const filters: string[] = []
  const concatInputs: string[] = []

  mediaInfos.forEach((info, index) => {
    filters.push(
      `[${index}:v]scale=${width}:${height}:force_original_aspect_ratio=decrease,pad=${width}:${height}:(ow-iw)/2:(oh-ih)/2,setsar=1,format=yuv420p[v${index}]`,
    )

    if (info.hasAudio) {
      filters.push(`[${index}:a]aresample=48000,aformat=channel_layouts=stereo,asetpts=PTS-STARTPTS[a${index}]`)
    } else {
      filters.push(`anullsrc=r=48000:cl=stereo:d=${Math.max(0.1, info.duration)},asetpts=PTS-STARTPTS[a${index}]`)
    }

    concatInputs.push(`[v${index}][a${index}]`)
  })

  filters.push(`${concatInputs.join('')}concat=n=${mediaInfos.length}:v=1:a=1[outv][outa]`)

  const args = ['-y']
  items.forEach((item, index) => {
    args.push('-i', `merge-${index}.${getFileExtension(item.file.name)}`)
  })
  args.push('-filter_complex', filters.join(';'), '-map', '[outv]', '-map', '[outa]')

  if (format === 'WEBM') {
    args.push('-c:v', 'libvpx-vp9', '-b:v', '0', '-crf', '32', '-c:a', 'libopus', '-b:a', '96k')
  } else {
    args.push('-c:v', 'libx264', '-preset', 'veryfast', '-crf', '23', '-pix_fmt', 'yuv420p', '-c:a', 'aac', '-b:a', '128k')
  }

  args.push(...getMergeContainerArgs(format, false), outputName)
  return args
}

function getMergeContainerArgs(format: string, useCopy: boolean) {
  if (format === 'MOV') return ['-f', 'mov']
  if (format === 'WEBM') return ['-f', 'webm']
  return useCopy ? ['-f', 'mp4'] : ['-movflags', 'faststart', '-f', 'mp4']
}

function shouldUseMergeCopy(items: MergeFileItem[], mediaInfos: MergeMediaInfo[], format: string) {
  if (format !== 'MP4') return false
  const firstItem = items[0]
  const firstInfo = mediaInfos[0]
  const firstExtension = getFileExtension(firstItem.file.name)
  if (!firstInfo?.width || !firstInfo?.height || firstExtension !== 'mp4') return false

  return items.every((item, index) => {
    const info = mediaInfos[index]
    return getFileExtension(item.file.name) === firstExtension
      && item.file.type === firstItem.file.type
      && info.width === firstInfo.width
      && info.height === firstInfo.height
      && info.hasAudio === firstInfo.hasAudio
  })
}

function getMergeTargetSize(mediaInfos: MergeMediaInfo[]) {
  const width = makeEven(Math.max(...mediaInfos.map((info) => info.width || 2)))
  const height = makeEven(Math.max(...mediaInfos.map((info) => info.height || 2)))
  return { width, height }
}

function createTrimArgs(inputName: string, outputName: string, range: { start: number; end: number }) {
  return [
    '-y',
    '-ss',
    `${range.start}`,
    '-t',
    `${Math.max(0.2, range.end - range.start)}`,
    '-i',
    inputName,
    '-map',
    '0:v:0',
    '-map',
    '0:a?',
    '-c:v',
    'libx264',
    '-preset',
    'veryfast',
    '-crf',
    '22',
    '-pix_fmt',
    'yuv420p',
    '-c:a',
    'aac',
    '-b:a',
    '128k',
    '-movflags',
    'faststart',
    outputName,
  ]
}

function createCropArgs(inputName: string, outputName: string) {
  const crop = getSourceCropRect()
  return [
    '-y',
    '-i',
    inputName,
    '-map',
    '0:v:0',
    '-map',
    '0:a?',
    '-vf',
    `crop=${crop.width}:${crop.height}:${crop.x}:${crop.y}`,
    '-c:v',
    'libx264',
    '-preset',
    'veryfast',
    '-crf',
    '22',
    '-pix_fmt',
    'yuv420p',
    '-c:a',
    'aac',
    '-b:a',
    '128k',
    '-movflags',
    'faststart',
    outputName,
  ]
}

function getSourceCropRect() {
  const width = Math.min(videoWidth.value, makeEven(Math.round((cropBox.value.width / 100) * videoWidth.value)))
  const height = Math.min(videoHeight.value, makeEven(Math.round((cropBox.value.height / 100) * videoHeight.value)))
  const x = makeEven(Math.min(videoWidth.value - width, Math.max(0, Math.round((cropBox.value.x / 100) * videoWidth.value))))
  const y = makeEven(Math.min(videoHeight.value - height, Math.max(0, Math.round((cropBox.value.y / 100) * videoHeight.value))))
  return {
    x,
    y,
    width: Math.max(2, width),
    height: Math.max(2, height),
  }
}

function makeEven(value: number) {
  return Math.max(2, Math.floor(value / 2) * 2)
}

function getMergeMediaInfo(file: File, sourceUrl?: string) {
  const url = sourceUrl || URL.createObjectURL(file)
  const shouldRevoke = !sourceUrl
  return new Promise<MergeMediaInfo>((resolve) => {
    const video = document.createElement('video')
    video.preload = 'metadata'
    video.onloadedmetadata = () => {
      const audioTracks = (video as HTMLVideoElement & { audioTracks?: { length: number } }).audioTracks
      const hasAudio = audioTracks ? audioTracks.length > 0 : true
      const mediaInfo = {
        duration: video.duration || 0,
        width: video.videoWidth || 0,
        height: video.videoHeight || 0,
        hasAudio,
      }
      video.removeAttribute('src')
      video.load()
      if (shouldRevoke) URL.revokeObjectURL(url)
      resolve(mediaInfo)
    }
    video.onerror = () => {
      if (shouldRevoke) URL.revokeObjectURL(url)
      resolve({ duration: 0, width: 0, height: 0, hasAudio: true })
    }
    video.src = url
  })
}

function clamp(value: number, min: number, max: number) {
  return Math.min(max, Math.max(min, value))
}

function createCompressionArgs(inputName: string, outputName: string, file: File, videoDuration: number) {
  const { videoKbps, audioKbps, maxRateKbps, bufferKbps, preset } = getCompressionBitrate(file, videoDuration)
  return [
    '-y',
    '-i',
    inputName,
    '-map',
    '0:v:0',
    '-map',
    '0:a?',
    '-c:v',
    'libx264',
    '-preset',
    preset,
    '-b:v',
    `${videoKbps}k`,
    '-maxrate',
    `${maxRateKbps}k`,
    '-bufsize',
    `${bufferKbps}k`,
    '-pix_fmt',
    'yuv420p',
    '-c:a',
    'aac',
    '-b:a',
    `${audioKbps}k`,
    '-movflags',
    'faststart',
    outputName,
  ]
}

function createFallbackCompressionArgs(inputName: string, outputName: string, file: File, videoDuration: number) {
  const { videoKbps, audioKbps } = getCompressionBitrate(file, videoDuration)
  return [
    '-y',
    '-i',
    inputName,
    '-map',
    '0:v:0',
    '-map',
    '0:a?',
    '-c:v',
    'mpeg4',
    '-b:v',
    `${videoKbps}k`,
    '-c:a',
    'aac',
    '-b:a',
    `${audioKbps}k`,
    outputName,
  ]
}

function getCompressionBitrate(file: File, videoDuration: number) {
  const selectedMode = selectedSettingOptions.value.compressMode || '智能压缩'
  const targetSizeMb = compressionSizeMode.value === 'percent'
    ? (file.size / MB) * (compressionPercent.value / 100)
    : compressionTargetSizeMb.value
  const targetBytes = Math.max(1 * MB, Math.min(file.size * 0.95, targetSizeMb * MB))
  const totalKbps = Math.max(128, Math.floor((targetBytes * 8) / Math.max(1, videoDuration) / 1000))
  const baseAudioKbps = totalKbps > 420 ? 96 : totalKbps > 220 ? 64 : 48
  const modeRatio: Record<string, number> = {
    智能压缩: 1,
    优先清晰: 1.15,
    最小体积: 0.82,
  }
  const presetMap: Record<string, string> = {
    智能压缩: 'veryfast',
    优先清晰: 'fast',
    最小体积: 'veryfast',
  }
  const audioKbps = selectedMode === '最小体积' ? 48 : baseAudioKbps
  const videoKbps = Math.max(80, Math.floor((totalKbps - audioKbps) * (modeRatio[selectedMode] || 1)))
  return {
    videoKbps,
    audioKbps,
    maxRateKbps: Math.floor(videoKbps * 1.35),
    bufferKbps: Math.floor(videoKbps * 2),
    preset: presetMap[selectedMode] || 'veryfast',
  }
}

function createConvertArgs(inputName: string, outputName: string, file: File, useFallback: boolean) {
  const format = getSelectedConvertFormat()
  const videoCodec = useFallback
    ? getFallbackConvertVideoCodec(format)
    : getPreferredConvertVideoCodec(format, file)
  return [
    '-y',
    '-i',
    inputName,
    '-map',
    '0:v:0',
    '-map',
    '0:a?',
    ...getConvertResolutionArgs(videoCodec),
    ...getConvertVideoCodecArgs(format, videoCodec),
    ...getConvertAudioCodecArgs(format, videoCodec),
    ...getConvertContainerArgs(format, videoCodec),
    outputName,
  ]
}

function getPreferredConvertVideoCodec(format: string, file: File) {
  const selectedCodec = getSelectedConvertCodec()
  const sourceExtension = getFileExtension(file.name).toUpperCase()
  const canCopySource = convertResolution.value === 'source'
    && (format === sourceExtension || ['MKV', 'MOV', 'MP4', 'M4V'].includes(format))

  if (selectedCodec === '同源编码' && canCopySource) return 'copy'
  if (['MPEG', 'VOB', 'MXF'].includes(format)) return 'mpeg2video'
  if (['WMV', 'ASF'].includes(format)) return 'wmv2'
  if (['3GP', 'XVID', 'DIVX', 'AVI'].includes(format)) return 'mpeg4'
  if (selectedCodec === 'H.265' && ['MP4', 'MOV', 'M4V', 'MKV', 'MTS'].includes(format)) return 'libx265'
  if (selectedCodec === 'MPEG-4') return 'mpeg4'
  if (selectedCodec === 'VP9' && format === 'MKV') return 'libvpx-vp9'
  return 'libx264'
}

function getFallbackConvertVideoCodec(format: string) {
  if (['MPEG', 'VOB', 'MXF'].includes(format)) return 'mpeg2video'
  if (['WMV', 'ASF'].includes(format)) return 'msmpeg4v2'
  if (['3GP', 'XVID', 'DIVX', 'AVI'].includes(format)) return 'mpeg4'
  return 'libx264'
}

function getConvertResolutionArgs(videoCodec: string) {
  if (videoCodec === 'copy' || convertResolution.value === 'source') return []
  return ['-vf', `scale=-2:${convertResolution.value}`]
}

function getConvertVideoCodecArgs(format: string, videoCodec: string) {
  const codecArgs: Record<string, string[]> = {
    copy: ['-c:v', 'copy'],
    libx264: ['-c:v', 'libx264', '-preset', 'veryfast', '-crf', '23', '-pix_fmt', 'yuv420p'],
    libx265: ['-c:v', 'libx265', '-preset', 'ultrafast', '-crf', '28', '-pix_fmt', 'yuv420p'],
    mpeg4: ['-c:v', 'mpeg4', '-q:v', '4', '-pix_fmt', 'yuv420p'],
    mpeg2video: ['-c:v', 'mpeg2video', '-q:v', '4', '-pix_fmt', 'yuv420p'],
    wmv2: ['-c:v', 'wmv2', '-b:v', '1800k'],
    msmpeg4v2: ['-c:v', 'msmpeg4v2', '-b:v', '1800k'],
    'libvpx-vp9': ['-c:v', 'libvpx-vp9', '-b:v', '0', '-crf', '34', '-pix_fmt', 'yuv420p'],
  }
  const args = [...(codecArgs[videoCodec] || codecArgs.libx264)]
  if (videoCodec === 'libx265' && ['MP4', 'MOV', 'M4V'].includes(format)) args.push('-tag:v', 'hvc1')
  if (format === 'XVID') args.push('-vtag', 'xvid')
  if (format === 'DIVX') args.push('-vtag', 'DIVX')
  return args
}

function getConvertAudioCodecArgs(format: string, videoCodec: string) {
  if (videoCodec === 'copy') return ['-c:a', 'copy']
  if (['MPEG', 'VOB'].includes(format)) return ['-c:a', 'mp2', '-b:a', '128k']
  if (['WMV', 'ASF'].includes(format)) return ['-c:a', 'wmav2', '-b:a', '128k']
  if (format === '3GP') return ['-c:a', 'aac', '-b:a', '96k', '-ar', '44100']
  return ['-c:a', 'aac', '-b:a', '128k']
}

function getConvertContainerArgs(format: string, videoCodec: string) {
  const containerMap: Record<string, string> = {
    MP4: 'mp4',
    AVI: 'avi',
    MOV: 'mov',
    MPEG: 'mpeg',
    WMV: 'asf',
    '3GP': '3gp',
    ASF: 'asf',
    MTS: 'mpegts',
    MKV: 'matroska',
    FLV: 'flv',
    M4V: 'mp4',
    XVID: 'avi',
    VOB: 'vob',
    DIVX: 'avi',
    MXF: 'mxf',
  }
  const args = ['-f', containerMap[format] || format.toLowerCase()]
  if (['MP4', 'MOV', 'M4V'].includes(format) && videoCodec !== 'copy') args.unshift('-movflags', 'faststart')
  return args
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
              <button class="pill-button" type="button" @click="openReplacePicker">选择本地文件</button>
              <button type="button" class="back-button" @click="requestResetEditor">返回上传</button>
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

                <div v-if="compressionStatus || compressionError || showWindowsCompressPrompt || compressionOutputUrl" class="compression-feedback">
                  <div v-if="compressionStatus" class="compression-status">
                    <div class="compression-progress-track">
                      <span :style="{ width: `${compressionProgress}%` }" />
                    </div>
                    <p>{{ compressionStatus }}</p>
                  </div>

                  <div v-if="compressionOutputUrl" class="compression-result-card">
                    <div>
                      <strong>{{ compressionOutputName }}</strong>
                      <small>{{ processResultSizePrefix }}{{ compressionResultSizeLabel }}</small>
                    </div>
                    <button type="button" @click="downloadCompressedVideo">{{ processDownloadText }}</button>
                  </div>

                  <div v-if="showWindowsCompressPrompt" class="windows-prompt-card">
                    <strong>建议使用 Windows 桌面端处理</strong>
                    <p>当前文件超过网页端 500MB / 30 分钟限制。基础功能 Windows 端全部永久免费，处理大文件更稳定可靠。</p>
                    <a :href="WINDOWS_DOWNLOAD_URL">下载 Windows 端</a>
                  </div>

                  <p v-else-if="compressionError" class="compression-error">{{ compressionError }}</p>
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

                <div v-if="compressionStatus || compressionError || showWindowsCompressPrompt || compressionOutputUrl" class="compression-feedback">
                  <div v-if="compressionStatus" class="compression-status">
                    <div class="compression-progress-track">
                      <span :style="{ width: `${compressionProgress}%` }" />
                    </div>
                    <p>{{ compressionStatus }}</p>
                  </div>

                  <div v-if="compressionOutputUrl" class="compression-result-card">
                    <div>
                      <strong>{{ compressionOutputName }}</strong>
                      <small>{{ processResultSizePrefix }}{{ compressionResultSizeLabel }}</small>
                    </div>
                    <button type="button" @click="downloadCompressedVideo">{{ processDownloadText }}</button>
                  </div>

                  <div v-if="showWindowsCompressPrompt" class="windows-prompt-card">
                    <strong>建议使用 Windows 桌面端处理</strong>
                    <p>当前文件超过网页端 500MB / 30 分钟限制。基础功能 Windows 端全部永久免费，处理大文件更稳定可靠。</p>
                    <a :href="WINDOWS_DOWNLOAD_URL">下载 Windows 端</a>
                  </div>

                  <p v-else-if="compressionError" class="compression-error">{{ compressionError }}</p>
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
                    <button
                      type="button"
                      :class="{ active: gifRangeMode === 'custom' }"
                      @click="gifRangeMode = 'custom'"
                    >
                      自定义
                    </button>
                  </div>
                  <label v-if="gifRangeMode === 'custom'" class="number-control gif-start-control">
                    <span>起始秒</span>
                    <input v-model.number="gifStartSecond" type="number" min="0" step="1">
                    <strong>s</strong>
                  </label>
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

                <div v-if="compressionStatus || compressionError || showWindowsCompressPrompt || compressionOutputUrl" class="compression-feedback">
                  <div v-if="compressionStatus" class="compression-status">
                    <div class="compression-progress-track">
                      <span :style="{ width: `${compressionProgress}%` }" />
                    </div>
                    <p>{{ compressionStatus }}</p>
                  </div>

                  <div v-if="compressionOutputUrl" class="compression-result-card">
                    <div>
                      <strong>{{ compressionOutputName }}</strong>
                      <small>{{ processResultSizePrefix }}{{ compressionResultSizeLabel }}</small>
                    </div>
                    <button type="button" @click="downloadCompressedVideo">{{ processDownloadText }}</button>
                  </div>

                  <div v-if="showWindowsCompressPrompt" class="windows-prompt-card">
                    <strong>建议使用 Windows 桌面端处理</strong>
                    <p>当前文件超过网页端 500MB / 30 分钟限制。基础功能 Windows 端全部永久免费，处理大文件更稳定可靠。</p>
                    <a :href="WINDOWS_DOWNLOAD_URL">下载 Windows 端</a>
                  </div>

                  <p v-else-if="compressionError" class="compression-error">{{ compressionError }}</p>
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

                <template v-for="setting in activeExportSettings" :key="setting.id">
                  <div v-if="setting.id !== 'transition'" class="compact-control">
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
                  </div>
                </template>

                <div v-if="compressionStatus || compressionError || showWindowsCompressPrompt || compressionOutputUrl" class="compression-feedback">
                  <div v-if="compressionStatus" class="compression-status">
                    <div class="compression-progress-track">
                      <span :style="{ width: `${compressionProgress}%` }" />
                    </div>
                    <p>{{ compressionStatus }}</p>
                  </div>

                  <div v-if="compressionOutputUrl" class="compression-result-card">
                    <div>
                      <strong>{{ compressionOutputName }}</strong>
                      <small>{{ processResultSizePrefix }}{{ compressionResultSizeLabel }}</small>
                    </div>
                    <button type="button" @click="downloadCompressedVideo">{{ processDownloadText }}</button>
                  </div>

                  <div v-if="showWindowsCompressPrompt" class="windows-prompt-card">
                    <strong>建议使用 Windows 桌面端处理</strong>
                    <p>当前文件超过网页端 500MB / 30 分钟限制。基础功能 Windows 端全部永久免费，处理大文件更稳定可靠。</p>
                    <a :href="WINDOWS_DOWNLOAD_URL">下载 Windows 端</a>
                  </div>

                  <p v-else-if="compressionError" class="compression-error">{{ compressionError }}</p>
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
                      <span class="trim-selection-fill" :style="trimSelectionStyle" />
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

                <div class="trim-range-panel">
                  <strong>剪切选区：{{ trimRangeLabel }}</strong>
                  <div class="trim-range-actions">
                    <button class="outline-button" type="button" @click="splitTrimAtCurrentTime">设为起点</button>
                    <button class="outline-button" type="button" @click="deleteTrimSegment">设为终点</button>
                    <button class="outline-button" type="button" @click="resetTrimRange">重置范围</button>
                  </div>
                </div>

                <div v-if="compressionStatus || compressionError || showWindowsCompressPrompt || compressionOutputUrl" class="compression-feedback">
                  <div v-if="compressionStatus" class="compression-status">
                    <div class="compression-progress-track">
                      <span :style="{ width: `${compressionProgress}%` }" />
                    </div>
                    <p>{{ compressionStatus }}</p>
                  </div>

                  <div v-if="compressionOutputUrl" class="compression-result-card">
                    <div>
                      <strong>{{ compressionOutputName }}</strong>
                      <small>{{ processResultSizePrefix }}{{ compressionResultSizeLabel }}</small>
                    </div>
                    <button type="button" @click="downloadCompressedVideo">{{ processDownloadText }}</button>
                  </div>

                  <div v-if="showWindowsCompressPrompt" class="windows-prompt-card">
                    <strong>建议使用 Windows 桌面端处理</strong>
                    <p>当前文件超过网页端 500MB / 30 分钟限制。基础功能 Windows 端全部永久免费，处理大文件更稳定可靠。</p>
                    <a :href="WINDOWS_DOWNLOAD_URL">下载 Windows 端</a>
                  </div>

                  <p v-else-if="compressionError" class="compression-error">{{ compressionError }}</p>
                </div>

                <div class="windows-download-card trim-windows-card">
                  <div>
                    <strong>需要多段剪切？</strong>
                    <small><b>Windows 端支持多段剪切</b>，可一次选择多个片段，进行多段合并或批量导出。</small>
                  </div>
                  <a class="download-button" :href="WINDOWS_DOWNLOAD_URL">下载 Windows 端</a>
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
                      <i class="corner tl" @pointerdown.stop.prevent="startCropResize($event, 'tl')" />
                      <i class="corner tr" @pointerdown.stop.prevent="startCropResize($event, 'tr')" />
                      <i class="corner bl" @pointerdown.stop.prevent="startCropResize($event, 'bl')" />
                      <i class="corner br" @pointerdown.stop.prevent="startCropResize($event, 'br')" />
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

                <div v-if="compressionStatus || compressionError || showWindowsCompressPrompt || compressionOutputUrl" class="compression-feedback">
                  <div v-if="compressionStatus" class="compression-status">
                    <div class="compression-progress-track">
                      <span :style="{ width: `${compressionProgress}%` }" />
                    </div>
                    <p>{{ compressionStatus }}</p>
                  </div>

                  <div v-if="compressionOutputUrl" class="compression-result-card">
                    <div>
                      <strong>{{ compressionOutputName }}</strong>
                      <small>{{ processResultSizePrefix }}{{ compressionResultSizeLabel }}</small>
                    </div>
                    <button type="button" @click="downloadCompressedVideo">{{ processDownloadText }}</button>
                  </div>

                  <div v-if="showWindowsCompressPrompt" class="windows-prompt-card">
                    <strong>建议使用 Windows 桌面端处理</strong>
                    <p>当前文件超过网页端 500MB / 30 分钟限制。基础功能 Windows 端全部永久免费，处理大文件更稳定可靠。</p>
                    <a :href="WINDOWS_DOWNLOAD_URL">下载 Windows 端</a>
                  </div>

                  <p v-else-if="compressionError" class="compression-error">{{ compressionError }}</p>
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

                <div
                  v-if="(activeTool === 'mp4-to-mp3' || activeTool === 'mp3-converter') && (compressionStatus || compressionError || showWindowsCompressPrompt || compressionOutputUrl)"
                  class="compression-feedback"
                >
                  <div v-if="compressionStatus" class="compression-status">
                    <div class="compression-progress-track">
                      <span :style="{ width: `${compressionProgress}%` }" />
                    </div>
                    <p>{{ compressionStatus }}</p>
                  </div>

                  <div v-if="compressionOutputUrl" class="compression-result-card">
                    <div>
                      <strong>{{ compressionOutputName }}</strong>
                      <small>{{ processResultSizePrefix }}{{ compressionResultSizeLabel }}</small>
                    </div>
                    <button type="button" @click="downloadCompressedVideo">{{ processDownloadText }}</button>
                  </div>

                  <div v-if="showWindowsCompressPrompt" class="windows-prompt-card">
                    <strong>建议使用 Windows 桌面端处理</strong>
                    <p>当前文件超过网页端 500MB / 30 分钟限制。基础功能 Windows 端全部永久免费，处理大文件更稳定可靠。</p>
                    <a :href="WINDOWS_DOWNLOAD_URL">下载 Windows 端</a>
                  </div>

                  <p v-else-if="compressionError" class="compression-error">{{ compressionError }}</p>
                </div>
              </template>
            </section>
          </div>

          <div class="settings-footer">
            <div class="panel-actions" :class="{ 'has-secondary': activeTool === 'merge' }">
              <button class="gradient-button" type="button" :disabled="isProcessing" @click="startProcess">
                {{ compressionButtonText }}
              </button>
              <button
                v-if="activeTool === 'merge'"
                class="outline-button"
                type="button"
                :disabled="isProcessing"
                @click="openMergePicker"
              >
                添加文件
              </button>
            </div>
          </div>
        </aside>
      </div>
    </template>

    <input ref="fileInputRef" type="file" accept="video/*,audio/*" hidden @change="handleFileInput">
    <input ref="replaceInputRef" type="file" accept="video/*,audio/*" hidden @change="handleFileInput">
    <input ref="mergeInputRef" type="file" accept="video/*" multiple hidden @change="handleMergeFileInput">

    <div v-if="pendingTaskAction" class="task-guard-backdrop" @click.self="closeTaskGuard">
      <div class="task-guard-dialog" role="dialog" aria-modal="true" aria-labelledby="task-guard-title">
        <button class="task-guard-close" type="button" aria-label="关闭" @click="closeTaskGuard">×</button>
        <strong id="task-guard-title">任务正在处理中</strong>
        <p>任务进行中返回会导致当前任务失败。确认后将停止当前任务，并刷新当前界面。</p>
        <div class="task-guard-actions">
          <button class="outline-button" type="button" @click="closeTaskGuard">继续等待</button>
          <button class="gradient-button" type="button" @click="confirmTaskGuard">确认停止</button>
        </div>
      </div>
    </div>
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

.task-guard-backdrop {
  position: fixed;
  z-index: 100;
  inset: 0;
  display: grid;
  padding: 20px;
  place-items: center;
  background: rgb(0 0 0 / 68%);
  backdrop-filter: blur(8px);
}

.task-guard-dialog {
  position: relative;
  display: grid;
  width: min(420px, 100%);
  gap: 14px;
  padding: 26px;
  border: 1px solid rgb(255 255 255 / 14%);
  border-radius: 16px;
  background: #1b1c21;
  box-shadow: 0 28px 90px rgb(0 0 0 / 42%);
}

.task-guard-dialog strong {
  color: #fff;
  font-size: 22px;
  line-height: 1.3;
}

.task-guard-dialog p {
  margin: 0;
  color: rgb(255 255 255 / 66%);
  font-size: 14px;
  line-height: 1.7;
}

.task-guard-close {
  position: absolute;
  top: 12px;
  right: 12px;
  display: grid;
  width: 34px;
  height: 34px;
  place-items: center;
  border: 0;
  border-radius: 50%;
  background: rgb(255 255 255 / 8%);
  color: rgb(255 255 255 / 72%);
  cursor: pointer;
  font-size: 20px;
}

.task-guard-actions {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
  gap: 12px;
  margin-top: 6px;
}

.task-guard-actions .gradient-button,
.task-guard-actions .outline-button {
  min-width: 0;
  min-height: 44px;
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

.compression-feedback {
  display: grid;
  gap: 10px;
}

.compression-status,
.compression-result-card,
.windows-prompt-card {
  display: grid;
  gap: 10px;
  padding: 14px;
  border: 1px solid rgb(255 255 255 / 10%);
  border-radius: 10px;
  background: rgb(0 0 0 / 18%);
}

.compression-progress-track {
  position: relative;
  height: 8px;
  overflow: hidden;
  border-radius: 999px;
  background: rgb(255 255 255 / 14%);
}

.compression-progress-track span {
  position: absolute;
  inset: 0 auto 0 0;
  border-radius: inherit;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  transition: width 0.2s ease;
}

.compression-status p,
.windows-prompt-card p,
.compression-error {
  margin: 0;
  color: rgb(255 255 255 / 64%);
  font-size: 12px;
  line-height: 1.55;
}

.compression-result-card {
  grid-template-columns: minmax(0, 1fr) auto;
  align-items: center;
  background: rgb(118 249 177 / 9%);
}

.compression-result-card div {
  display: grid;
  min-width: 0;
  gap: 4px;
}

.compression-result-card strong {
  overflow: hidden;
  color: #fff;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.compression-result-card small {
  color: rgb(255 255 255 / 58%);
}

.compression-result-card button,
.windows-prompt-card a {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-height: 36px;
  padding: 0 16px;
  border: 0;
  border-radius: 999px;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  color: #091018;
  cursor: pointer;
  font-weight: 900;
  text-decoration: none;
  white-space: nowrap;
}

.windows-prompt-card {
  border-color: rgb(214 251 114 / 28%);
  background:
    radial-gradient(circle at right top, rgb(118 249 177 / 18%), transparent 42%),
    rgb(255 255 255 / 5%);
}

.windows-prompt-card strong {
  color: #fff;
}

.compression-error {
  color: #ffb0b0;
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
  background: rgb(255 255 255 / 7%);
  box-shadow: inset 0 0 0 1px rgb(255 255 255 / 8%);
  cursor: pointer;
}

.trim-track-fill {
  position: absolute;
  inset: 0 auto 0 0;
  border-radius: inherit;
  background: linear-gradient(90deg, rgb(30 188 62 / 34%), rgb(118 249 177 / 22%));
  z-index: 0;
}

.trim-selection-fill {
  position: absolute;
  top: 4px;
  bottom: 4px;
  border-radius: 6px;
  background: linear-gradient(90deg, rgb(214 251 114 / 92%), rgb(118 249 177 / 88%));
  box-shadow:
    0 0 0 1px rgb(214 251 114 / 52%),
    0 0 18px rgb(118 249 177 / 26%),
    inset 0 0 0 1px rgb(255 255 255 / 24%);
  z-index: 1;
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
  z-index: 2;
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

.trim-range-panel {
  display: grid;
  gap: 10px;
  padding: 12px;
  border: 1px solid rgb(255 255 255 / 10%);
  border-radius: 10px;
  background: rgb(255 255 255 / 5%);
}

.trim-range-panel strong {
  color: #fff;
  font-size: 13px;
}

.trim-range-actions {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 8px;
}

.trim-range-actions .outline-button {
  min-height: 36px;
  font-size: 12px;
}

.trim-windows-card {
  align-items: center;
  padding: 14px;
}

.trim-windows-card strong {
  font-size: 15px;
}

.trim-windows-card .download-button {
  min-height: 38px;
  padding: 0 16px;
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
  cursor: nwse-resize;
}

.corner.tr {
  top: -6px;
  right: -6px;
  cursor: nesw-resize;
}

.corner.bl {
  bottom: -6px;
  left: -6px;
  cursor: nesw-resize;
}

.corner.br {
  right: -6px;
  bottom: -6px;
  cursor: nwse-resize;
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
  grid-template-columns: minmax(0, 1fr);
  gap: 12px;
}

.panel-actions.has-secondary {
  grid-template-columns: minmax(0, 1fr) minmax(0, 0.85fr);
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
  .compression-result-card,
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
