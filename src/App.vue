<script setup>
import { ref, reactive, onMounted, onBeforeUnmount } from 'vue'
import axios from 'axios'

// Tamaño del "lienzo" SVG
const width = 700
const height = 480

// Colores disponibles para el coloreo
const palette = ['#FF6B6B', '#4ECDC4', '#FFD93D', '#1A8FE3', '#B084F5']

// Grafo actual
const nodes = ref([])
const edges = ref([])

// Referencia al SVG para exportar como PNG
const svgRef = ref(null)

// Configuración de desafío
const numColors = ref(3)

// Estado de animación / solución
const steps = ref([])              // lista de pasos que enviará el backend
const currentStepIndex = ref(0)
const isPlaying = ref(false)
const playSpeed = ref(600)         // ms entre pasos
let playTimer = null

// Información del paso actual (para mostrar en panel)
const currentInfo = ref('Aún no se ha ejecutado el backtracking.')

// Drag de nodos
const dragging = reactive({
  active: false,
  nodeId: null
})

// Mapa seleccionado (ejemplos)
const selectedSample = ref('mexico_simple')

// ======= Ejemplos de mapas (puedes modificar) =======
const sampleMaps = {
  mexico_simple: {
    label: 'Ejemplo 1 – 6 regiones',
    nodes: [
      { id: 'A', label: 'A', x: 150, y: 100 },
      { id: 'B', label: 'B', x: 300, y: 80 },
      { id: 'C', label: 'C', x: 470, y: 120 },
      { id: 'D', label: 'D', x: 200, y: 260 },
      { id: 'E', label: 'E', x: 360, y: 260 },
      { id: 'F', label: 'F', x: 520, y: 260 }
    ],
    edges: [
      { from: 'A', to: 'B' },
      { from: 'B', to: 'C' },
      { from: 'A', to: 'D' },
      { from: 'B', to: 'D' },
      { from: 'B', to: 'E' },
      { from: 'C', to: 'E' },
      { from: 'D', to: 'E' },
      { from: 'E', to: 'F' },
      { from: 'C', to: 'F' }
    ]
  },
  cuadrado: {
    label: 'Ejemplo 2 – Cuadrado con cruz',
    nodes: [
      { id: '1', label: '1', x: 180, y: 120 },
      { id: '2', label: '2', x: 520, y: 120 },
      { id: '3', label: '3', x: 520, y: 350 },
      { id: '4', label: '4', x: 180, y: 350 },
      { id: '5', label: '5', x: 350, y: 235 }
    ],
    edges: [
      { from: '1', to: '2' },
      { from: '2', to: '3' },
      { from: '3', to: '4' },
      { from: '4', to: '1' },
      { from: '1', to: '5' },
      { from: '2', to: '5' },
      { from: '3', to: '5' },
      { from: '4', to: '5' }
    ]
  }
}

// Cargar el mapa seleccionado
function loadSampleMap () {
  const sample = sampleMaps[selectedSample.value]
  nodes.value = sample.nodes.map(n => ({ ...n, colorIndex: null }))
  edges.value = sample.edges.map(e => ({ ...e }))
  resetAnimation()
}

function resetAnimation () {
  steps.value = []
  currentStepIndex.value = 0
  isPlaying.value = false
  stopTimer()
  currentInfo.value = 'Mapa listo. Configura colores y presiona "Resolver con Backtracking".'
  // limpia colores
  nodes.value = nodes.value.map(n => ({ ...n, colorIndex: null }))
}

// ========= Carga de mapa desde archivo JSON =========
// Formato esperado:
// {
//   "nodes": [{ "id": "1", "label":"1", "x":100, "y":120 }, ...],
//   "edges": [{ "from":"1", "to":"2" }, ...]
// }
function onFileChange (event) {
  const file = event.target.files[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = e => {
    try {
      const data = JSON.parse(e.target.result)
      if (Array.isArray(data.nodes) && Array.isArray(data.edges)) {
        nodes.value = data.nodes.map(n => ({
          ...n,
          colorIndex: null
        }))
        edges.value = data.edges
        resetAnimation()
        currentInfo.value = 'Mapa cargado desde archivo.'
      } else {
        alert('El JSON no tiene el formato esperado.')
      }
    } catch (err) {
      console.error(err)
      alert('Error al leer el archivo JSON.')
    }
  }
  reader.readAsText(file)
}

// ========= Control de drag en SVG =========
function startDrag (node, event) {
  dragging.active = true
  dragging.nodeId = node.id
}

function svgMouseMove (event) {
  if (!dragging.active) return
  const svg = event.currentTarget
  const pt = svg.createSVGPoint()
  pt.x = event.clientX
  pt.y = event.clientY
  const cursorpt = pt.matrixTransform(svg.getScreenCTM().inverse())

  nodes.value = nodes.value.map(n =>
    n.id === dragging.nodeId
      ? { ...n, x: cursorpt.x, y: cursorpt.y }
      : n
  )
}

function stopDrag () {
  dragging.active = false
  dragging.nodeId = null
}

// ========= Llamada al microservicio =========
async function solveWithBacktracking () {
  resetAnimation()

  if (nodes.value.length === 0) {
    alert('Primero carga o selecciona un mapa.')
    return
  }

  try {
    // Armar datos para backend
    const payload = {
      nodes: nodes.value.map(n => n.id),
      edges: edges.value,
      num_colors: numColors.value
    }

    const res = await axios.post('https://backtracking-antonio-api.onrender.com/solve', payload)

    // { steps: [ { assignment: { "A":0,"B":1,... }, note: "texto..." }, ... ] }
    steps.value = res.data.steps || []
    if (steps.value.length === 0) {
      currentInfo.value = 'No se encontraron soluciones con esos colores.'
      return
    }

    currentStepIndex.value = 0
    applyStep(0)
    currentInfo.value = 'Solución recibida. Puedes reproducir la animación.'
  } catch (err) {
    console.error(err)
    alert('Error al llamar al microservicio de backtracking. Revisa la consola.')
  }
}

// Aplica un paso a los nodos
function applyStep (index) {
  if (!steps.value[index]) return
  const assignment = steps.value[index].assignment || {}
  nodes.value = nodes.value.map(n => ({
    ...n,
    colorIndex:
      typeof assignment[n.id] === 'number'
        ? assignment[n.id] % palette.length
        : null
  }))
  currentInfo.value = steps.value[index].note || `Paso ${index + 1} aplicado.`
}

// ========= Controles de animación =========
function play () {
  if (steps.value.length === 0) return
  isPlaying.value = true
  stopTimer()
  playTimer = setInterval(() => {
    currentStepIndex.value++
    if (currentStepIndex.value >= steps.value.length) {
      currentStepIndex.value = steps.value.length - 1
      stopTimer()
      isPlaying.value = false
      return
    }
    applyStep(currentStepIndex.value)
  }, playSpeed.value)
}

function pause () {
  isPlaying.value = false
  stopTimer()
}

function stepForward () {
  if (steps.value.length === 0) return
  if (currentStepIndex.value < steps.value.length - 1) {
    currentStepIndex.value++
    applyStep(currentStepIndex.value)
  }
}

function stepBack () {
  if (steps.value.length === 0) return
  if (currentStepIndex.value > 0) {
    currentStepIndex.value--
    applyStep(currentStepIndex.value)
  }
}

function stopTimer () {
  if (playTimer) {
    clearInterval(playTimer)
    playTimer = null
  }
}

onMounted(() => {
  loadSampleMap()
})

onBeforeUnmount(() => {
  stopTimer()
})

// ========= Descargar mapa como JSON =========
function downloadMap () {
  const data = {
    nodes: nodes.value.map(({ id, label, x, y }) => ({ id, label, x, y })),
    edges: edges.value
  }

  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'mapa_backtracking.json'
  a.click()
  URL.revokeObjectURL(url)
}

// ========= Exportar mapa como PNG =========
function exportMapAsPng () {
  if (!svgRef.value) {
    alert('No se encontró el mapa para exportar.')
    return
  }

  const svg = svgRef.value

  const serializer = new XMLSerializer()
  let source = serializer.serializeToString(svg)

  // Asegurar namespace correcto
  if (!source.includes('xmlns="http://www.w3.org/2000/svg"')) {
    source = source.replace(
      '<svg',
      '<svg xmlns="http://www.w3.org/2000/svg"'
    )
  }

  const svgBlob = new Blob([source], { type: 'image/svg+xml;charset=utf-8' })
  const url = URL.createObjectURL(svgBlob)

  const img = new Image()
  img.onload = () => {
    const canvas = document.createElement('canvas')
    canvas.width = width
    canvas.height = height

    const ctx = canvas.getContext('2d')
    ctx.fillStyle = '#ffffff'
    ctx.fillRect(0, 0, canvas.width, canvas.height)
    ctx.drawImage(img, 0, 0, canvas.width, canvas.height)

    URL.revokeObjectURL(url)

    canvas.toBlob(blob => {
      const pngUrl = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = pngUrl
      a.download = 'mapa_backtracking.png'
      a.click()
      URL.revokeObjectURL(pngUrl)
    }, 'image/png')
  }

  img.src = url
}
</script>

<template>
  <div class="app">
    <header class="topbar">
      <div>
        <h1>Coloreo de Mapas – Backtracking</h1>
        <p class="subtitle">
          Actividad 3.4 · Front en Vue · Motor de backtracking en Python (microservicio).
        </p>
      </div>
      <div class="badge">
        Antonio Garcia Cruz
      </div>
    </header>

    <main class="layout">
      <!-- COLUMNA IZQUIERDA: MAPA -->
      <section class="column map-column">
        <h2>1. Definir / cargar mapa</h2>

        <div class="card">
          <h3>Seleccionar mapa de ejemplo</h3>
          <select v-model="selectedSample" @change="loadSampleMap">
            <option value="mexico_simple">{{ sampleMaps.mexico_simple.label }}</option>
            <option value="cuadrado">{{ sampleMaps.cuadrado.label }}</option>
          </select>

          <div class="file-row">
            <label class="file-label">
              Cargar mapa desde JSON
              <input type="file" accept=".json,application/json" @change="onFileChange">
            </label>
            <button @click="downloadMap">Descargar mapa actual (.json)</button>
          </div>

          <p class="hint">
            Puedes arrastrar los nodos directamente en el lienzo para acomodar mejor el mapa.
          </p>
        </div>

        <div class="card">
          <h3>Vista del grafo</h3>
          <div class="svg-wrapper">
            <svg
              ref="svgRef"
              :viewBox="`0 0 ${width} ${height}`"
              @mousemove="svgMouseMove"
              @mouseup="stopDrag"
              @mouseleave="stopDrag"
            >
              <!-- Aristas -->
              <g class="edges">
                <line
                  v-for="(e, idx) in edges"
                  :key="idx"
                  :x1="nodes.find(n => n.id === e.from)?.x"
                  :y1="nodes.find(n => n.id === e.from)?.y"
                  :x2="nodes.find(n => n.id === e.to)?.x"
                  :y2="nodes.find(n => n.id === e.to)?.y"
                  stroke="#999"
                  stroke-width="2"
                />
              </g>

              <!-- Nodos -->
              <g class="nodes">
                <g
                  v-for="node in nodes"
                  :key="node.id"
                  class="node"
                  @mousedown.stop="startDrag(node, $event)"
                >
                  <circle
                    :cx="node.x"
                    :cy="node.y"
                    r="25"
                    :fill="node.colorIndex !== null ? palette[node.colorIndex] : '#f5f5f5'"
                    stroke="#333"
                    stroke-width="2"
                  />
                  <text
                    :x="node.x"
                    :y="node.y + 5"
                    text-anchor="middle"
                    font-size="16"
                    font-weight="bold"
                  >
                    {{ node.label }}
                  </text>
                </g>
              </g>
            </svg>
          </div>
        </div>
      </section>

      <!-- COLUMNA DERECHA: CONFIGURACIÓN & ANIMACIÓN -->
      <section class="column side-column">
        <!-- Configuración -->
        <div class="card">
          <h2>2. Configurar desafío</h2>
          <div class="config-grid">
            <label>
              Número de colores permitidos:
              <input
                type="number"
                min="2"
                max="5"
                v-model.number="numColors"
              >
            </label>

            <div class="palette-preview">
              <span>Paleta:</span>
              <div class="palette-list">
                <span
                  v-for="(c, idx) in palette"
                  :key="idx"
                  class="color-dot"
                  :style="{ backgroundColor: c }"
                />
              </div>
            </div>
          </div>
        </div>

        <!-- Resolver -->
        <div class="card">
          <h2>3. Resolver con Backtracking</h2>
          <p class="small-text">
            Este botón mandará el grafo y el número de colores al microservicio de Python.
          </p>
          <button class="primary-btn" @click="solveWithBacktracking">
            Resolver con Backtracking
          </button>
        </div>

        <!-- Animación -->
        <div class="card">
          <h2>4. Animación del proceso</h2>
          <div class="controls-row">
            <button @click="play" :disabled="isPlaying || steps.length === 0">▶ Reproducir</button>
            <button @click="pause" :disabled="!isPlaying">⏸ Pausar</button>
            <button @click="stepBack" :disabled="steps.length === 0">⏮ Paso anterior</button>
            <button @click="stepForward" :disabled="steps.length === 0">⏭ Siguiente paso</button>
          </div>
          <div class="slider-row">
            <label>
              Velocidad (ms entre pasos):
              <input type="range" min="200" max="1200" step="100" v-model.number="playSpeed">
              <span>{{ playSpeed }} ms</span>
            </label>
          </div>
          <p class="status">
            Paso actual: {{ steps.length ? currentStepIndex + 1 : 0 }} / {{ steps.length }}
          </p>
          <p class="info">
            {{ currentInfo }}
          </p>
        </div>

        <!-- Compartir -->
        <div class="card">
          <h2>5. Compartir resultados</h2>
          <p class="small-text">
            Descarga el mapa coloreado como imagen para tu reporte o evidencia.
          </p>
          <div class="controls-row" style="margin-top: 8px;">
            <button @click="exportMapAsPng">
              📷 Exportar mapa como PNG
            </button>
          </div>
          <p class="small-text" style="margin-top: 10px;">
            El PNG incluye la última configuración de colores mostrada en el lienzo.
          </p>
        </div>
      </section>
    </main>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  background: #f3f4f6;
  color: #111827;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

.topbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 18px 32px;
  background: #111827;
  color: #f9fafb;
}

.topbar h1 {
  margin: 0;
  font-size: 1.5rem;
}

.subtitle {
  margin: 4px 0 0 0;
  font-size: 0.9rem;
  color: #e5e7eb;
}

.badge {
  font-size: 0.8rem;
  padding: 6px 10px;
  border-radius: 999px;
  border: 1px solid #4b5563;
  background: #1f2937;
}

.layout {
  display: flex;
  gap: 20px;
  padding: 20px 24px 28px;
}

.column {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.map-column {
  flex: 3;
}

.side-column {
  flex: 2;
}

.card {
  background: #ffffff;
  border-radius: 12px;
  padding: 14px 16px 16px;
  box-shadow: 0 8px 20px rgba(15, 23, 42, 0.08);
}

.card h2,
.card h3 {
  margin-top: 0;
}

.svg-wrapper {
  border-radius: 10px;
  border: 1px solid #d1d5db;
  background: #f9fafb;
  overflow: hidden;
}

svg {
  width: 100%;
  height: 420px;
  cursor: default;
}

.node {
  cursor: grab;
}

.node:active {
  cursor: grabbing;
}

.hint {
  font-size: 0.8rem;
  color: #6b7280;
  margin-top: 8px;
}

.file-row {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-top: 10px;
  flex-wrap: wrap;
}

.file-label {
  font-size: 0.85rem;
  padding: 4px 10px;
  border-radius: 999px;
  border: 1px dashed #9ca3af;
  background: #f9fafb;
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.file-label input {
  display: none;
}

select,
input[type="number"] {
  margin-top: 6px;
  padding: 4px 8px;
  border-radius: 8px;
  border: 1px solid #d1d5db;
  font-size: 0.9rem;
}

button {
  padding: 6px 10px;
  border-radius: 999px;
  border: none;
  font-size: 0.85rem;
  cursor: pointer;
  background: #16a34a; /* verde */
  color: #ffffff;
}

button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.primary-btn {
  background: #15803d; /* verde oscuro para botón primario */
  color: #ffffff;
  font-weight: 600;
  padding: 8px 14px;
}

.config-grid {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.config-grid label {
  font-size: 0.9rem;
  display: flex;
  flex-direction: column;
}

.palette-preview {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  font-size: 0.9rem;
}

.palette-list {
  display: flex;
  gap: 6px;
}

.color-dot {
  width: 18px;
  height: 18px;
  border-radius: 999px;
  border: 1px solid #111827;
}

.controls-row {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.slider-row {
  margin-top: 10px;
  font-size: 0.85rem;
}

.slider-row input[type="range"] {
  width: 120px;
  margin: 0 8px;
}

.status {
  margin-top: 8px;
  font-size: 0.9rem;
  font-weight: 500;
}

.info {
  margin-top: 4px;
  font-size: 0.85rem;
  color: #4b5563;
}

.small-text {
  font-size: 0.85rem;
  color: #4b5563;
}

.small-list {
  margin-top: 6px;
  padding-left: 18px;
  font-size: 0.85rem;
  color: #4b5563;
}

@media (max-width: 980px) {
  .layout {
    flex-direction: column;
  }
}
</style>
