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
      @wheel="onWheelZoom"
    >
      <div class="map-layer" :style="mapLayerStyle">
        <img
          class="home-image"
          :src="mapaGta"
          alt="Mapa interativo GTA"
          draggable="false"
          @load="onImageLoad"
        />

        <div
          v-for="pin in pins"
          :key="pin.id"
          class="map-pin"
          :style="getPinStyle(pin)"
          :title="`Pin ${pin.id}`"
        >
          <img
            v-if="pin.image_url && pin.icon === ''"
            class="map-pin__icon-image"
            :src="pin.image_url"
            alt=""
            aria-hidden="true"
          />
          <span v-else class="map-pin__icon" aria-hidden="true">{{ pin.icon || '•' }}</span>
        </div>
      </div>
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

<script setup>
import { computed, onMounted, onUnmounted, ref, nextTick } from 'vue'
import mapaGta from './assets/mapa-gta.webp'

const MIN_ZOOM = 0.0001
const MAX_ZOOM = 3
const ZOOM_STEP = 0.0002

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
const pins = ref([])
const isImageLoaded = ref(false)

const mapLayerStyle = computed(() => ({
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

function fitMapToViewport() {
  if (!mapNaturalSize.value.width || !mapNaturalSize.value.height || !viewportSize.value.width || !viewportSize.value.height) {
    return
  }

  const scaleX = viewportSize.value.width / mapNaturalSize.value.width
  const scaleY = viewportSize.value.height / mapNaturalSize.value.height
  const fitZoom = Math.min(scaleX, scaleY, MAX_ZOOM)
  
  zoom.value = Math.max(MIN_ZOOM, Math.min(fitZoom, MAX_ZOOM))
  
  // Center the map
  const scaledWidth = mapNaturalSize.value.width * zoom.value
  const scaledHeight = mapNaturalSize.value.height * zoom.value
  
  offset.value = {
    x: (viewportSize.value.width - scaledWidth) / 2,
    y: (viewportSize.value.height - scaledHeight) / 2,
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

  if (mapNaturalSize.value.width && mapNaturalSize.value.height && isImageLoaded.value) {
    fitMapToViewport()
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
  isImageLoaded.value = true
  refreshViewportSize()
}

function getPinStyle(pin) {
  return {
    left: `${pin.cord_x}px`,
    top: `${pin.cord_y}px`,
    backgroundColor: pin.color_pin || '#000000',
  }
}

function normalizeText(value) {
  return typeof value === 'string' ? value.trim() : ''
}

function normalizePin(pin, index) {
  const normalizedIcon = normalizeText(pin?.icon)
  const normalizedImageUrl = normalizeText(pin?.image_url)

  return {
    id: pin?.id ?? `pin-${index}`,
    icon: normalizedIcon,
    image_url: normalizedImageUrl,
    cord_x: Number(pin?.cord_x),
    cord_y: Number(pin?.cord_y),
    color_pin: pin?.color_pin ?? '#000000',
  }
}

async function loadPins() {
  try {
    const response = await fetch('/pins/tags.json')
    if (!response.ok) {
      console.error(`Falha ao carregar pins: ${response.status} ${response.statusText}`)
      pins.value = []
      return
    }

    const data = await response.json()
    let rawPins = []
    if (Array.isArray(data)) {
      rawPins = data
    } else if (Array.isArray(data?.pins)) {
      rawPins = data.pins
    }

    pins.value = rawPins
      .map(normalizePin)
      .filter((pin) => Number.isFinite(pin.cord_x) && Number.isFinite(pin.cord_y))
  } catch (error) {
    console.error('Falha ao processar pins em /pins/tags.json', error)
    pins.value = []
  }
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

function setZoom(nextZoom, clientX = null, clientY = null) {
  if (!viewportRef.value) {
    return
  }

  const clampedZoom = Math.min(Math.max(nextZoom, MIN_ZOOM), MAX_ZOOM)
  
  // If specific point is provided (for wheel zoom), zoom around that point
  if (clientX !== null && clientY !== null) {
    const rect = viewportRef.value.getBoundingClientRect()
    const x = clientX - rect.left
    const y = clientY - rect.top
    const worldX = (x - offset.value.x) / zoom.value
    const worldY = (y - offset.value.y) / zoom.value
    
    zoom.value = clampedZoom
    offset.value = clampOffset({
      x: x - worldX * zoom.value,
      y: y - worldY * zoom.value,
    })
  } else {
    // Fallback to center zoom (for button clicks)
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
}

function zoomIn(event) {
  const clientX = event?.clientX ?? null
  const clientY = event?.clientY ?? null
  setZoom(zoom.value + ZOOM_STEP, clientX, clientY)
}

function zoomOut(event) {
  const clientX = event?.clientX ?? null
  const clientY = event?.clientY ?? null
  setZoom(zoom.value - ZOOM_STEP, clientX, clientY)
}

function onWheelZoom(event) {
  event.preventDefault()
  
  // Determine zoom direction based on deltaY
  if (event.deltaY < 0) {
    // Scrolling up -> zoom in
    zoomIn(event)
  } else if (event.deltaY > 0) {
    // Scrolling down -> zoom out
    zoomOut(event)
  }
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
  loadPins()
})

onUnmounted(() => {
  window.removeEventListener('resize', refreshViewportSize)
})
</script>

<style scoped>
.home {
  position: relative;
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

.coords {
  margin: 0;
  font-size: 0.95rem;
  font-weight: 600;
  background: rgba(0, 0, 0, 0.7);
  padding: 0.5rem 1rem;
  border-radius: 0.35rem;
  font-family: monospace;
}

.zoom-controls {
  position: fixed;
  bottom: 1.5rem;
  right: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  z-index: 1000;
}

.zoom-controls button {
  border: none;
  border-radius: 0.5rem;
  width: 2.5rem;
  height: 2.5rem;
  font-size: 1.3rem;
  font-weight: bold;
  background: #000000;
  color: #ffffff;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  transition: all 0.2s ease;
}

.zoom-controls button:hover:not(:disabled) {
  background: #333333;
  transform: scale(1.05);
}

.zoom-controls button:active:not(:disabled) {
  transform: scale(0.95);
}

.zoom-controls button:disabled {
  cursor: not-allowed;
  opacity: 0.5;
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
  user-select: none;
  -webkit-user-drag: none;
}

.map-layer {
  position: relative;
  display: inline-block;
  transform-origin: top left;
}

.map-pin {
  position: absolute;
  width: 2.2rem;
  height: 2.2rem;
  border-radius: 50% 50% 50% 0;
  transform: translate(-50%, -100%) rotate(-45deg);
  display: flex;
  align-items: center;
  justify-content: center;
  color: #ffffff;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.35);
  pointer-events: none;
}

.map-pin__icon,
.map-pin__icon-image {
  transform: rotate(45deg);
}

.map-pin__icon {
  font-size: 1rem;
  line-height: 1;
}

.map-pin__icon-image {
  width: 1rem;
  height: 1rem;
  object-fit: cover;
  border-radius: 999px;
}
</style>
