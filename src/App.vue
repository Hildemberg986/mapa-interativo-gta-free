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
      @mouseleave="onMouseLeave"
      @mousemove="onMouseMove"
    >
      <div class="map-layer" :style="mapLayerStyle">
        <!-- Imagem do mapa -->
        <img
          class="home-image"
          :src="mapaGta"
          alt="Mapa interativo GTA"
          draggable="false"
          :style="imageStyle"
          @load="onImageLoad"
        />

        <!-- Pins sobre a imagem -->
        <div
          v-for="pin in pins"
          :key="pin.id"
          class="map-pin"
          :style="getPinStyle(pin)"
          :title="`${pin.icon || pin.id}`"
        >
          <img
            v-if="pin.image_url && pin.image_url.trim() !== ''"
            class="map-pin__icon-image"
            :src="pin.image_url"
            alt=""
            aria-hidden="true"
          />
          <span v-else class="map-pin__icon" aria-hidden="true">{{ pin.icon || '📍' }}</span>
          
          <!-- Rabinho com as coordenadas -->
          <div class="map-pin__tail">
            <span class="map-pin__coordinates">
              X: {{ pin.centerX }}, Y: {{ pin.centerY }}
            </span>
          </div>
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
const lastValidCoords = ref({ x: 0, y: 0 })
const pins = ref([])
const isImageLoaded = ref(false)
const minZoom = ref(0.1)

// Estilo da imagem
const imageStyle = computed(() => ({
  width: `${mapNaturalSize.value.width}px`,
  height: `${mapNaturalSize.value.height}px`,
  display: 'block',
  pointerEvents: 'none'
}))

const mapLayerStyle = computed(() => ({
  transform: `translate(${offset.value.x}px, ${offset.value.y}px) scale(${zoom.value})`,
  transformOrigin: '0 0',
  willChange: 'transform',
  position: 'relative',
  width: `${mapNaturalSize.value.width}px`,
  height: `${mapNaturalSize.value.height}px`
}))

// Coordenadas com origem no centro da imagem (0,0 no meio)
const coordLabel = computed(() => {
  if (pointerCoords.value.x === null || pointerCoords.value.y === null) {
    const centerX = lastValidCoords.value.x - (mapNaturalSize.value.width / 2)
    const centerY = lastValidCoords.value.y - (mapNaturalSize.value.height / 2)
    return `X: ${Math.round(centerX)} | Y: ${Math.round(centerY)}`
  }
  const centerX = pointerCoords.value.x - (mapNaturalSize.value.width / 2)
  const centerY = pointerCoords.value.y - (mapNaturalSize.value.height / 2)
  return `X: ${Math.round(centerX)} | Y: ${Math.round(centerY)}`
})

// Função que impede a imagem de sair da tela
function clampOffset(nextOffset) {
  const scaledWidth = mapNaturalSize.value.width * zoom.value
  const scaledHeight = mapNaturalSize.value.height * zoom.value
  const { width: viewportWidth, height: viewportHeight } = viewportSize.value

  if (!scaledWidth || !scaledHeight || !viewportWidth || !viewportHeight) {
    return { x: 0, y: 0 }
  }

  let minX, maxX, minY, maxY
  
  if (scaledWidth <= viewportWidth) {
    minX = (viewportWidth - scaledWidth) / 2
    maxX = minX
  } else {
    minX = viewportWidth - scaledWidth
    maxX = 0
  }
  
  if (scaledHeight <= viewportHeight) {
    minY = (viewportHeight - scaledHeight) / 2
    maxY = minY
  } else {
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
  if (!mapNaturalSize.value.width || !mapNaturalSize.value.height || !viewportSize.value.width || !viewportSize.value.height) {
    return
  }

  const scaledWidth = mapNaturalSize.value.width * zoom.value
  const scaledHeight = mapNaturalSize.value.height * zoom.value
  
  offset.value = {
    x: (viewportSize.value.width - scaledWidth) / 2,
    y: (viewportSize.value.height - scaledHeight) / 2,
  }
}

// Função para ajustar o mapa ao viewport (zoom inicial)
function fitMapToViewport() {
  if (!mapNaturalSize.value.width || !mapNaturalSize.value.height || !viewportSize.value.width || !viewportSize.value.height) {
    return
  }

  const scaleX = viewportSize.value.width / mapNaturalSize.value.width
  const scaleY = viewportSize.value.height / mapNaturalSize.value.height
  const fitZoom = Math.min(scaleX, scaleY)
  
  zoom.value = Math.max(minZoom.value, Math.min(fitZoom, MAX_ZOOM))
  
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
  if (!viewportRef.value || !isImageLoaded.value) {
    return
  }

  const rect = viewportRef.value.getBoundingClientRect()
  const x = (event.clientX - rect.left - offset.value.x) / zoom.value
  const y = (event.clientY - rect.top - offset.value.y) / zoom.value

  if (
    x >= 0 &&
    y >= 0 &&
    x <= mapNaturalSize.value.width &&
    y <= mapNaturalSize.value.height
  ) {
    pointerCoords.value = { x, y }
    lastValidCoords.value = { x, y }
  } else {
    pointerCoords.value = { x: null, y: null }
  }
}

function onMouseMove(event) {
  if (!isDragging.value) {
    updatePointerCoords(event)
  }
}

function onImageLoad(event) {
  mapNaturalSize.value = {
    width: event.target.naturalWidth,
    height: event.target.naturalHeight,
  }
  
  lastValidCoords.value = {
    x: mapNaturalSize.value.width / 2,
    y: mapNaturalSize.value.height / 2
  }
  
  isImageLoaded.value = true
  refreshViewportSize()
}

function getPinStyle(pin) {
  const imageX = pin.centerX + (mapNaturalSize.value.width / 2)
  const imageY = pin.centerY + (mapNaturalSize.value.height / 2)
  
  // Configurações ajustáveis
  const minPinSize = 0.4    // Tamanho mínimo do pin (40%)
  const maxPinSize = 1.2    // Tamanho máximo do pin (120%)
  const baseSize = 0.7      // Tamanho base no zoom 1.0 (70%)
  
  // Calcula a escala baseada no zoom
  const rawScale = baseSize / zoom.value
  const pinScale = Math.min(Math.max(rawScale, minPinSize), maxPinSize)
  
  return {
    left: `${imageX}px`,
    top: `${imageY}px`,
    '--pin-color': pin.color_pin || '#ff4444',
    '--pin-scale': pinScale,
    transform: `translate(-50%, -100%) scale(${pinScale})`,
    position: 'absolute',
    zIndex: 20
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
    icon: normalizedIcon || '📍',
    image_url: normalizedImageUrl,
    centerX: Number(pin?.cord_x),
    centerY: Number(pin?.cord_y),
    color_pin: pin?.color_pin ?? '#ff4444',
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
      .filter((pin) => Number.isFinite(pin.centerX) && Number.isFinite(pin.centerY))
    
    console.log('Pins carregados:', pins.value.length)
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

.map-layer {
  position: relative;
  transform-origin: 0 0;
  will-change: transform;
}

.home-image {
  display: block;
  pointer-events: none;
  position: relative;
  z-index: 1;
}

/* ========== PIN EM FORMATO DE GOTA COM ESCALA INVERSA ========== */
.map-pin {
  position: absolute;
  width: 2.5rem;
  height: 3.25rem;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  z-index: 10;
  will-change: left, top;
  /* A ponta do marcador fica exatamente na coordenada */
  transform: translate(-50%, -100%) scale(var(--pin-scale, 1));
  cursor: pointer;
  transform-origin: bottom center;
}

/* Corpo do marcador - formato de gota */
.map-pin::before {
  content: '';
  position: absolute;
  top: 0;
  left: 50%;
  transform: translateX(-50%) rotate(-45deg);
  width: 2.5rem;
  height: 2.5rem;
  background-color: var(--pin-color, #ff4444);
  border-radius: 50% 50% 50% 0;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.35);
  border: 2px solid white;
  z-index: 1;
}

/* Ícone/Imagem centralizado */
.map-pin__icon,
.map-pin__icon-image {
  position: relative;
  z-index: 3;
  width: 1.2rem;
  height: 1.2rem;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 0.5rem;
  font-weight: bold;
  color: #ffffff;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.5);
}

.map-pin__icon-image {
  object-fit: cover;
  border-radius: 50%;
}

/* Rabinho do pin - acima do marcador */
.map-pin__tail {
  position: absolute;
  top: -2.8rem;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.85);
  backdrop-filter: blur(4px);
  padding: 0.25rem 0.5rem;
  border-radius: 0.25rem;
  white-space: nowrap;
  font-size: 0.7rem;
  font-weight: normal;
  pointer-events: none;
  z-index: 15;
  opacity: 0;
  transition: opacity 0.2s ease;
  border: 1px solid rgba(255, 255, 255, 0.3);
}

/* Seta do rabinho apontando para baixo */
.map-pin__tail::before {
  content: '';
  position: absolute;
  bottom: -0.4rem;
  left: 50%;
  transform: translateX(-50%);
  border-left: 0.4rem solid transparent;
  border-right: 0.4rem solid transparent;
  border-top: 0.4rem solid rgba(0, 0, 0, 0.85);
}

.map-pin:hover .map-pin__tail {
  opacity: 1;
}

.map-pin__coordinates {
  font-family: monospace;
  font-weight: 600;
  color: #ffffff;
}

:global(body) {
  overflow: hidden;
  margin: 0;
  padding: 0;
}
</style>