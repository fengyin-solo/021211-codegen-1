<template>
  <!-- 折叠态：只保留一条窄轨道，点击展开 -->
  <div v-if="collapsed" class="thumb-rail" @click="collapsed = false" title="展开页面缩略图">
    <svg class="thumb-rail__expand" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="16" height="16">
      <polyline points="9 18 15 12 9 6" />
    </svg>
    <span class="thumb-rail__text">页面缩略图</span>
  </div>

  <!-- 展开态 -->
  <aside v-else class="thumb-panel">
    <div class="thumb-panel__header">
      <span class="thumb-panel__title">页面</span>
      <span class="thumb-panel__count">{{ totalPages }} 页</span>
      <button class="thumb-panel__collapse" @click="collapsed = true" title="收起缩略图栏">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="16" height="16">
          <polyline points="15 18 9 12 15 6" />
        </svg>
      </button>
    </div>

    <div ref="listRef" class="thumb-panel__list" @scroll="onListScroll" @pointerdown="onUserInteract">
      <div class="thumb-panel__spacer" :style="{ height: `${contentHeight}px`, position: 'relative' }">
        <div
          v-for="n in visibleItems"
          :key="n"
          class="thumb-item"
          :class="{ 'thumb-item--active': n === currentPage }"
          :style="{ transform: `translateY(${itemTop(n - 1)}px)` }"
          @click="onClickItem(n)"
        >
          <div class="thumb-item__paper" :style="{ height: `${thumbHeight(n)}px` }">
            <div
              :ref="(el) => setItemRef(el as HTMLDivElement | null, n)"
              v-show="getState(n).status === 'success'"
              class="thumb-item__canvas-host"
            />
            <div v-if="getState(n).status === 'pending'" class="thumb-item__placeholder">
              <p class="thumb-item__placeholder-text">等待生成</p>
            </div>
            <div v-else-if="getState(n).status === 'loading'" class="thumb-item__placeholder">
              <div class="thumb-item__spinner" />
              <p class="thumb-item__placeholder-text">正在生成…</p>
            </div>
            <div v-else-if="getState(n).status === 'error'" class="thumb-item__placeholder thumb-item__placeholder--error">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="18" height="18">
                <circle cx="12" cy="12" r="10"/><line x1="12" y1="8" x2="12" y2="12"/><line x1="12" y1="16" x2="12.01" y2="16"/>
              </svg>
              <p class="thumb-item__placeholder-text">生成失败</p>
              <p class="thumb-item__placeholder-reason">{{ getState(n).error || '页面渲染出错' }}</p>
              <button class="thumb-item__retry" @click.stop="retry(n)">重试</button>
            </div>
          </div>
          <span class="thumb-item__number" :class="{ 'thumb-item__number--active': n === currentPage }">{{ n }}</span>
        </div>
      </div>
    </div>
  </aside>
</template>

<script setup lang="ts">
import { ref, reactive, computed, watch, onMounted, onUnmounted } from 'vue'
import { renderPageThumbnail, type PdfjsDocument } from '@/utils/pdf-engine'

interface PageBaseDim { baseWidth: number; baseHeight: number }
type ThumbStatus = 'pending' | 'loading' | 'success' | 'error'

interface ThumbState {
  status: ThumbStatus
  error: string
}

const props = defineProps<{
  pdfDoc: PdfjsDocument
  totalPages: number
  /** 当前主视图正在阅读的页码（用于高亮与联动滚动） */
  currentPage: number
  /** 主视图当前缩放比；变化后缩略图按最新外观重新生成 */
  scale: number
  /** 每页 scale=1 的基础尺寸（主视图预计算完成后已填充） */
  pageBaseDims: Map<number, PageBaseDim>
}>()

const emit = defineEmits<{
  (e: 'navigate', pageNumber: number): void
}>()

/* ---- 布局常量（CSS 像素） ---- */
const THUMB_WIDTH = 128          // 缩略纸张宽度
const ITEM_PAD_TOP = 10         // 列表内每项顶部留白
const LABEL_HEIGHT = 18         // 页码标签高度
const LABEL_GAP = 6             // 纸张与页码间距
const ITEM_GAP_BOTTOM = 14      // 与下一项之间的间隔
const FALLBACK_RATIO = 297 / 210 // 尺寸未知时按 A4 纵向占位

const collapsed = ref(false)
const listRef = ref<HTMLElement | null>(null)

/* ---- 虚拟列表视口状态 ---- */
const viewportH = ref(0)
const scrollTop = ref(0)
const BUFFER_PX = 600 // 上下各多渲染约两屏，快速滚动不易看到空白占位

/* ---- 每页缩略图状态（驱动模板） ---- */
const states = reactive(new Map<number, ThumbState>())
/** 预生成全部页的状态对象，模板只读不写，避免渲染期间变更响应式状态 */
function seedStates() {
  states.clear()
  for (let n = 1; n <= props.totalPages; n++) {
    states.set(n, { status: 'pending', error: '' })
  }
}
function getState(n: number): ThumbState {
  return states.get(n) ?? { status: 'pending', error: '' }
}

/* ---- 单页高度 / 总高度（依赖基础尺寸） ---- */
function thumbHeight(n: number): number {
  const d = props.pageBaseDims.get(n)
  const ratio = d ? d.baseHeight / d.baseWidth : FALLBACK_RATIO
  return Math.round(THUMB_WIDTH * ratio)
}
function itemHeight(n: number): number {
  return ITEM_PAD_TOP + thumbHeight(n) + LABEL_GAP + LABEL_HEIGHT + ITEM_GAP_BOTTOM
}
function itemTop(index: number): number {
  // index 为页码前的页数（0-based）
  let top = 0
  for (let i = 1; i <= index; i++) top += itemHeight(i)
  return top
}
const contentHeight = computed(() => {
  let h = 0
  for (let n = 1; n <= props.totalPages; n++) h += itemHeight(n)
  return h
})

/* ---- 可见窗口（含缓冲区），只渲染窗口内的缩略图 ---- */
const visibleItems = computed<number[]>(() => {
  const start = scrollTop.value - BUFFER_PX
  const end = scrollTop.value + viewportH.value + BUFFER_PX
  const items: number[] = []
  let acc = 0
  for (let n = 1; n <= props.totalPages; n++) {
    const h = itemHeight(n)
    if (acc + h >= start && acc <= end) items.push(n)
    acc += h
    if (acc > end) break
  }
  return items
})

/* ---- 缩略图位图 LRU：离开窗口的位图保留一段时间，再次进入即时挂回 ---- */
const bitmapCache = new Map<number, HTMLCanvasElement>()
const bitmapOrder: number[] = []
const MAX_CACHED_BITMAPS = 40

function evictBitmaps() {
  while (bitmapOrder.length > MAX_CACHED_BITMAPS) {
    const old = bitmapOrder.shift()
    if (old === undefined) break
    bitmapCache.delete(old)
    const st = states.get(old)
    // 位图被真正回收后，若该项仍在窗口内会被重新排队；否则回到待生成占位
    if (st && st.status !== 'error') {
      st.status = 'pending'
      st.error = ''
    }
  }
}

/* ---- 渲染队列（串行低优先级，避免与主视图抢渲染资源） ---- */
const renderQueue: number[] = []
let processing = false
let renderVersion = 0

function enqueueVisible() {
  const vis = visibleItems.value
  if (vis.length === 0) return
  const centerN = centerPage.value
  const wanted = new Set(vis)
  // 移除已不在窗口的排队任务
  for (let i = renderQueue.length - 1; i >= 0; i--) {
    if (!wanted.has(renderQueue[i])) renderQueue.splice(i, 1)
  }
  // 按距视口中心的距离排序，越靠近当前阅读位置越先生成
  const ordered = vis
    .filter((n) => {
      if (bitmapCache.has(n)) return false
      const st = states.get(n)
      return !st || st.status === 'pending' || st.status === 'error'
    })
    .sort((a, b) => Math.abs(a - centerN) - Math.abs(b - centerN))
  for (const n of ordered) {
    if (!renderQueue.includes(n)) renderQueue.push(n)
  }
  void processQueue()
}

async function processQueue() {
  if (processing) return
  processing = true
  const ver = renderVersion
  while (renderQueue.length > 0) {
    if (ver !== renderVersion) break
    const n = renderQueue.shift()!
    // 离开窗口的任务不生成
    if (!visibleItems.value.includes(n)) continue
    if (bitmapCache.has(n)) continue
    await renderOne(n, ver)
    // 让出主线程，保证主视图滚动 / 缩放交互流畅
    await new Promise((r) => setTimeout(r, 0))
  }
  processing = false
}

async function renderOne(n: number, ver: number) {
  const state = getState(n)
  if (state.status === 'loading' || state.status === 'success') return
  const dim = props.pageBaseDims.get(n)
  if (!dim) {
    // 尺寸尚未预计算完成（理论上挂载时已就绪）：保持 pending，下一轮可见性变化时再入队
    return
  }
  state.status = 'loading'
  state.error = ''
  try {
    const page = await props.pdfDoc.getPage(n)
    if (ver !== renderVersion) return // 期间已被缩放 / 换文档作废
    const { canvas } = await renderPageThumbnail(
      page, dim.baseWidth, dim.baseHeight, THUMB_WIDTH,
    )
    if (ver !== renderVersion) return // 已被缩放 / 换文档作废
    bitmapCache.set(n, canvas)
    const idx = bitmapOrder.indexOf(n)
    if (idx !== -1) bitmapOrder.splice(idx, 1)
    bitmapOrder.push(n)
    evictBitmaps()
    state.status = 'success'
    state.error = ''
    attachCanvas(n)
  } catch (e) {
    if (ver !== renderVersion) return
    state.status = 'error'
    state.error = e instanceof Error ? e.message : String(e)
  }
}

/* ---- Canvas 挂载：位图在离屏缓存中生成，进入窗口后挂到当前节点 ---- */
const itemRefs = new Map<number, HTMLDivElement>()
function setItemRef(el: HTMLDivElement | null, n: number) {
  if (el) itemRefs.set(n, el)
  else itemRefs.delete(n)
}

function attachCanvas(n: number) {
  const canvas = bitmapCache.get(n)
  const host = itemRefs.get(n)
  if (!canvas || !host) return
  if (host.firstChild !== canvas) {
    host.innerHTML = ''
    host.appendChild(canvas)
  }
  canvas.style.width = '100%'
  canvas.style.height = '100%'
  canvas.style.display = 'block'
}

function attachAll() {
  for (const n of visibleItems.value) attachCanvas(n)
}

/* ---- 当前列表视口中心对应的页码（用于生成优先级） ---- */
const centerPage = ref(1)
function computeCenterPage() {
  const list = listRef.value
  if (!list) return
  const center = list.scrollTop + list.clientHeight / 2
  let acc = 0
  for (let n = 1; n <= props.totalPages; n++) {
    const h = itemHeight(n)
    if (acc + h >= center) { centerPage.value = n; return }
    acc += h
  }
  centerPage.value = props.totalPages
}

/* ---- 列表自身滚动（rAF 节流） ---- */
let listRafId: number | null = null
function onListScroll() {
  if (listRafId !== null) return
  listRafId = requestAnimationFrame(() => {
    listRafId = null
    const list = listRef.value
    if (!list) return
    scrollTop.value = list.scrollTop
    viewportH.value = list.clientHeight
    computeCenterPage()
  })
}

/** 用户主动操作缩略图列表后短时间内暂停“跟随当前页”，避免抢走用户的滚动位置 */
let lastUserInteract = 0
function onUserInteract() {
  lastUserInteract = Date.now()
}

/* ---- 可见窗口变化：挂回位图并补排队（pageBaseDims 在挂载前已预计算完成） ---- */
watch(
  visibleItems,
  () => {
    attachAll()
    enqueueVisible()
  },
  { flush: 'post' },
)

/* ---- 主视图当前页变化：高亮 + 把当前页缩略图滚动进视野 ---- */
watch(
  () => props.currentPage,
  (n) => {
    if (collapsed.value) return
    if (Date.now() - lastUserInteract < 1500) return // 用户正在自己翻阅缩略图
    const list = listRef.value
    if (!list) return
    const top = itemTop(n - 1)
    const targetTop = top - Math.max(0, (list.clientHeight - itemHeight(n)) / 2)
    // 已在可见区域内则不强制滚动
    if (top >= list.scrollTop && top + itemHeight(n) <= list.scrollTop + list.clientHeight) return
    list.scrollTo({ top: targetTop, behavior: 'smooth' })
  },
)

/* ---- 缩放：缩略图必须与页面最新外观保持一致，全部位图作废重生成 ---- */
watch(
  () => props.scale,
  () => {
    renderVersion++
    renderQueue.length = 0
    bitmapCache.clear()
    bitmapOrder.length = 0
    for (const [, st] of states) {
      st.status = 'pending'
      st.error = ''
    }
    attachAll()
    enqueueVisible()
  },
)

/* ---- 点击缩略图：跳转主视图（同时恢复跟随） ---- */
function onClickItem(n: number) {
  lastUserInteract = 0
  emit('navigate', n)
}

function retry(n: number) {
  const st = getState(n)
  st.status = 'pending'
  st.error = ''
  enqueueVisible()
}

let resizeObserver: ResizeObserver | null = null

onMounted(() => {
  renderVersion++
  seedStates()
  const list = listRef.value
  if (list) {
    viewportH.value = list.clientHeight
    scrollTop.value = list.scrollTop
    computeCenterPage()
    resizeObserver = new ResizeObserver(() => {
      viewportH.value = list.clientHeight
    })
    resizeObserver.observe(list)
  }
  attachAll()
  enqueueVisible()
})

onUnmounted(() => {
  renderVersion++
  renderQueue.length = 0
  bitmapCache.clear()
  bitmapOrder.length = 0
  itemRefs.clear()
  if (listRafId !== null) cancelAnimationFrame(listRafId)
  resizeObserver?.disconnect()
})
</script>

<style lang="scss" scoped>
/* ---- 折叠态窄轨道 ---- */
.thumb-rail {
  width: 24px; flex-shrink: 0;
  background: var(--card-bg);
  border-right: 1px solid var(--border-color);
  display: flex; flex-direction: column; align-items: center;
  padding-top: 12px; gap: 10px; cursor: pointer;
  transition: background 0.2s;
  &:hover { background: var(--bg-color); }
  &__expand { color: var(--text-secondary); flex-shrink: 0; }
  &__text {
    writing-mode: vertical-rl;
    font-size: var(--font-size-sm); color: var(--text-tertiary);
    letter-spacing: 2px; user-select: none;
  }
}

/* ---- 展开态面板 ---- */
.thumb-panel {
  width: 176px; flex-shrink: 0;
  background: var(--card-bg);
  border-right: 1px solid var(--border-color);
  display: flex; flex-direction: column;
  &__header {
    display: flex; align-items: center; gap: 8px;
    height: 40px; padding: 0 10px 0 14px;
    border-bottom: 1px solid var(--border-color);
    flex-shrink: 0;
  }
  &__title {
    font-size: var(--font-size-base); font-weight: 600;
    color: var(--text-primary); flex: 1;
  }
  &__count {
    font-size: var(--font-size-sm); color: var(--text-tertiary);
    font-variant-numeric: tabular-nums;
  }
  &__collapse {
    display: flex; align-items: center; justify-content: center;
    width: 24px; height: 24px; padding: 0;
    border: none; background: transparent;
    color: var(--text-tertiary); cursor: pointer;
    border-radius: var(--radius-sm); transition: all 0.2s;
    &:hover { background: var(--bg-color); color: var(--text-primary); }
  }
  &__list {
    flex: 1; overflow-y: auto; overflow-x: hidden;
    position: relative;
  }
}

/* ---- 单个缩略图项（绝对定位，由 transform 排布） ---- */
.thumb-item {
  position: absolute; left: 0; right: 0;
  padding: 10px 12px 0;
  display: flex; flex-direction: column; align-items: center;
  cursor: pointer;
  &__paper {
    position: relative;
    width: 128px;
    background: #fff;
    border: 1px solid var(--border-color);
    border-radius: 2px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);
    overflow: hidden;
    transition: border-color 0.2s, box-shadow 0.2s;
  }
  &__canvas-host {
    position: absolute; inset: 0;
    line-height: 0;
    canvas { display: block; width: 100%; height: 100%; }
  }
  &__number {
    margin-top: 6px;
    font-size: var(--font-size-sm); line-height: 18px;
    color: var(--text-tertiary);
    font-variant-numeric: tabular-nums; user-select: none;
    &--active { color: var(--primary-color); font-weight: 600; }
  }
  &:hover .thumb-item__paper {
    border-color: var(--primary-hover);
    box-shadow: 0 2px 8px rgba(22, 119, 255, 0.25);
  }
  &--active .thumb-item__paper {
    border: 2px solid var(--primary-color);
    box-shadow: 0 0 0 2px rgba(22, 119, 255, 0.18);
  }
}

/* ---- 占位：等待 / 生成中 / 失败 ---- */
.thumb-item__placeholder {
  display: flex; flex-direction: column; align-items: center; justify-content: center;
  gap: 6px; width: 100%; height: 100%;
  padding: 12px 8px;
  background: #fafafa; color: var(--text-tertiary);
  text-align: center;
  svg { flex-shrink: 0; }
  &-text {
    font-size: var(--font-size-sm); color: inherit;
  }
  &-reason {
    font-size: 11px; line-height: 1.4; color: var(--text-tertiary);
    word-break: break-word;
    display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical;
    overflow: hidden;
  }
  &--error {
    background: #fff2f0; color: #cf1322;
    .thumb-item__placeholder-reason { color: #cf6b60; }
  }
}
.thumb-item__spinner {
  width: 22px; height: 22px;
  border: 2px solid var(--border-color);
  border-top-color: var(--primary-color);
  border-radius: 50%;
  animation: thumb-spin 0.8s linear infinite;
}
.thumb-item__retry {
  margin-top: 2px;
  padding: 2px 14px;
  border: 1px solid var(--border-color); border-radius: var(--radius-sm);
  background: var(--card-bg); color: var(--primary-color);
  font-size: var(--font-size-sm); cursor: pointer;
  &:hover { border-color: var(--primary-color); background: #e6f4ff; }
}
@keyframes thumb-spin { to { transform: rotate(360deg); } }
</style>
