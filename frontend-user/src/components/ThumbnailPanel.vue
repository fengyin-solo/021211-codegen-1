<template>
  <aside class="thumb-panel">
    <div class="thumb-panel__header">
      <span class="thumb-panel__title">页面缩略图</span>
      <span class="thumb-panel__count">共 {{ totalPages }} 页</span>
    </div>
    <div ref="listRef" class="thumb-panel__list" @scroll.passive="onScroll">
      <div
        v-for="n in totalPages"
        :key="n"
        :ref="(el) => setItemRef(el as HTMLElement | null, n)"
        class="thumb-item"
        :class="{ 'thumb-item--active': n === currentPage }"
      >
        <button type="button" class="thumb-item__page" @click="emit('jump', n)" :title="`跳转到第 ${n} 页`">
          <div class="thumb-item__stage" :style="stageStyle(n)">
            <!-- 已生成：页面等比缩影，任意缩放级别下均与页面外观一致 -->
            <canvas
              v-show="stateOf(n).status === 'done'"
              :ref="(el) => setCanvasRef(el as HTMLCanvasElement | null, n)"
              class="thumb-item__canvas"
            />
            <!-- 等待生成 -->
            <div v-if="stateOf(n).status === 'pending'" class="thumb-item__placeholder thumb-item__placeholder--pending">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" width="22" height="22">
                <rect x="4" y="3" width="16" height="18" rx="2"/>
                <line x1="8" y1="8" x2="16" y2="8"/>
                <line x1="8" y1="12" x2="16" y2="12"/>
                <line x1="8" y1="16" x2="12" y2="16"/>
              </svg>
              <p class="thumb-item__placeholder-text">{{ stateOf(n).text || '等待生成' }}</p>
              <p class="thumb-item__placeholder-sub">滚动到此处后自动生成</p>
            </div>
            <!-- 生成中 -->
            <div v-else-if="stateOf(n).status === 'loading'" class="thumb-item__placeholder thumb-item__placeholder--loading">
              <div class="thumb-item__spinner"/>
              <p class="thumb-item__placeholder-text">正在生成…</p>
            </div>
            <!-- 生成失败 -->
            <div v-else-if="stateOf(n).status === 'error'" class="thumb-item__placeholder thumb-item__placeholder--error">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" width="22" height="22">
                <circle cx="12" cy="12" r="9"/>
                <line x1="12" y1="8" x2="12" y2="13"/>
                <circle cx="12" cy="16.5" r="0.6" fill="currentColor"/>
              </svg>
              <p class="thumb-item__placeholder-text">生成失败</p>
              <p class="thumb-item__placeholder-sub">{{ stateOf(n).text || '页面渲染出错' }}</p>
              <span
                class="thumb-item__retry"
                role="button"
                tabindex="0"
                @click.stop="retry(n)"
                @keydown.enter.stop.prevent="retry(n)"
              >点击重试</span>
            </div>
          </div>
          <span class="thumb-item__number">{{ n }}</span>
        </button>
      </div>
    </div>
  </aside>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, nextTick, watch } from 'vue'
import type { PropType } from 'vue'
import type { PdfjsDocument } from '@/utils/pdf-engine'

/**
 * 缩略图单项状态：
 * - pending：尚未生成（含离屏回收后待重新生成）
 * - loading：正在渲染
 * - done：位图已就绪（canvas 可见）
 * - error：渲染失败，展示原因并允许重试
 */
interface ThumbState {
  status: 'pending' | 'loading' | 'done' | 'error'
  /** pending/error 时展示的补充说明 */
  text?: string
}

const props = defineProps({
  pdfDoc: { type: Object as PropType<PdfjsDocument>, required: true },
  totalPages: { type: Number, required: true },
  /** scale=1 时每页的基础尺寸，保证混合页面尺寸的占位框比例正确 */
  pageBaseDims: {
    type: Object as PropType<Map<number, { baseWidth: number; baseHeight: number }>>,
    required: true,
  },
  /** 主视图当前所在页（按视口中心判定） */
  currentPage: { type: Number, default: 1 },
})

/*
 * 说明：缩略图是页面的等比缩影，而页面外观（内容与宽高比）不随主视图
 * 缩放改变，因此缩略图按固定宽度渲染即可，在任意缩放级别下都与页面保持
 * 一致，无需随缩放重绘；DPR 提升保证高分屏下依旧清晰。
 */

const emit = defineEmits<{
  (e: 'jump', pageNumber: number): void
}>()

/* ---- 缩略图绘制参数：固定宽度，高度按页面真实比例 ---- */
const THUMB_WIDTH = 120
const THUMB_PADDING = 8
/* 最多保留的已生成位图数（LRU 上限），防止 200 页文档位图常驻内存 */
const MAX_THUMBS = 30

const listRef = ref<HTMLElement | null>(null)

/** 每页状态（懒初始化），驱动占位 / 加载 / 错误说明的渲染 */
const states = reactive(new Map<number, ThumbState>())
function stateOf(n: number): ThumbState {
  let s = states.get(n)
  if (!s) {
    s = { status: 'pending' }
    states.set(n, s)
  }
  return s
}

/** 每页占位框的等比尺寸（CSS 像素） */
const sizeCache = new Map<number, { width: number; height: number }>()
function displaySize(n: number) {
  let d = sizeCache.get(n)
  if (!d) {
    const base = props.pageBaseDims.get(n)
    // 尺寸信息缺失时退化为 A4 纵向比例，拿到真实尺寸后会重新计算
    const ratio = base && base.baseWidth > 0 ? base.baseHeight / base.baseWidth : 297 / 210
    d = { width: THUMB_WIDTH, height: Math.round(THUMB_WIDTH * ratio) }
    sizeCache.set(n, d)
  }
  return d
}
function stageStyle(n: number) {
  const d = displaySize(n)
  return { width: `${d.width}px`, height: `${d.height}px` }
}

/* ---- 非响应式渲染调度状态 ---- */
const itemRefs = new Map<number, HTMLElement>()
const canvasRefs = new Map<number, HTMLCanvasElement>()
/** 已完成渲染的位图，按最近使用顺序排列，队首优先被回收 */
const renderedOrder: number[] = []
/** 待渲染页码队列 */
let queue: number[] = []
let processing = false
/** 作废令牌：组件卸载时自增，使 await 中的过期任务不再回写状态 */
let version = 0
let scrollRafId: number | null = null

function setItemRef(el: HTMLElement | null, n: number) {
  if (el) itemRefs.set(n, el)
  else itemRefs.delete(n)
}
function setCanvasRef(el: HTMLCanvasElement | null, n: number) {
  if (el) canvasRefs.set(n, el)
  else canvasRefs.delete(n)
}

/* ---- 视口可见性 ---- */
function getVisibleNums(buffer = 0): number[] {
  const list = listRef.value
  if (!list) return []
  const top = list.scrollTop - buffer
  const bottom = list.scrollTop + list.clientHeight + buffer
  const nums: number[] = []
  for (const [n, el] of itemRefs) {
    const t = el.offsetTop
    if (t + el.offsetHeight >= top && t <= bottom) nums.push(n)
  }
  return nums.sort((a, b) => a - b)
}

/**
 * 离屏回收：超出可视区域及缓冲区的已生成位图按 LRU 回收。
 * 回收后回到 pending 并给出原因说明，绝不会留下空白。
 */
function recycle(n: number, reason: string) {
  const canvas = canvasRefs.get(n)
  if (canvas) canvas.width = canvas.height = 0
  const idx = renderedOrder.indexOf(n)
  if (idx !== -1) renderedOrder.splice(idx, 1)
  const s = stateOf(n)
  if (s.status !== 'loading') {
    s.status = 'pending'
    s.text = reason
  }
}

function evict(keep: Set<number>) {
  while (renderedOrder.length > MAX_THUMBS) {
    const victim = renderedOrder.find((n) => !keep.has(n))
    if (victim === undefined) break
    recycle(victim, '已回收，滚动到此处自动重新生成')
  }
}

/* ---- 边滚边生成调度 ---- */
function schedule() {
  const list = listRef.value
  if (!list) return
  const buffer = list.clientHeight
  const visible = getVisibleNums(buffer)
  const keep = new Set(visible)

  // 按距视口中心的距离排序，先生成正在看的页
  const center = list.scrollTop + list.clientHeight / 2
  visible.sort((a, b) => {
    const ea = itemRefs.get(a), eb = itemRefs.get(b)
    if (!ea || !eb) return 0
    return Math.abs(ea.offsetTop + ea.offsetHeight / 2 - center) -
           Math.abs(eb.offsetTop + eb.offsetHeight / 2 - center)
  })

  queue = visible.filter((n) => stateOf(n).status === 'pending')
  evict(keep)
  if (queue.length > 0 && !processing) {
    processing = true
    void processQueue()
  }
}

async function processQueue() {
  const ver = version
  while (queue.length > 0) {
    // 组件在 await 期间卸载则整体丢弃，不再回写状态
    if (ver !== version) break
    const n = queue.shift()!
    if (stateOf(n).status !== 'pending') continue
    await renderThumb(n, ver)
  }
  processing = false
  // 作废后可能已有新任务入队，补一次调度
  if (ver !== version) schedule()
}

async function renderThumb(n: number, ver: number) {
  const doc = props.pdfDoc
  if (!doc) return
  const state = stateOf(n)
  state.status = 'loading'
  state.text = undefined

  try {
    const page = await doc.getPage(n)
    if (ver !== version) return

    // 固定缩略宽度，高度由 PDF 页面真实宽高比决定（页面外观不随缩放改变）
    const base = props.pageBaseDims.get(n)
    const baseW = base?.baseWidth || page.getViewport({ scale: 1 }).width
    const thumbScale = THUMB_WIDTH / baseW

    const viewport = page.getViewport({ scale: thumbScale })
    const dpr = window.devicePixelRatio || 1
    const canvas = canvasRefs.get(n)
    if (!canvas) return

    // 先渲染到离屏 canvas，成功后再整体拷贝到可见 canvas，避免半成品闪烁
    const off = document.createElement('canvas')
    off.width = Math.floor(viewport.width * dpr)
    off.height = Math.floor(viewport.height * dpr)
    const ctx = off.getContext('2d')
    if (!ctx) throw new Error('无法创建绘图上下文')
    ctx.scale(dpr, dpr)
    ctx.fillStyle = '#ffffff'
    ctx.fillRect(0, 0, viewport.width, viewport.height)
    await page.render({ canvasContext: ctx, viewport }).promise
    if (ver !== version) return

    const target = canvas.getContext('2d')
    if (!target) throw new Error('画布不可用')
    canvas.width = off.width
    canvas.height = off.height
    canvas.style.width = `${viewport.width}px`
    canvas.style.height = `${viewport.height}px`
    target.drawImage(off, 0, 0)

    const idx = renderedOrder.indexOf(n)
    if (idx !== -1) renderedOrder.splice(idx, 1)
    renderedOrder.push(n)

    state.status = 'done'
    state.text = undefined
  } catch (e) {
    if (ver !== version) return
    const reason = e instanceof Error ? e.message : String(e)
    state.status = 'error'
    state.text = reason ? `页面渲染出错：${reason}` : '页面渲染出错，请重试'
    console.error(`缩略图第${n}页生成失败:`, e)
  }
}

function retry(n: number) {
  const s = stateOf(n)
  if (s.status !== 'error') return
  s.status = 'pending'
  s.text = undefined
  schedule()
}

function onScroll() {
  if (scrollRafId !== null) return
  scrollRafId = requestAnimationFrame(() => {
    scrollRafId = null
    schedule()
  })
}

/* ---- 当前页变化时自动把高亮项滚动进缩略图列表 ---- */
watch(() => props.currentPage, (n) => {
  const list = listRef.value
  const el = itemRefs.get(n)
  if (!list || !el) return
  const top = el.offsetTop
  const bottom = top + el.offsetHeight
  if (top < list.scrollTop) {
    list.scrollTo({ top: top - THUMB_PADDING })
  } else if (bottom > list.scrollTop + list.clientHeight) {
    list.scrollTo({ top: bottom - list.clientHeight + THUMB_PADDING })
  }
}, { flush: 'post' })

onMounted(async () => {
  await nextTick()
  schedule()
})

onUnmounted(() => {
  // 作废所有进行中的渲染任务
  version++
  queue = []
  processing = false
  if (scrollRafId !== null) cancelAnimationFrame(scrollRafId)
})
</script>

<style lang="scss" scoped>
.thumb-panel {
  width: 168px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  background: var(--card-bg);
  border-right: 1px solid var(--border-color);
  &__header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 10px 12px;
    border-bottom: 1px solid var(--border-color);
    flex-shrink: 0;
  }
  &__title {
    font-size: var(--font-size-sm);
    font-weight: 600;
    color: var(--text-primary);
  }
  &__count {
    font-size: var(--font-size-sm);
    color: var(--text-tertiary);
    font-variant-numeric: tabular-nums;
  }
  &__list {
    flex: 1;
    overflow-y: auto;
    overflow-x: hidden;
    padding: 8px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }
}

.thumb-item {
  display: flex;
  justify-content: center;
  border-radius: var(--radius-sm);
  padding: 6px;
  border: 2px solid transparent;
  transition: border-color 0.15s, background 0.15s;
  &--active {
    border-color: var(--primary-color);
    background: #e6f4ff;
  }
  &__page {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 4px;
    padding: 0;
    border: none;
    background: transparent;
    cursor: pointer;
    font: inherit;
  }
  &__stage {
    position: relative;
    background: #fff;
    border: 1px solid var(--border-color);
    border-radius: 2px;
    box-shadow: var(--shadow-sm);
    overflow: hidden;
  }
  &__canvas {
    display: block;
    width: 100%;
    height: 100%;
  }
  &__number {
    font-size: var(--font-size-sm);
    color: var(--text-tertiary);
    font-variant-numeric: tabular-nums;
    line-height: 1.4;
  }
  &--active &__number {
    color: var(--primary-color);
    font-weight: 600;
  }
  &__placeholder {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 4px;
    padding: 6px;
    text-align: center;
    background: #fafafa;
  }
  &__placeholder--pending { color: var(--text-tertiary); }
  &__placeholder--loading { color: var(--primary-color); background: #f5faff; }
  &__placeholder--error { color: #cf1322; background: #fff8f7; }
  &__placeholder-text {
    font-size: 11px;
    line-height: 1.3;
    color: inherit;
  }
  &__placeholder-sub {
    font-size: 10px;
    line-height: 1.3;
    color: var(--text-tertiary);
    word-break: break-all;
  }
  &__spinner {
    width: 20px;
    height: 20px;
    border: 2px solid var(--border-color);
    border-top-color: var(--primary-color);
    border-radius: 50%;
    animation: thumb-spin 0.8s linear infinite;
    flex-shrink: 0;
  }
  &__retry {
    margin-top: 2px;
    font-size: 10px;
    line-height: 1.2;
    color: var(--primary-color);
    cursor: pointer;
    text-decoration: underline;
    &:hover { color: var(--primary-active); }
  }
}

@keyframes thumb-spin {
  to { transform: rotate(360deg); }
}
</style>
