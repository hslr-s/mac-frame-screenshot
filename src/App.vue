<script setup lang="ts">
import { ref, computed, watch, onMounted, onBeforeUnmount, nextTick } from 'vue'

type ImgSource = HTMLImageElement | HTMLCanvasElement
type Theme = 'dark' | 'light'

const sourceWidth = (s: ImgSource) => (s instanceof HTMLImageElement ? s.naturalWidth : s.width)
const sourceHeight = (s: ImgSource) => (s instanceof HTMLImageElement ? s.naturalHeight : s.height)

const theme = ref<Theme>('dark')
const corner = ref(18)
const titleRatio = ref(3.5)
const padding = ref(0)

const shadowOn = ref(true)
const shadowBlur = ref(48)
const shadowOpacity = ref(32)

const exportScale = ref(2)
const exportFormat = ref<'png' | 'webp'>('png')
const exportQuality = ref(92)

const fileInput = ref<HTMLInputElement | null>(null)
const canvasRef = ref<HTMLCanvasElement | null>(null)
const source = ref<ImgSource | null>(null)
const isDragging = ref(false)
const toast = ref('')
const exportScaleOptions = [1, 2, 3]

let objectUrl = ''
let toastTimer = 0

const themeColor = computed(() =>
  theme.value === 'dark'
    ? { bar: '#2c2c2e', content: '#1c1c1e', dotStroke: 'rgba(0,0,0,.28)', line: 'rgba(255,255,255,.06)' }
    : { bar: '#ececec', content: '#ffffff', dotStroke: 'rgba(0,0,0,.12)', line: 'rgba(0,0,0,.08)' },
)

function roundRectPath(ctx: CanvasRenderingContext2D, x: number, y: number, w: number, h: number, r: number) {
  const radius = Math.max(0, Math.min(r, w / 2, h / 2))
  ctx.beginPath()
  ctx.moveTo(x + radius, y)
  ctx.arcTo(x + w, y, x + w, y + h, radius)
  ctx.arcTo(x + w, y + h, x, y + h, radius)
  ctx.arcTo(x, y + h, x, y, radius)
  ctx.arcTo(x, y, x + w, y, radius)
  ctx.closePath()
}

function render() {
  const canvas = canvasRef.value
  const img = source.value
  if (!canvas || !img) return

  const scale = exportScale.value
  const iw = sourceWidth(img)
  const ih = sourceHeight(img)

  const padX = padding.value
  const padBottom = padding.value
  const titleH = Math.max(26, Math.min(96, Math.round(iw * (titleRatio.value / 100))))
  const frameW = iw + padX * 2
  const frameH = titleH + ih + padBottom

  const blur = shadowOn.value ? shadowBlur.value : 0
  const shadowPad = Math.ceil(blur * 1.4 + 24)
  const canvasW = frameW + shadowPad * 2
  const canvasH = frameH + shadowPad * 2

  canvas.width = Math.round(canvasW * scale)
  canvas.height = Math.round(canvasH * scale)

  const ctx = canvas.getContext('2d')
  if (!ctx) return
  ctx.setTransform(scale, 0, 0, scale, 0, 0)
  ctx.clearRect(0, 0, canvasW, canvasH)

  const fx = shadowPad
  const fy = shadowPad
  const color = themeColor.value
  const radius = corner.value

  // 1) 整个窗口：背景色 + 阴影
  ctx.save()
  if (shadowOn.value && shadowOpacity.value > 0) {
    ctx.shadowColor = `rgba(0, 0, 0, ${shadowOpacity.value / 100})`
    ctx.shadowBlur = blur
    ctx.shadowOffsetY = Math.max(6, blur * 0.28)
  }
  ctx.fillStyle = color.content
  roundRectPath(ctx, fx, fy, frameW, frameH, radius)
  ctx.fill()
  ctx.restore()

  // 2) 裁到圆角内，画标题栏与截图（保证图片底部圆角被保留）
  ctx.save()
  roundRectPath(ctx, fx, fy, frameW, frameH, radius)
  ctx.clip()

  ctx.fillStyle = color.bar
  ctx.fillRect(fx, fy, frameW, titleH)

  ctx.drawImage(img, fx + padX, fy + titleH, iw, ih)

  if (padding.value > 0) {
    ctx.fillStyle = color.line
    ctx.fillRect(fx, fy + titleH - 1, frameW, 1)
  }
  ctx.restore()

  // 3) 红黄绿三个控制点
  const dotR = Math.max(4, Math.min(8, titleH * 0.14))
  const dotGap = dotR * 2 + dotR * 1.1
  const startX = fx + Math.max(14, titleH * 0.42)
  const dotY = fy + titleH / 2
  const dots = ['#ff5f57', '#febc2e', '#28c840']
  dots.forEach((fill, i) => {
    ctx.beginPath()
    ctx.arc(startX + i * dotGap, dotY, dotR, 0, Math.PI * 2)
    ctx.fillStyle = fill
    ctx.fill()
    ctx.lineWidth = 0.8
    ctx.strokeStyle = color.dotStroke
    ctx.stroke()
  })
}

function showToast(message: string) {
  toast.value = message
  window.clearTimeout(toastTimer)
  toastTimer = window.setTimeout(() => (toast.value = ''), 2600)
}

function loadFromSource(src: string, revoke = false) {
  const image = new Image()
  image.onload = () => {
    if (objectUrl) URL.revokeObjectURL(objectUrl)
    objectUrl = revoke ? src : ''
    source.value = image
    nextTick(render)
  }
  image.onerror = () => showToast('图片加载失败，换一张试试')
  image.src = src
}

function handleFile(file?: File) {
  if (!file) return
  if (!file.type.startsWith('image/')) {
    showToast('请选择图片文件')
    return
  }
  const url = URL.createObjectURL(file)
  loadFromSource(url, true)
  showToast('图片已载入')
}

function onFileChange(e: Event) {
  const input = e.target as HTMLInputElement
  handleFile(input.files?.[0])
  input.value = ''
}

function onPaste(e: ClipboardEvent) {
  const items = e.clipboardData?.items
  if (!items) return
  for (const item of items) {
    if (item.type.startsWith('image/')) {
      e.preventDefault()
      handleFile(item.getAsFile() ?? undefined)
      return
    }
  }
}

function onDragOver(e: DragEvent) {
  e.preventDefault()
  isDragging.value = true
}

function onDragLeave() {
  isDragging.value = false
}

function onDrop(e: DragEvent) {
  e.preventDefault()
  isDragging.value = false
  handleFile(e.dataTransfer?.files?.[0])
}

function createDemoImage(): HTMLCanvasElement {
  const c = document.createElement('canvas')
  c.width = 1280
  c.height = 800
  const x = c.getContext('2d')!

  const g = x.createLinearGradient(0, 0, 1280, 800)
  g.addColorStop(0, '#241a4d')
  g.addColorStop(0.55, '#1a2a5e')
  g.addColorStop(1, '#3a1a4d')
  x.fillStyle = g
  x.fillRect(0, 0, 1280, 800)

  const blob = (cx: number, cy: number, r: number, color: string) => {
    const rg = x.createRadialGradient(cx, cy, 0, cx, cy, r)
    rg.addColorStop(0, color)
    rg.addColorStop(1, 'rgba(0,0,0,0)')
    x.fillStyle = rg
    x.fillRect(cx - r, cy - r, r * 2, r * 2)
  }
  blob(240, 700, 420, 'rgba(220,40,120,.45)')
  blob(1000, 180, 380, 'rgba(40,90,220,.35)')

  x.textAlign = 'center'
  x.fillStyle = '#ffffff'
  x.font = '600 66px -apple-system, "PingFang SC", sans-serif'
  x.fillText('Put your screenshot here', 640, 350)
  x.fillStyle = 'rgba(255,255,255,.72)'
  x.font = '400 26px -apple-system, "PingFang SC", sans-serif'
  x.fillText('拖入图片 · 粘贴截图 · 或点击左侧选择文件', 640, 412)

  const cards = [
    [150, 500, 220],
    [420, 500, 220],
    [690, 500, 220],
    [960, 500, 220],
  ]
  cards.forEach(([cx, cy, w]) => {
    const h = 150
    const rx = cx - w / 2
    const ry = cy - h / 2
    const r = 16
    x.beginPath()
    x.moveTo(rx + r, ry)
    x.arcTo(rx + w, ry, rx + w, ry + h, r)
    x.arcTo(rx + w, ry + h, rx, ry + h, r)
    x.arcTo(rx, ry + h, rx, ry, r)
    x.arcTo(rx, ry, rx + w, ry, r)
    x.closePath()
    x.fillStyle = 'rgba(255,255,255,.08)'
    x.fill()
    x.fillStyle = 'rgba(255,255,255,.32)'
    x.fillRect(rx + 20, ry + 28, 92, 10)
    x.fillRect(rx + 20, ry + 56, 140, 8)
    x.fillRect(rx + 20, ry + 80, 112, 8)
  })

  return c
}

function exportImage() {
  const canvas = canvasRef.value
  if (!canvas || !source.value) return
  const format = exportFormat.value
  const mime = format === 'webp' ? 'image/webp' : 'image/png'
  const quality = exportQuality.value / 100
  canvas.toBlob(
    (blob) => {
      if (!blob) {
        showToast('导出失败，请重试')
        return
      }
      const actualFormat = blob.type === 'image/webp' ? 'webp' : 'png'
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      a.download = `screenshot-framed-${Date.now()}.${actualFormat}`
      a.click()
      setTimeout(() => URL.revokeObjectURL(url), 1000)
      showToast(
        actualFormat === format
          ? `${format.toUpperCase()} 已保存`
          : '当前浏览器不支持 WebP，已保存为 PNG',
      )
    },
    mime,
    format === 'webp' ? quality : undefined,
  )
}

watch([theme, corner, titleRatio, padding, shadowOn, shadowBlur, shadowOpacity, exportScale], () => {
  nextTick(render)
})

onMounted(() => {
  source.value = createDemoImage()
  nextTick(render)
  window.addEventListener('paste', onPaste)
})

onBeforeUnmount(() => {
  window.removeEventListener('paste', onPaste)
  window.clearTimeout(toastTimer)
  if (objectUrl) URL.revokeObjectURL(objectUrl)
})
</script>

<template>
  <div class="app">
    <header class="topbar">
      <div class="brand">
        <span class="logo" aria-hidden="true"></span>
        <div>
          <h1>截图套壳</h1>
          <p>上传截图，套上 macOS 窗口，导出高清图片</p>
        </div>
      </div>
      <button class="export-btn" :disabled="!source" @click="exportImage">
        <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <path d="M12 3v12m0 0 4-4m-4 4-4-4" />
          <path d="M4 17v3h16v-3" />
        </svg>
        导出 {{ exportFormat.toUpperCase() }}
      </button>
    </header>

    <main class="layout">
      <aside class="panel">
        <section class="card">
          <h2 class="card-title">图片</h2>
          <input ref="fileInput" type="file" accept="image/*" class="hidden-input" @change="onFileChange" />
          <div
            class="dropzone"
            :class="{ dragging: isDragging }"
            @click="fileInput?.click()"
            @dragover="onDragOver"
            @dragleave="onDragLeave"
            @drop="onDrop"
          >
            <svg viewBox="0 0 24 24" width="26" height="26" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
              <rect x="3" y="4" width="18" height="16" rx="2" />
              <circle cx="8.5" cy="9" r="1.5" />
              <path d="m4 17 5-5 4 4 3-3 4 4" />
            </svg>
            <span>点击选择，或拖入图片</span>
            <small>也可以直接 ⌘V 粘贴截图</small>
          </div>
        </section>

        <section class="card">
          <h2 class="card-title">框架主题</h2>
          <div class="segmented">
            <button :class="{ active: theme === 'dark' }" @click="theme = 'dark'">深色</button>
            <button :class="{ active: theme === 'light' }" @click="theme = 'light'">浅色</button>
          </div>
        </section>

        <section class="card">
          <h2 class="card-title">窗口样式</h2>
          <label class="field">
            <span>圆角 <b>{{ corner }}px</b></span>
            <input v-model.number="corner" type="range" min="0" max="40" step="1" />
          </label>
          <label class="field">
            <span>标题栏高度 <b>{{ titleRatio }}%</b></span>
            <input v-model.number="titleRatio" type="range" min="2" max="8" step="0.1" />
          </label>
          <label class="field">
            <span>边框留白 <b>{{ padding }}px</b></span>
            <input v-model.number="padding" type="range" min="0" max="40" step="1" />
          </label>
        </section>

        <section class="card">
          <div class="card-head">
            <h2 class="card-title">阴影</h2>
            <button class="switch" :class="{ on: shadowOn }" role="switch" :aria-checked="shadowOn" @click="shadowOn = !shadowOn">
              <span></span>
            </button>
          </div>
          <label class="field" :class="{ disabled: !shadowOn }">
            <span>柔和度 <b>{{ shadowBlur }}</b></span>
            <input v-model.number="shadowBlur" type="range" min="0" max="100" step="1" :disabled="!shadowOn" />
          </label>
          <label class="field" :class="{ disabled: !shadowOn }">
            <span>浓度 <b>{{ shadowOpacity }}%</b></span>
            <input v-model.number="shadowOpacity" type="range" min="0" max="70" step="1" :disabled="!shadowOn" />
          </label>
        </section>

        <section class="card">
          <h2 class="card-title">导出格式</h2>
          <div class="segmented">
            <button :class="{ active: exportFormat === 'png' }" @click="exportFormat = 'png'">PNG · 无损</button>
            <button :class="{ active: exportFormat === 'webp' }" @click="exportFormat = 'webp'">WebP · 更小</button>
          </div>
          <label v-if="exportFormat === 'webp'" class="field">
            <span>画质 <b>{{ exportQuality }}%</b></span>
            <input v-model.number="exportQuality" type="range" min="50" max="100" step="1" />
          </label>
          <p class="hint">WebP 在相同画质下体积更小，PNG 兼容性最好。</p>
        </section>

        <section class="card">
          <h2 class="card-title">导出倍率</h2>
          <div class="segmented">
            <button v-for="s in exportScaleOptions" :key="s" :class="{ active: exportScale === s }" @click="exportScale = s">{{ s }}x</button>
          </div>
          <p class="hint">倍率越高，导出的图片越清晰、文件越大。</p>
        </section>
      </aside>

      <section class="preview" :class="{ dragging: isDragging }" @dragover="onDragOver" @dragleave="onDragLeave" @drop="onDrop">
        <div class="preview-head">
          <span>预览</span>
          <span v-if="source" class="resolution">{{ sourceWidth(source) }} × {{ sourceHeight(source) }}</span>
        </div>
        <div class="stage">
          <canvas ref="canvasRef" class="result-canvas"></canvas>
        </div>
      </section>
    </main>

    <transition name="toast">
      <div v-if="toast" class="toast">{{ toast }}</div>
    </transition>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  background: #f4f5f7;
  color: #1d1d1f;
  font-family: -apple-system, BlinkMacSystemFont, 'PingFang SC', 'Segoe UI', sans-serif;
}
.topbar {
  height: 72px;
  background: #fff;
  border-bottom: 1px solid #e8e8ea;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 28px;
  position: sticky;
  top: 0;
  z-index: 5;
}
.brand { display: flex; align-items: center; gap: 13px; }
.logo {
  width: 34px; height: 34px; border-radius: 9px;
  background: linear-gradient(145deg, #3a3a3c, #1c1c1e);
  box-shadow: inset 0 0 0 1px rgba(255,255,255,.08);
  position: relative;
}
.logo::before {
  content: ''; position: absolute; top: 8px; left: 8px; width: 6px; height: 6px; border-radius: 50%;
  background: #ff5f57; box-shadow: 9px 0 #febc2e, 18px 0 #28c840;
}
.brand h1 { font-size: 16px; font-weight: 600; margin: 0 0 2px; }
.brand p { font-size: 12px; color: #86868b; margin: 0; }
.export-btn {
  display: inline-flex; align-items: center; gap: 8px;
  background: #1d1d1f; color: #fff; border: 0; border-radius: 9px;
  padding: 10px 18px; font-size: 14px; font-weight: 500; cursor: pointer;
  transition: background .2s, transform .1s;
}
.export-btn:hover { background: #343436; }
.export-btn:active { transform: translateY(1px); }
.export-btn:disabled { opacity: .45; cursor: not-allowed; }
.layout { display: flex; align-items: flex-start; gap: 22px; padding: 24px 28px 48px; max-width: 1560px; margin: 0 auto; }
.panel { width: 300px; flex: 0 0 300px; display: flex; flex-direction: column; gap: 16px; }
.card { background: #fff; border: 1px solid #e8e8ea; border-radius: 13px; padding: 17px 18px; }
.card-title { font-size: 13px; font-weight: 600; margin: 0 0 14px; }
.card-head { display: flex; align-items: center; justify-content: space-between; }
.card-head .card-title { margin-bottom: 0; }
.hidden-input { display: none; }
.dropzone {
  border: 1.5px dashed #d2d2d6; border-radius: 10px; padding: 24px 14px;
  display: flex; flex-direction: column; align-items: center; gap: 7px;
  color: #6e6e73; cursor: pointer; text-align: center; transition: all .2s;
}
.dropzone:hover, .dropzone.dragging { border-color: #7a7a80; background: #fafafa; color: #1d1d1f; }
.dropzone span { font-size: 13px; font-weight: 500; }
.dropzone small { font-size: 11px; color: #a1a1a6; }
.segmented { display: flex; background: #f0f0f2; border-radius: 9px; padding: 3px; gap: 3px; }
.segmented button {
  flex: 1; border: 0; background: transparent; padding: 8px 0; border-radius: 7px;
  font-size: 13px; color: #6e6e73; cursor: pointer; transition: all .18s;
}
.segmented button.active { background: #fff; color: #1d1d1f; font-weight: 500; box-shadow: 0 1px 3px rgba(0,0,0,.1); }
.field { display: block; margin-top: 16px; }
.field:first-of-type { margin-top: 0; }
.field > span { display: flex; justify-content: space-between; font-size: 12px; color: #6e6e73; margin-bottom: 9px; }
.field b { color: #1d1d1f; font-weight: 500; }
.field.disabled { opacity: .4; }
input[type='range'] { width: 100%; height: 4px; appearance: none; background: #e3e3e6; border-radius: 3px; outline: none; }
input[type='range']::-webkit-slider-thumb {
  appearance: none; width: 16px; height: 16px; border-radius: 50%; background: #fff;
  border: 1px solid #c7c7cc; box-shadow: 0 1px 3px rgba(0,0,0,.18); cursor: pointer;
}
.switch { width: 40px; height: 23px; border-radius: 20px; border: 0; background: #d2d2d6; padding: 2px; cursor: pointer; transition: background .2s; }
.switch span { display: block; width: 19px; height: 19px; border-radius: 50%; background: #fff; box-shadow: 0 1px 2px rgba(0,0,0,.2); transition: transform .2s; }
.switch.on { background: #34c759; }
.switch.on span { transform: translateX(17px); }
.hint { font-size: 11px; color: #a1a1a6; margin: 12px 0 0; line-height: 1.5; }
.preview { flex: 1; min-width: 0; background: #fff; border: 1px solid #e8e8ea; border-radius: 13px; overflow: hidden; transition: border-color .2s; }
.preview.dragging { border-color: #7a7a80; }
.preview-head {
  height: 46px; padding: 0 18px; border-bottom: 1px solid #eeeef0;
  display: flex; align-items: center; justify-content: space-between;
  font-size: 12px; font-weight: 500; color: #6e6e73;
}
.resolution { color: #a1a1a6; font-variant-numeric: tabular-nums; }
.stage {
  padding: 40px; min-height: 560px;
  display: flex; align-items: center; justify-content: center;
  background-color: #efeff1;
  background-image: linear-gradient(45deg, #e6e6e9 25%, transparent 25%), linear-gradient(-45deg, #e6e6e9 25%, transparent 25%),
    linear-gradient(45deg, transparent 75%, #e6e6e9 75%), linear-gradient(-45deg, transparent 75%, #e6e6e9 75%);
  background-size: 20px 20px;
  background-position: 0 0, 0 10px, 10px -10px, -10px 0;
}
.result-canvas { max-width: 100%; height: auto; border-radius: 2px; }
.toast {
  position: fixed; bottom: 30px; left: 50%; transform: translateX(-50%);
  background: #1d1d1f; color: #fff; padding: 11px 20px; border-radius: 10px;
  font-size: 13px; box-shadow: 0 8px 24px rgba(0,0,0,.2); z-index: 20;
}
.toast-enter-active, .toast-leave-active { transition: opacity .25s, transform .25s; }
.toast-enter-from, .toast-leave-to { opacity: 0; transform: translate(-50%, 10px); }
@media (max-width: 900px) {
  .layout { flex-direction: column; }
  .panel { width: 100%; flex: none; }
  .stage { padding: 20px; min-height: 360px; }
}
</style>
