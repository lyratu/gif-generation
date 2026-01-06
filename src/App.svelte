<script lang="ts">
  import { flip } from 'svelte/animate'
  import { dndzone, type DndEvent } from 'svelte-dnd-action'
  import GIF from 'gif.js'
  import { saveAs } from 'file-saver'

  // --- 类型定义 ---
  interface Frame {
    id: number
    src: string
  }
  interface GridConfig {
    rows: number
    cols: number
    offsetX: number
    offsetY: number
    gapX: number
    gapY: number
  }
  interface TransformState {
    x: number
    y: number
    k: number
  }

  // --- 状态管理 ---
  let config = $state<GridConfig>({
    rows: 4,
    cols: 4,
    offsetX: 0,
    offsetY: 0,
    gapX: 0,
    gapY: 0,
  })

  // 播放倍率
  const BASE_FPS = 8
  let playRate = $state<number>(1.0)
  let fps = $derived(Math.max(1, Math.round(BASE_FPS * playRate)))

  let exportScale = $state<number>(1)
  let isPlaying = $state<boolean>(false)
  let isExporting = $state<boolean>(false)
  let viewMode = $state<'edit' | 'preview'>('edit')

  let frames = $state<Frame[]>([])
  let currentFrameIndex = $state<number>(0)
  let uploadedImage = $state<HTMLImageElement | null>(null)
  let exportQuality = $state<boolean>(true)

  // 变换状态
  let editTransform = $state<TransformState>({ x: 0, y: 0, k: 1 })
  let previewTransform = $state<TransformState>({ x: 0, y: 0, k: 1 })
  let currentTransform = $derived(viewMode === 'edit' ? editTransform : previewTransform)

  // 🟢 拖拽控制状态
  // 默认为 true (禁用拖拽，允许原生滚动)
  // 只有按住手柄时才变为 false
  let dragDisabled = $state<boolean>(true)
  let isDndActive = $state<boolean>(false)

  // 🟢 边缘滚动相关
  let scrollContainer = $state<HTMLDivElement | undefined>(undefined)
  let pointerPos = { x: 0, y: 0 }
  let autoScrollRafId: number | null = null

  const setTransform = (newT: TransformState) => {
    if (viewMode === 'edit') editTransform = newT
    else previewTransform = newT
  }

  let isCanvasDragging = false
  let startPos = { x: 0, y: 0 }
  let lastDist = 0
  let canvasContainer: HTMLDivElement | undefined
  let debounceTimer: ReturnType<typeof setTimeout> | undefined
  let playTimer: ReturnType<typeof setTimeout> | undefined

  // --- 核心逻辑 ---

  const handleImageUpload = (e: Event) => {
    const input = e.target as HTMLInputElement
    if (!input.files || input.files.length === 0) return
    const file = input.files[0]
    const reader = new FileReader()
    reader.onload = (event) => {
      if (!event.target?.result) return
      const img = new Image()
      img.onload = () => {
        uploadedImage = img
        config.offsetX = 0
        config.offsetY = 0
        config.gapX = 0
        config.gapY = 0
        viewMode = 'edit'
        editTransform = { x: 0, y: 0, k: 1 }
        previewTransform = { x: 0, y: 0, k: 1 }
      }
      img.src = event.target.result as string
    }
    reader.readAsDataURL(file)
  }

  const resetCurrentView = () => {
    setTransform({ x: 0, y: 0, k: 1 })
  }

  const sliceImage = () => {
    if (!uploadedImage) return
    const img = uploadedImage
    const { rows, cols, offsetX, offsetY, gapX, gapY } = config
    if (rows <= 0 || cols <= 0) return
    const safeWidth = img.width - offsetX * 2
    const safeHeight = img.height - offsetY * 2
    if (safeWidth <= 0 || safeHeight <= 0) return
    const cellWidth = Math.floor((safeWidth - (cols - 1) * gapX) / cols)
    const cellHeight = Math.floor((safeHeight - (rows - 1) * gapY) / rows)
    if (cellWidth <= 0 || cellHeight <= 0) return
    const tempCanvas = document.createElement('canvas')
    tempCanvas.width = cellWidth
    tempCanvas.height = cellHeight
    const ctx = tempCanvas.getContext('2d', { willReadFrequently: true })
    if (!ctx) return
    const newFrames: Frame[] = []
    for (let y = 0; y < rows; y++) {
      for (let x = 0; x < cols; x++) {
        const srcX = offsetX + x * (cellWidth + gapX)
        const srcY = offsetY + y * (cellHeight + gapY)
        ctx.clearRect(0, 0, cellWidth, cellHeight)
        ctx.drawImage(img, srcX, srcY, cellWidth, cellHeight, 0, 0, cellWidth, cellHeight)
        newFrames.push({
          id: Date.now() + Math.random(),
          src: tempCanvas.toDataURL('image/png'),
        })
      }
    }
    frames = newFrames
    if (currentFrameIndex >= frames.length) currentFrameIndex = 0
  }

  const getGridRects = () => {
    if (!uploadedImage) return []
    const { rows, cols, offsetX, offsetY, gapX, gapY } = config
    if (rows > 50 || cols > 50) return []
    const safeWidth = uploadedImage.width - offsetX * 2
    const safeHeight = uploadedImage.height - offsetY * 2
    const cellW = (safeWidth - (cols - 1) * gapX) / cols
    const cellH = (safeHeight - (rows - 1) * gapY) / rows
    if (cellW <= 0 || cellH <= 0) return []
    let rects = []
    for (let y = 0; y < rows; y++) {
      for (let x = 0; x < cols; x++) {
        rects.push({
          x: offsetX + x * (cellW + gapX),
          y: offsetY + y * (cellH + gapY),
          w: cellW,
          h: cellH,
        })
      }
    }
    return rects
  }

  // --- Canvas Zoom & Pan ---

  const handleWheel = (e: WheelEvent) => {
    if (!uploadedImage) return
    e.preventDefault()
    const currentT = currentTransform
    const scaleFactor = 0.1
    const delta = e.deltaY > 0 ? 1 - scaleFactor : 1 + scaleFactor
    const newK = Math.min(Math.max(0.1, currentT.k * delta), 10)
    if (!canvasContainer) return
    const rect = canvasContainer.getBoundingClientRect()
    const mouseX = e.clientX - rect.left
    const mouseY = e.clientY - rect.top
    const newX = mouseX - (mouseX - currentT.x) * (newK / currentT.k)
    const newY = mouseY - (mouseY - currentT.y) * (newK / currentT.k)
    setTransform({ k: newK, x: newX, y: newY })
  }

  const handleMouseDown = (e: MouseEvent) => {
    if (!uploadedImage) return
    const currentT = currentTransform
    isCanvasDragging = true
    startPos = { x: e.clientX - currentT.x, y: e.clientY - currentT.y }
  }

  const handleMouseMove = (e: MouseEvent) => {
    if (!isCanvasDragging) return
    e.preventDefault()
    setTransform({ ...currentTransform, x: e.clientX - startPos.x, y: e.clientY - startPos.y })
  }

  const handleMouseUp = () => {
    isCanvasDragging = false
  }

  const handleTouchStart = (e: TouchEvent) => {
    if (!uploadedImage) return
    const currentT = currentTransform
    if (e.touches.length === 1) {
      isCanvasDragging = true
      startPos = { x: e.touches[0].clientX - currentT.x, y: e.touches[0].clientY - currentT.y }
    } else if (e.touches.length === 2) {
      isCanvasDragging = false
      lastDist = Math.hypot(e.touches[0].clientX - e.touches[1].clientX, e.touches[0].clientY - e.touches[1].clientY)
    }
  }

  const handleTouchMove = (e: TouchEvent) => {
    if (!uploadedImage) return
    const currentT = currentTransform
    if (e.touches.length === 1 && isCanvasDragging) {
      setTransform({ ...currentT, x: e.touches[0].clientX - startPos.x, y: e.touches[0].clientY - startPos.y })
    } else if (e.touches.length === 2) {
      const dist = Math.hypot(e.touches[0].clientX - e.touches[1].clientX, e.touches[0].clientY - e.touches[1].clientY)
      if (lastDist > 0) {
        const delta = dist / lastDist
        const newK = Math.min(Math.max(0.1, currentT.k * delta), 10)
        const centerX = (e.touches[0].clientX + e.touches[1].clientX) / 2
        const centerY = (e.touches[0].clientY + e.touches[1].clientY) / 2
        if (canvasContainer) {
          const rect = canvasContainer.getBoundingClientRect()
          const relCenterX = centerX - rect.left
          const relCenterY = centerY - rect.top
          const newX = relCenterX - (relCenterX - currentT.x) * (newK / currentT.k)
          const newY = relCenterY - (relCenterY - currentT.y) * (newK / currentT.k)
          setTransform({ k: newK, x: newX, y: newY })
        }
      }
      lastDist = dist
    }
  }

  // --- 🟢 关键：手柄拖拽与边缘滚动逻辑 ---

  const trackPointer = (x: number, y: number) => {
    pointerPos = { x, y }
  }

  // 边缘自动滚动
  const processAutoScroll = () => {
    if (!isDndActive || !scrollContainer) return

    const rect = scrollContainer.getBoundingClientRect()
    const { x, y } = pointerPos
    const threshold = 50
    const maxSpeed = 15

    let scrollX = 0
    let scrollY = 0

    // X轴
    if (x < rect.left + threshold) {
      const intensity = Math.max(0, (rect.left + threshold - x) / threshold)
      scrollX = -maxSpeed * intensity
    } else if (x > rect.right - threshold) {
      const intensity = Math.max(0, (x - (rect.right - threshold)) / threshold)
      scrollX = maxSpeed * intensity
    }

    // Y轴
    if (y < rect.top + threshold) {
      const intensity = Math.max(0, (rect.top + threshold - y) / threshold)
      scrollY = -maxSpeed * intensity
    } else if (y > rect.bottom - threshold) {
      const intensity = Math.max(0, (y - (rect.bottom - threshold)) / threshold)
      scrollY = maxSpeed * intensity
    }

    if (scrollX !== 0) scrollContainer.scrollLeft += scrollX
    if (scrollY !== 0) scrollContainer.scrollTop += scrollY

    autoScrollRafId = requestAnimationFrame(processAutoScroll)
  }

  // 🔴 核心：手柄交互逻辑
  // 仅当用户接触手柄时，解锁拖拽权限
  const startDragHandle = (e: TouchEvent | MouseEvent) => {
    dragDisabled = false
  }

  // PC端鼠标悬停逻辑保持不变，PC用户习惯点哪里都能拖
  const enableDragOnHover = () => {
    // 检测是否为触摸设备环境，如果是触摸设备，hover 不应该触发解锁，必须用手柄
    // 这里简单判断：如果屏幕宽度大于 md 断点，且是指针设备，则允许
    if (window.matchMedia('(min-width: 768px)').matches) {
      dragDisabled = false
    }
  }
  const disableDragOnLeave = () => {
    if (window.matchMedia('(min-width: 768px)').matches) {
      dragDisabled = true
    }
  }

  const handleDndConsider = (e: CustomEvent<DndEvent<Frame>>) => {
    frames = e.detail.items

    // 激活自动滚动监测
    if (!isDndActive) {
      isDndActive = true
      if (autoScrollRafId) cancelAnimationFrame(autoScrollRafId)
      autoScrollRafId = requestAnimationFrame(processAutoScroll)
    }
  }

  const handleDndFinalize = (e: CustomEvent<DndEvent<Frame>>) => {
    frames = e.detail.items

    // 🔴 核心：拖拽结束，立即恢复滚动模式
    isDndActive = false
    dragDisabled = true

    if (autoScrollRafId) {
      cancelAnimationFrame(autoScrollRafId)
      autoScrollRafId = null
    }
  }

  // --- Animation & Export ---

  const togglePlay = () => {
    isPlaying = !isPlaying
    if (isPlaying) {
      viewMode = 'preview'
      runAnimation()
    } else {
      clearTimeout(playTimer)
    }
  }

  const runAnimation = () => {
    if (!isPlaying) return
    playTimer = setTimeout(() => {
      if (frames.length > 0) currentFrameIndex = (currentFrameIndex + 1) % frames.length
      runAnimation()
    }, 1000 / fps)
  }

  const exportGif = async () => {
    if (frames.length === 0) return
    isExporting = true
    isPlaying = false
    clearTimeout(playTimer)
    try {
      const gif = new GIF({ workers: 2, quality: exportQuality ? 10 : 20, workerScript: '/gif.worker.js' })
      const loadedImages = await Promise.all(
        frames.map((f) => {
          return new Promise<HTMLImageElement>((r) => {
            const i = new Image()
            i.src = f.src
            i.onload = () => r(i)
          })
        })
      )
      loadedImages.forEach((img) => {
        const width = Math.floor(img.width * exportScale)
        const height = Math.floor(img.height * exportScale)
        const canvas = document.createElement('canvas')
        canvas.width = width
        canvas.height = height
        const ctx = canvas.getContext('2d')
        if (ctx) {
          ctx.imageSmoothingEnabled = false
          ctx.drawImage(img, 0, 0, width, height)
          gif.addFrame(canvas, { delay: 1000 / fps })
        }
      })
      gif.on('finished', (blob) => {
        saveAs(blob, `sprite-${Date.now()}_${exportScale}x.gif`)
        isExporting = false
      })
      gif.render()
    } catch (e) {
      console.error(e)
      alert('导出失败，请检查 gif.worker.js 是否存在')
      isExporting = false
    }
  }

  $effect(() => {
    const deps = [uploadedImage, config.rows, config.cols, config.offsetX, config.offsetY, config.gapX, config.gapY]
    if (uploadedImage) {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        sliceImage()
      }, 300)
    }
  })

  $effect(() => {
    if (isPlaying) {
      clearTimeout(playTimer)
      runAnimation()
    }
  })
</script>

<svelte:window
  onmousemove={(e) => trackPointer(e.clientX, e.clientY)}
  ontouchmove={(e) => trackPointer(e.touches[0].clientX, e.touches[0].clientY)}
  onmouseup={() => {
    if (isCanvasDragging) isCanvasDragging = false
  }}
/>

<div class="h-dvh w-full bg-slate-50 text-slate-800 flex flex-col md:flex-row overflow-hidden font-sans">
  <aside class="order-3 md:order-none w-full md:w-64 h-32 md:h-full bg-white md:border-r border-t md:border-t-0 border-slate-200 flex flex-col shadow-sm z-30 shrink-0">
    <div class="hidden md:flex p-4 border-b border-slate-100 bg-slate-50 h-16 items-center justify-between shrink-0">
      <h2 class="font-bold text-slate-700">切片结果 ({frames.length})</h2>
    </div>

    {#if frames.length > 0}
      <div
        bind:this={scrollContainer}
        class="flex-1 custom-scrollbar pb-8 select-none overflow-x-auto md:overflow-x-hidden md:overflow-y-auto {!dragDisabled ? 'touch-none' : 'touch-pan-x md:touch-pan-y'}"
      >
        <section
          class="flex flex-row md:flex-col gap-2 items-center md:items-stretch min-w-max md:min-w-full min-h-full md:pb-0 pr-10 md:pr-0"
          use:dndzone={{
            items: frames,
            flipDurationMs: 0,
            dragDisabled,
            dropTargetStyle: { outline: '2px dashed #3b82f6', borderRadius: '8px' },
          }}
          onconsider={handleDndConsider}
          onfinalize={handleDndFinalize}
        >
          {#each frames as frame, index (frame.id)}
            <div
              class="shrink-0 flex items-center gap-2 p-2 bg-white border border-slate-100 rounded-lg relative shadow-sm h-20 w-20 md:h-auto md:w-full group transition-all duration-200"
              class:border-blue-400={!dragDisabled}
              animate:flip={{ duration: 200 }}
              onmouseenter={enableDragOnHover}
              onmouseleave={disableDragOnLeave}
              role="listitem"
            >
              <span class="text-xs text-slate-300 w-4 hidden md:block shrink-0">{index + 1}</span>

              <div class="w-16 h-16 md:w-12 md:h-12 bg-slate-100 border border-slate-200 relative shrink-0 overflow-hidden rounded-sm pointer-events-none">
                <img src={frame.src} alt="" class="w-full h-full object-contain pixelated" />
              </div>

              <button
                class="absolute inset-y-0 right-0 w-10 md:hidden z-10 flex items-center justify-center touch-none active:bg-slate-50/80 rounded-r-lg cursor-grab active:cursor-grabbing"
                ontouchstart={startDragHandle}
                onmousedown={startDragHandle}
                aria-label="Drag handle"
              >
                <div class="bg-black/5 text-slate-400 p-1.5 rounded backdrop-blur-[1px]">
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"
                    ><path d="M5 9l0 6" /><path d="M9 9l0 6" /><path d="M13 9l0 6" /><path d="M17 9l0 6" /></svg
                  >
                </div>
              </button>

              {#if index === currentFrameIndex}
                <div class="absolute -top-1 -right-1 md:top-auto md:right-2 md:relative w-3 h-3 bg-green-500 rounded-full animate-pulse ring-2 ring-white z-20 pointer-events-none"></div>
              {/if}
            </div>
          {/each}
        </section>
      </div>
    {:else}
      <div class="flex-1 flex flex-col items-center justify-center text-slate-300 p-4 select-none min-h-[100px]">
        <div class="text-4xl mb-2 opacity-50">📂</div>
        <p class="text-xs text-center">上传图片后<br />生成切片</p>
      </div>
    {/if}
  </aside>

  <main class="order-1 md:order-none flex-1 md:flex-auto h-[45%] md:h-full bg-slate-100 flex flex-col relative overflow-hidden min-w-0 z-10 border-b md:border-b-0 border-slate-200">
    <header class="h-10 md:h-16 bg-white/80 backdrop-blur border-b border-slate-200 flex items-center justify-between px-4 md:px-6 shadow-sm z-20 shrink-0">
      <div class="flex items-center gap-4">
        <label class="btn-primary cursor-pointer px-3 py-1.5 md:px-4 md:py-2 bg-blue-600 text-white rounded-lg text-xs md:text-sm font-medium hover:bg-blue-700 transition flex items-center gap-2">
          <svg xmlns="http://www.w3.org/2000/svg" class="w-4 h-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"
            ><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" /><polyline points="17 8 12 3 7 8" /><line x1="12" y1="3" x2="12" y2="15" /></svg
          >
          <span class="hidden md:inline">上传 Sprite 图</span>
          <span class="md:hidden">上传</span>
          <input type="file" accept="image/*" class="hidden" onchange={handleImageUpload} />
        </label>
      </div>

      <div class="flex bg-slate-200 p-1 rounded-lg">
        <button
          class="px-3 md:px-4 py-1.5 text-xs font-bold rounded-md transition-all {viewMode === 'edit' ? 'bg-white text-blue-600 shadow-sm' : 'text-slate-500 hover:text-slate-700'}"
          onclick={() => {
            viewMode = 'edit'
            isPlaying = false
          }}
        >
          📐 编辑
        </button>
        <button
          class="px-3 md:px-4 py-1.5 text-xs font-bold rounded-md transition-all {viewMode === 'preview' ? 'bg-white text-blue-600 shadow-sm' : 'text-slate-500 hover:text-slate-700'}"
          onclick={() => (viewMode = 'preview')}
        >
          🎬 预览
        </button>
      </div>
    </header>

    <div
      bind:this={canvasContainer}
      class="flex-1 relative overflow-hidden bg-slate-100 cursor-grab active:cursor-grabbing touch-none select-none"
      class:cursor-grab={!isCanvasDragging}
      class:cursor-grabbing={isCanvasDragging}
      onwheel={handleWheel}
      onmousedown={handleMouseDown}
      onmousemove={handleMouseMove}
      onmouseup={handleMouseUp}
      onmouseleave={handleMouseUp}
      ontouchstart={handleTouchStart}
      ontouchmove={handleTouchMove}
      ontouchend={handleMouseUp}
    >
      <div
        class="absolute inset-0 bg-[url('data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIyMCIgaGVpZ2h0PSIyMCIgZmlsbD0iI2YxZjVmOSI+PHJlY3Qgd2lkdGg9IjEwIiBoZWlnaHQ9IjEwIiBmaWxsPSIjZmZmIi8+PHJlY3QgeD0iMTAiIHk9IjEwIiB3aWR0aD0iMTAiIGhlaWdodD0iMTAiIGZpbGw9IiNmZmYiLz48L3N2Zz4=')] opacity-50"
      ></div>

      <div
        class="absolute top-0 left-0 w-full h-full flex items-center justify-center transition-transform duration-75 ease-out origin-top-left"
        style="transform: translate({currentTransform.x}px, {currentTransform.y}px) scale({currentTransform.k});"
      >
        {#if !uploadedImage}
          <div class="text-slate-400 text-center select-none scale-[1/currentTransform.k]">
            <div class="text-6xl mb-4 opacity-20">📐</div>
            <p>请上传素材图</p>
          </div>
        {:else if viewMode === 'edit'}
          <div class="relative shadow-2xl bg-white ring-1 ring-slate-900/5 select-none inline-block">
            <img src={uploadedImage.src} alt="Source" class="block pixelated max-w-none pointer-events-none" />
            <svg class="absolute inset-0 w-full h-full pointer-events-none" viewBox="0 0 {uploadedImage.width} {uploadedImage.height}" preserveAspectRatio="none" xmlns="http://www.w3.org/2000/svg">
              <defs>
                <pattern id="gridPattern" width="10" height="10" patternUnits="userSpaceOnUse">
                  <path d="M 10 0 L 0 0 0 10" fill="none" stroke="white" stroke-width="0.5" />
                </pattern>
              </defs>
              <rect width="100%" height="100%" fill="black" fill-opacity="0.6" />
              {#each getGridRects() as r}
                <rect x={r.x} y={r.y} width={r.w} height={r.h} fill="none" stroke="#ef4444" stroke-width="1" vector-effect="non-scaling-stroke" />
                <rect x={r.x} y={r.y} width={r.w} height={r.h} fill="white" fill-opacity="0.05" />
              {/each}
            </svg>
          </div>
        {:else}
          <div class="relative shadow-2xl rounded-lg overflow-hidden bg-white ring-1 ring-slate-900/5 select-none group">
            {#if frames.length > 0}
              <img src={frames[currentFrameIndex].src} alt="" class="block pixelated pointer-events-none max-w-[85vw] max-h-[45vh] md:max-w-none md:max-h-none" />
            {/if}
          </div>
        {/if}
      </div>

      {#if uploadedImage}
        <button
          class="absolute bottom-4 right-4 bg-white/90 backdrop-blur shadow-md border border-slate-200 text-slate-600 px-3 py-1.5 rounded-lg text-xs font-bold hover:bg-white transition z-50"
          onclick={resetCurrentView}
        >
          复位视图 ({Math.round(currentTransform.k * 100)}%)
        </button>
      {/if}
    </div>

    {#if viewMode === 'preview'}
      <div class="bg-white p-1 border-t border-slate-200 flex items-center gap-4 shrink-0 z-20 animate-in slide-in-from-bottom-2">
        <button onclick={togglePlay} class="w-6 h-6 rounded-full bg-blue-600 text-white flex items-center justify-center hover:bg-blue-700 transition">
          {isPlaying ? '⏸' : '▶'}
        </button>
        <input type="range" min="0" max={Math.max(0, frames.length - 1)} bind:value={currentFrameIndex} class="flex-1 h-1 bg-slate-200 rounded-lg appearance-none cursor-pointer accent-blue-600" />
      </div>
    {/if}
  </main>

  <aside class="order-2 md:order-none w-full md:w-80 flex-1 md:flex-auto bg-white md:border-l border-slate-200 flex flex-col shadow-sm z-20 overflow-hidden min-h-0 pb-safe">
    <div class="flex-1 custom-scrollbar overflow-y-auto p-4 md:p-6 space-y-6 md:space-y-8 pb-10">
      <section>
        <h3 class="text-sm font-bold text-slate-800 mb-3 flex items-center gap-2">
          <span class="w-1 h-4 bg-blue-600 rounded"></span> 基础网格
        </h3>
        <div class="grid grid-cols-2 gap-4">
          <div class="space-y-1">
            <label class="text-xs font-semibold text-slate-500">行数</label>
            <input
              type="number"
              min="1"
              max="20"
              bind:value={config.rows}
              class="w-full px-3 py-2 bg-slate-50 border border-slate-200 rounded-lg text-sm font-bold focus:ring-2 focus:ring-blue-500 outline-none"
            />
          </div>
          <div class="space-y-1">
            <label class="text-xs font-semibold text-slate-500">列数</label>
            <input
              type="number"
              min="1"
              max="20"
              bind:value={config.cols}
              class="w-full px-3 py-2 bg-slate-50 border border-slate-200 rounded-lg text-sm font-bold focus:ring-2 focus:ring-blue-500 outline-none"
            />
          </div>
        </div>
      </section>

      <hr class="border-slate-100" />

      <section class="space-y-4">
        <h3 class="text-sm font-bold text-slate-800 mb-2 flex items-center gap-2">
          <span class="w-1 h-4 bg-orange-500 rounded"></span>
          <span>微调</span>
          <span class="text-xs text-slate-400 font-normal ml-auto">移动滑块调整</span>
        </h3>

        <div class="space-y-3 bg-orange-50/50 p-3 rounded-xl border border-orange-100">
          <div class="flex justify-between text-xs text-slate-600 font-medium">
            <span>起始偏移</span>
            <span class="text-orange-600">X:{config.offsetX} / Y:{config.offsetY}</span>
          </div>
          <div class="space-y-3">
            <div class="flex items-center gap-3 w-full">
              <span class="text-[10px] text-slate-400 w-3 shrink-0">X</span>
              <input type="range" min="0" max="100" bind:value={config.offsetX} class="flex-1 min-w-0 h-2 bg-slate-200 rounded-lg appearance-none cursor-pointer accent-orange-500" />
            </div>
            <div class="flex items-center gap-3 w-full">
              <span class="text-[10px] text-slate-400 w-3 shrink-0">Y</span>
              <input type="range" min="0" max="100" bind:value={config.offsetY} class="flex-1 min-w-0 h-2 bg-slate-200 rounded-lg appearance-none cursor-pointer accent-orange-500" />
            </div>
          </div>
        </div>

        <div class="space-y-3 bg-indigo-50/50 p-3 rounded-xl border border-indigo-100">
          <div class="flex justify-between text-xs text-slate-600 font-medium">
            <span>边框间距</span>
            <span class="text-indigo-600">X:{config.gapX} / Y:{config.gapY}</span>
          </div>
          <div class="space-y-3">
            <div class="flex items-center gap-3 w-full">
              <span class="text-[10px] text-slate-400 w-3 shrink-0">X</span>
              <input type="range" min="0" max="50" bind:value={config.gapX} class="flex-1 min-w-0 h-2 bg-slate-200 rounded-lg appearance-none cursor-pointer accent-indigo-500" />
            </div>
            <div class="flex items-center gap-3 w-full">
              <span class="text-[10px] text-slate-400 w-3 shrink-0">Y</span>
              <input type="range" min="0" max="50" bind:value={config.gapY} class="flex-1 min-w-0 h-2 bg-slate-200 rounded-lg appearance-none cursor-pointer accent-indigo-500" />
            </div>
          </div>
        </div>
      </section>

      <hr class="border-slate-100" />

      <section class="space-y-4">
        <div class="space-y-2">
          <div class="flex justify-between items-center">
            <label class="text-sm font-medium text-slate-600">播放速度</label>
            <span class="text-xs font-bold text-blue-600 bg-blue-50 px-2 py-0.5 rounded">{playRate.toFixed(1)}x ({fps} FPS)</span>
          </div>
          <input type="range" min="0.5" max="8" step="0.5" bind:value={playRate} class="w-full h-2 bg-slate-200 rounded-lg appearance-none cursor-pointer accent-blue-600" />
        </div>

        <div class="space-y-2">
          <label class="text-sm font-medium text-slate-600">导出尺寸 (倍率)</label>
          <div class="flex gap-2 bg-slate-50 p-1 rounded-lg border border-slate-200">
            {#each [0.5, 1, 1.5, 2] as scale}
              <button
                class="flex-1 py-1.5 text-xs font-bold rounded transition-colors {exportScale === scale
                  ? 'bg-white text-blue-600 shadow-sm ring-1 ring-slate-200'
                  : 'text-slate-400 hover:text-slate-600'}"
                onclick={() => (exportScale = scale)}
              >
                {scale}x
              </button>
            {/each}
          </div>
        </div>

        <button class="w-full py-3 bg-slate-900 text-white rounded-lg font-bold hover:bg-slate-800 transition shadow-lg shadow-slate-500/20 active:scale-[0.98]" onclick={exportGif}>
          {#if isExporting}
            ⏳ 渲染中...
          {:else}
            📤 导出 GIF
          {/if}
        </button>
      </section>
    </div>
  </aside>
</div>

<style>
  @reference "tailwindcss";
  .pixelated {
    image-rendering: pixelated;
  }
  .pb-safe {
    padding-bottom: env(safe-area-inset-bottom, 20px);
  }

  .custom-scrollbar::-webkit-scrollbar {
    width: 6px;
    height: 6px;
  }
  .custom-scrollbar::-webkit-scrollbar-track {
    background: transparent;
  }
  .custom-scrollbar::-webkit-scrollbar-thumb {
    background-color: #cbd5e1;
    border-radius: 999px;
  }
  .custom-scrollbar::-webkit-scrollbar-thumb:hover {
    background-color: #94a3b8;
  }
</style>
