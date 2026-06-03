<script setup>
import { computed, onMounted, onUnmounted, ref } from 'vue'
import mapaGta from './assets/mapa-gta.webp'

const MIN_ZOOM = 0.2
const MAX_ZOOM = 8
const ZOOM_STEP = 0.15

const viewportRef = ref(null)
const mapNaturalSize = ref({ width: 0, height: 0 })
const viewportSize = ref({ width: 0, height: 0 })
const zoom = ref(1)
const offset = ref({ x: 0, y: 0 })
const isDragging = ref(false)
const dragStartPoint = ref({ x: 0, y: 0 })
const dragStartOffset = ref({ x: 0, y: 0 })
const activePointerId = ref(null)
const pointerCoords = ref({ x: null, y: null })

const mapStyle = computed(() => ({
  transform: `translate(${offset.value.x}px, ${offset.value.y}px) scale(${zoom.value})`,
}))

const coordLabel = computed(() => {
  if (pointerCoords.value.x === null || pointerCoords.value.y === null) {
    return 'X: -- | Y: --'
  }

  return `X: ${Math.round(pointerCoords.value.x)} | Y: ${Math.round(pointerCoords.value.y)}`
})

function clampOffset(nextOffset) {
  const scaledWidth = mapNaturalSize.value.width * zoom.value
  const scaledHeight = mapNaturalSize.value.height * zoom.value
  const { width: viewportWidth, height: viewportHeight } = viewportSize.value

  if (!scaledWidth || !scaledHeight || !viewportWidth || !viewportHeight) {
    return { x: 0, y: 0 }
  }

  const minX = scaledWidth > viewportWidth ? viewportWidth - scaledWidth : (viewportWidth - scaledWidth) / 2
  const maxX = scaledWidth > viewportWidth ? 0 : minX
  const minY = scaledHeight > viewportHeight ? viewportHeight - scaledHeight : (viewportHeight - scaledHeight) / 2
  const maxY = scaledHeight > viewportHeight ? 0 : minY

  return {
    x: Math.min(Math.max(nextOffset.x, minX), maxX),
    y: Math.min(Math.max(nextOffset.y, minY), maxY),
  }
}

function refreshViewportSize() {
  if (!viewportRef.value) {
    return
  }

  viewportSize.value = {
    width: viewportRef.value.clientWidth,
    height: viewportRef.value.clientHeight,
  }

  if (mapNaturalSize.value.width && mapNaturalSize.value.height) {
    offset.value = clampOffset(offset.value)
  }
}

function updatePointerCoords(event) {
  if (!viewportRef.value) {
    return
  }

  const rect = viewportRef.value.getBoundingClientRect()
  const x = (event.clientX - rect.left - offset.value.x) / zoom.value
  const y = (event.clientY - rect.top - offset.value.y) / zoom.value

  if (
    x < 0 ||
    y < 0 ||
    x > mapNaturalSize.value.width ||
    y > mapNaturalSize.value.height
  ) {
    pointerCoords.value = { x: null, y: null }
    return
  }

  pointerCoords.value = { x, y }
}

function onImageLoad(event) {
  mapNaturalSize.value = {
    width: event.target.naturalWidth,
    height: event.target.naturalHeight,
  }
  refreshViewportSize()
  applyInitialZoom()
}

function onPointerDown(event) {
  if (event.button !== 0) {
    return
  }

  isDragging.value = true
  activePointerId.value = event.pointerId
  dragStartPoint.value = { x: event.clientX, y: event.clientY }
  dragStartOffset.value = { ...offset.value }
  viewportRef.value?.setPointerCapture(event.pointerId)
}

function onPointerMove(event) {
  updatePointerCoords(event)

  if (!isDragging.value || event.pointerId !== activePointerId.value) {
    return
  }

  const deltaX = event.clientX - dragStartPoint.value.x
  const deltaY = event.clientY - dragStartPoint.value.y

  offset.value = clampOffset({
    x: dragStartOffset.value.x + deltaX,
    y: dragStartOffset.value.y + deltaY,
  })
  event.preventDefault()
}

function stopDragging() {
  isDragging.value = false
  activePointerId.value = null
}

function onPointerUp(event) {
  if (event.pointerId !== activePointerId.value) {
    return
  }

  viewportRef.value?.releasePointerCapture(event.pointerId)
  stopDragging()
}

function onPointerCancel(event) {
  if (event.pointerId !== activePointerId.value) {
    return
  }

  stopDragging()
}

function onMouseLeave() {
  pointerCoords.value = { x: null, y: null }
}

function setZoom(nextZoom) {
  if (!viewportRef.value) {
    return
  }

  const clampedZoom = Math.min(Math.max(nextZoom, MIN_ZOOM), MAX_ZOOM)
  const centerX = viewportSize.value.width / 2
  const centerY = viewportSize.value.height / 2
  const worldX = (centerX - offset.value.x) / zoom.value
  const worldY = (centerY - offset.value.y) / zoom.value

  zoom.value = clampedZoom
  offset.value = clampOffset({
    x: centerX - worldX * zoom.value,
    y: centerY - worldY * zoom.value,
  })
}

function applyInitialZoom() {
  if (
    !viewportSize.value.width ||
    !viewportSize.value.height ||
    !mapNaturalSize.value.width ||
    !mapNaturalSize.value.height
  ) {
    return
  }

  const fitZoom = Math.min(
    viewportSize.value.width / mapNaturalSize.value.width,
    viewportSize.value.height / mapNaturalSize.value.height,
  )

  zoom.value = Math.min(Math.max(fitZoom, MIN_ZOOM), MAX_ZOOM)
  offset.value = clampOffset({
    x: (viewportSize.value.width - mapNaturalSize.value.width * zoom.value) / 2,
    y: (viewportSize.value.height - mapNaturalSize.value.height * zoom.value) / 2,
  })
}

function zoomIn() {
  setZoom(zoom.value + ZOOM_STEP)
}

function zoomOut() {
  setZoom(zoom.value - ZOOM_STEP)
}

function getZoomStepWithModifiers(event) {
  if (event.shiftKey) {
    return ZOOM_STEP * 2
  }

  if (event.ctrlKey || event.metaKey) {
    return ZOOM_STEP * 0.8
  }

  if (event.altKey) {
    return ZOOM_STEP * 0.5
  }

  return ZOOM_STEP
}

function onViewportWheel(event) {
  const direction = event.deltaY < 0 ? 1 : -1
  const zoomStep = getZoomStepWithModifiers(event)
  setZoom(zoom.value + direction * zoomStep)
}

function moveByKeyboard(deltaX, deltaY) {
  offset.value = clampOffset({
    x: offset.value.x + deltaX,
    y: offset.value.y + deltaY,
  })
}

function onViewportKeydown(event) {
  const step = 30

  if (event.key === 'ArrowLeft') {
    moveByKeyboard(step, 0)
    event.preventDefault()
  } else if (event.key === 'ArrowRight') {
    moveByKeyboard(-step, 0)
    event.preventDefault()
  } else if (event.key === 'ArrowUp') {
    moveByKeyboard(0, step)
    event.preventDefault()
  } else if (event.key === 'ArrowDown') {
    moveByKeyboard(0, -step)
    event.preventDefault()
  }
}

onMounted(() => {
  refreshViewportSize()
  window.addEventListener('resize', refreshViewportSize)
})

onUnmounted(() => {
  window.removeEventListener('resize', refreshViewportSize)
})
</script>

<template>
  <main class="home">
    <div class="hud">
      <p class="coords">{{ coordLabel }}</p>
    </div>

    <section
      ref="viewportRef"
      class="map-viewport"
      :class="{ 'is-dragging': isDragging }"
      aria-label="Área do mapa interativo"
      tabindex="0"
      @keydown="onViewportKeydown"
      @pointerdown="onPointerDown"
      @pointermove="onPointerMove"
      @pointerup="onPointerUp"
      @pointercancel="onPointerCancel"
      @pointerleave="onMouseLeave"
      @wheel.prevent="onViewportWheel"
    >
      <img
        class="home-image"
        :style="mapStyle"
        :src="mapaGta"
        alt="Mapa interativo GTA"
        draggable="false"
        @load="onImageLoad"
      />
    </section>

    <div class="zoom-controls">
      <button
        type="button"
        aria-label="Diminuir zoom"
        @click="zoomOut"
        :disabled="zoom <= MIN_ZOOM"
      >
        -
      </button>
      <button
        type="button"
        aria-label="Aumentar zoom"
        @click="zoomIn"
        :disabled="zoom >= MAX_ZOOM"
      >
        +
      </button>
    </div>
  </main>
</template>

<style scoped>
.home {
  min-height: 100vh;
  display: grid;
  grid-template-rows: auto 1fr;
  gap: 0.75rem;
  margin: 0;
  padding: 0.75rem;
  box-sizing: border-box;
  background: #2f8fbd;
}

.hud {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 0.75rem;
  color: #ffffff;
}

.zoom-controls {
  position: fixed;
  right: 1rem;
  bottom: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
  padding: 0.35rem;
  border-radius: 0.5rem;
  background: #000000;
  z-index: 10;
}

.zoom-controls button {
  border: 1px solid #ffffff;
  border-radius: 0.35rem;
  width: 2.2rem;
  height: 2rem;
  font-size: 1.2rem;
  font-weight: 700;
  color: #ffffff;
  background: transparent;
  cursor: pointer;
}

.zoom-controls button:disabled {
  cursor: not-allowed;
  opacity: 0.55;
}

.coords {
  margin: 0;
  font-size: 0.95rem;
  font-weight: 600;
}

.map-viewport {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 0;
  overflow: hidden;
  border: 6px solid #2f8fbd;
  border-radius: 0.35rem;
  background: #2f8fbd;
  cursor: grab;
  touch-action: none;
}

.map-viewport.is-dragging {
  cursor: grabbing;
}

.home-image {
  width: auto;
  max-width: none;
  height: auto;
  display: block;
  transform-origin: top left;
  user-select: none;
  -webkit-user-drag: none;
}
</style>
