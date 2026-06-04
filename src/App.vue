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
      @wheel="onWheelZoom"
    >
      <div class="map-layer" :style="mapLayerStyle">
        <div class="map-container" :style="mapContainerStyle">
          <img
            class="home-image"
            :src="mapaGta"
            alt="Mapa interativo GTA"
            draggable="false"
            @load="onImageLoad"
          />
        </div>

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
        :disabled="zoom <= minZoom"
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
import { computed, onMounted, onUnmounted, ref } from 'vue'
import mapaGta from './assets/mapa-gta.webp'

const MAX_ZOOM = 5
const ZOOM_STEP = 0.1
const EXTRA_SPACE = 100000

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
const minZoom = ref(0.05)

// Tamanho total incluindo as margens
const totalSize = computed(() => ({
  width: mapNaturalSize.value.width + (EXTRA_SPACE * 2),
  height: mapNaturalSize.value.height + (EXTRA_SPACE * 2)
}))

const mapContainerStyle = computed(() => ({
  position: 'absolute',
  left: `${EXTRA_SPACE}px`,
  top: `${EXTRA_SPACE}px`,
}))

const mapLayerStyle = computed(() => ({
  transform: `translate(${offset.value.x}px, ${offset.value.y}px) scale(${zoom.value})`,
  transformOrigin: '0 0',
  willChange: 'transform',
  width: `${totalSize.value.width}px`,
  height: `${totalSize.value.height}px`,
  position: 'relative'
}))

const coordLabel = computed(() => {
  if (pointerCoords.value.x === null || pointerCoords.value.y === null) {
    return 'X: -- | Y: --'
  }
  return `X: ${Math.round(pointerCoords.value.x)} | Y: ${Math.round(pointerCoords.value.y)}`
})

// Função que impede a imagem de sair da tela
function clampOffset(nextOffset) {
  const scaledWidth = totalSize.value.width * zoom.value
  const scaledHeight = totalSize.value.height * zoom.value
  const { width: viewportWidth, height: viewportHeight } = viewportSize.value

  if (!scaledWidth || !scaledHeight || !viewportWidth || !viewportHeight) {
    return { x: 0, y: 0 }
  }

  // Limites para manter a área total dentro da tela
  let minX, maxX, minY, maxY
  
  if (scaledWidth <= viewportWidth) {
    // Se a área total é menor que o viewport, centraliza
    minX = (viewportWidth - scaledWidth) / 2
    maxX = minX
  } else {
    // Se a área total é maior, limita para não sair da tela
    minX = viewportWidth - scaledWidth
    maxX = 0
  }
  
  if (scaledHeight <= viewportHeight) {
    // Se a área total é menor que o viewport, centraliza
    minY = (viewportHeight - scaledHeight) / 2
    maxY = minY
  } else {
    // Se a área total é maior, limita para não sair da tela
    minY = viewportHeight - scaledHeight
    maxY = 0
  }

  return {
    x: Math.min(Math.max(nextOffset.x, minX), maxX),
    y: Math.min(Math.max(nextOffset.y, minY), maxY),
  }
}

// Função para centralizar o mapa na tela
function centerMap() {
  if (!totalSize.value.width || !totalSize.value.height || !viewportSize.value.width || !viewportSize.value.height) {
    return
  }

  const scaledWidth = totalSize.value.width * zoom.value
  const scaledHeight = totalSize.value.height * zoom.value
  
  offset.value = {
    x: (viewportSize.value.width - scaledWidth) / 2,
    y: (viewportSize.value.height - scaledHeight) / 2,
  }
}

// Função para ajustar o mapa ao viewport (zoom inicial)
function fitMapToViewport() {
  if (!totalSize.value.width || !totalSize.value.height || !viewportSize.value.width || !viewportSize.value.height) {
    return
  }

  // Calcula o zoom que faz a área total caber completamente no viewport
  const scaleX = viewportSize.value.width / totalSize.value.width
  const scaleY = viewportSize.value.height / totalSize.value.height
  const fitZoom = Math.min(scaleX, scaleY)
  
  // Aplica o zoom, respeitando os limites
  zoom.value = Math.max(minZoom.value, Math.min(fitZoom, MAX_ZOOM))
  
  // Centraliza a área total
  centerMap()
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
  // Ajusta as coordenadas considerando o offset da imagem dentro do container
  const x = (event.clientX - rect.left - offset.value.x) / zoom.value - EXTRA_SPACE
  const y = (event.clientY - rect.top - offset.value.y) / zoom.value - EXTRA_SPACE

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
  // Ajusta a posição dos pins considerando as margens
  const adjustedX = pin.cord_x + EXTRA_SPACE
  const adjustedY = pin.cord_y + EXTRA_SPACE
  const scaledX = adjustedX * zoom.value
  const scaledY = adjustedY * zoom.value
  
  return {
    left: `${scaledX + offset.value.x}px`,
    top: `${scaledY + offset.value.y}px`,
    backgroundColor: pin.color_pin || '#000000',
    transform: 'translate(-50%, -100%) rotate(-45deg)',
    position: 'absolute'
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
  event.preventDefault()
  
  if (viewportRef.value) {
    viewportRef.value.style.cursor = 'grabbing'
  }
}

function onPointerMove(event) {
  if (!isDragging.value || event.pointerId !== activePointerId.value) {
    return
  }

  const deltaX = event.clientX - dragStartPoint.value.x
  const deltaY = event.clientY - dragStartPoint.value.y

  offset.value = clampOffset({
    x: dragStartOffset.value.x + deltaX,
    y: dragStartOffset.value.y + deltaY,
  })
  
  updatePointerCoords(event)
  event.preventDefault()
}

function stopDragging() {
  isDragging.value = false
  activePointerId.value = null
  if (viewportRef.value) {
    viewportRef.value.style.cursor = 'grab'
  }
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
  if (isDragging.value) {
    stopDragging()
  }
}

function setZoom(nextZoom, clientX = null, clientY = null) {
  if (!viewportRef.value) {
    return
  }

  const clampedZoom = Math.min(Math.max(nextZoom, minZoom.value), MAX_ZOOM)
  
  if (clampedZoom === zoom.value) {
    return
  }
  
  if (clientX !== null && clientY !== null) {
    const rect = viewportRef.value.getBoundingClientRect()
    const mouseX = clientX - rect.left
    const mouseY = clientY - rect.top
    
    const worldX = (mouseX - offset.value.x) / zoom.value
    const worldY = (mouseY - offset.value.y) / zoom.value
    
    zoom.value = clampedZoom
    
    offset.value = clampOffset({
      x: mouseX - worldX * zoom.value,
      y: mouseY - worldY * zoom.value,
    })
  } else {
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
  const newZoom = Math.min(zoom.value + ZOOM_STEP, MAX_ZOOM)
  const clientX = event?.clientX ?? null
  const clientY = event?.clientY ?? null
  setZoom(newZoom, clientX, clientY)
}

function zoomOut(event) {
  const newZoom = Math.max(zoom.value - ZOOM_STEP, minZoom.value)
  const clientX = event?.clientX ?? null
  const clientY = event?.clientY ?? null
  setZoom(newZoom, clientX, clientY)
}

function onWheelZoom(event) {
  event.preventDefault()
  
  const delta = event.deltaY > 0 ? -ZOOM_STEP : ZOOM_STEP
  const newZoom = Math.min(Math.max(zoom.value + delta, minZoom.value), MAX_ZOOM)
  
  if (newZoom !== zoom.value) {
    setZoom(newZoom, event.clientX, event.clientY)
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

  switch(event.key) {
    case 'ArrowLeft':
      moveByKeyboard(step, 0)
      event.preventDefault()
      break
    case 'ArrowRight':
      moveByKeyboard(-step, 0)
      event.preventDefault()
      break
    case 'ArrowUp':
      moveByKeyboard(0, step)
      event.preventDefault()
      break
    case 'ArrowDown':
      moveByKeyboard(0, -step)
      event.preventDefault()
      break
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
* {
  user-select: none;
  -webkit-tap-highlight-color: transparent;
}

.home {
  position: relative;
  width: 100vw;
  height: 100vh;
  display: grid;
  gap: 0.75rem;
  margin: 0;
  box-sizing: border-box;
  background: #728aaf;
  overflow: hidden;
}

.hud {
  position: absolute;
  top: 1rem;
  right: 1rem;
  z-index: 100;
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 0.75rem;
  color: #ffffff;
  background-color: transparent;
}

.coords {
  margin: 0;
  font-size: 0.95rem;
  font-weight: 600;
  background: rgba(0, 0, 0, 0.7);
  padding: 0.5rem 1rem;
  border-radius: 0.35rem;
  font-family: monospace;
  pointer-events: none;
  backdrop-filter: blur(4px);
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
  border-radius: 0.35rem;
  background: #728aaf;
  cursor: grab;
  touch-action: none;
}

.map-viewport:active {
  cursor: grabbing;
}

.map-viewport.is-dragging {
  cursor: grabbing;
}

.map-viewport::-webkit-scrollbar {
  display: none;
}

.home-image {
  width: auto;
  max-width: none;
  height: auto;
  display: block;
  user-select: none;
  -webkit-user-drag: none;
  pointer-events: none;
}

.map-layer {
  display: inline-block;
  transform-origin: 0 0;
  will-change: transform;
  position: relative;
}

.map-pin {
  position: absolute;
  width: 2.2rem;
  height: 2.2rem;
  border-radius: 50% 50% 50% 0;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #ffffff;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.35);
  pointer-events: none;
  transition: transform 0.1s ease;
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

:global(body) {
  overflow: hidden;
  margin: 0;
  padding: 0;
}
</style>