<template>
  <!-- Mobile warning overlay -->
  <div class="mobile-warning">
    <div class="mobile-warning-box">
      <svg width="48" height="48" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
        <path d="M12 2L2 19h20L12 2z" stroke="#ff9100" stroke-width="1.5" stroke-linejoin="round"/>
        <line x1="12" y1="9" x2="12" y2="13" stroke="#ff9100" stroke-width="1.5" stroke-linecap="round"/>
        <circle cx="12" cy="16" r="0.75" fill="#ff9100"/>
      </svg>
      <div class="mobile-warning-title">DESKTOP REQUIRED</div>
      <div class="mobile-warning-msg">
        Tampilan untuk mobile masih dalam pengembangan !.<br>
        Please open this on a PC, laptop, or tablet for the best experience.
      </div>
      <div class="mobile-warning-sub">Screen width &lt; 768px detected</div>
    </div>
  </div>

  <div class="gis-root">

    <!-- ── TOP TOOLBAR ── -->
    <div class="toolbar">
      <span class="brand">
        <img src="https://res.cloudinary.com/djm8hc5vg/image/upload/v1793903697/a0a3d6c55c13ebbfe324b4d6115d7751.png" alt="">
        <img src="https://res.cloudinary.com/djm8hc5vg/image/upload/v1793903697/cropped-The-Education-University.png" alt="">
      </span>

      <input ref="fileInput" type="file" multiple accept=".shp,.dbf,.shx,.prj" @change="handleImport" style="display:none" />

      <div class="divider" />

      <ToolbarBtn icon="point"   label="Point"   :active="drawMode === 'point'"   color="#76ff03" @click="toggleDraw('point')" />
      <ToolbarBtn icon="line"    label="Line"    :active="drawMode === 'line'"    color="#00e5ff" @click="toggleDraw('line')" />
      <ToolbarBtn icon="polygon" label="Polygon" :active="drawMode === 'polygon'" color="#ff9100" @click="toggleDraw('polygon')" />

      <span v-if="drawMode" class="draw-hint">
        {{ drawMode === 'point' ? 'Click to place point' : `Click vertices • Middle-click to finish ${drawMode}` }}
      </span>

      <div class="divider" />

      <div style="flex:1" />
      <ToolbarBtn icon="edit" label="Remove Feature" :active="editMode" color="#ffd740" @click="toggleEditMode" />
      <ToolbarBtn icon="vertex" label="Vertice" :active="showVertices" color="#a78bfa" @click="showVertices = !showVertices" />

      <template v-if="activeLayer">
        <ToolbarBtn icon="clear" label="Clear" color="#ff4444" @click="clearActiveLayer" />
      </template>

      <div class="basemap-picker">
        <button class="toolbar-btn" @click="basemapOpen = !basemapOpen" title="Change Basemap">
          <span style="display:flex;align-items:center"><svg width="14" height="14" viewBox="0 0 14 14" fill="none" xmlns="http://www.w3.org/2000/svg"><polygon points="1,3 5,1 9,3 13,1 13,11 9,13 5,11 1,13" stroke="#888" stroke-width="1.3" stroke-linejoin="round" fill="none"/><line x1="5" y1="1" x2="5" y2="11" stroke="#888" stroke-width="1.3"/><line x1="9" y1="3" x2="9" y2="13" stroke="#888" stroke-width="1.3"/></svg></span>
          <span style="font-size:10px;color:#555">Change Basemap ▾</span>
        </button>
        <div v-if="basemapOpen" class="basemap-dropdown">
          <div
            v-for="bm in basemaps" :key="bm.id"
            :class="['basemap-item', currentBasemap.id === bm.id ? 'active' : '']"
            @click="setBasemap(bm)"
          >
            <span class="bm-dot" :style="{ background: bm.dot }"></span>
            {{ bm.label }}
          </div>
        </div>
      </div>
    </div>

    <!-- ── BODY ── -->
    <div class="body">

      <!-- ── SIDEBAR ── -->
      <transition name="slide">
        <div v-if="sidebarOpen" class="sidebar">
          <div class="sidebar-header">LAYERS {{ layers.length ? `(${layers.length})` : '' }}</div>
          <div class="sidebar-actions">
            <ToolbarBtn icon="import" label="Import SHP" @click="fileInput.click()" />
            <ToolbarBtn icon="newlayer" label="New Layer" @click="createNewLayer" />
          </div>
          <div class="layer-list">

            <div v-if="!layers.length" class="empty-hint">
              No layers yet.<br>Import a shapefile<br>or create a new layer.
            </div>
            <LayerItem
             v-for="layer in displayedLayers"
              :key="layer.id"
              :layer="layer"
              :isActive="layer.id === activeLayerId"
              :color="layer.color"
              @select="activeLayerId = layer.id"
              @toggle="toggleLayer(layer.id)"
              @remove="removeLayer(layer.id)"
              @zoom="zoomToLayer(layer)"
              @rename="(name) => renameLayer(layer.id, name)"
            />
          </div>

          <!-- RASTER MANAGER -->
          <div class="raster-section">
            <div class="sidebar-header">
              RASTER LAYERS ({{ rasterLayers.length }})
            </div>

            <div class="raster-list">
              <div v-if="!rasterLayers.length" class="empty-hint">
                No raster files found.
              </div>

              <div
                v-for="raster in rasterLayers"
                :key="raster.name"
                class="raster-item"
                :class="{ active: raster.visible, loading: raster.loading }"
              >
                <button class="raster-btn" @click="toggleRaster(raster)">
                  <span class="raster-btn-icon">
                    <span v-if="raster.loading" class="spinner"></span>
                    <template v-else>{{ raster.visible ? '◉' : '○' }}</template>
                  </span>
                  <span class="raster-btn-text">
                    <span class="raster-name">{{ raster.name }}</span>
                    <span class="raster-state">
                      {{ raster.loading ? 'LOADING… (klik untuk batal)'
                       : raster.visible ? 'AKTIF — klik untuk sembunyikan'
                       : 'Klik untuk tampilkan' }}
                    </span>
                  </span>
                </button>

                <div v-if="raster.loading" class="raster-progress">
                  <div class="raster-progress-bar"></div>
                </div>

                <div class="raster-opacity">
                  <span>OPACITY</span>
                  <input type="range" min="0" max="100" :value="raster.opacity"
                         @input="changeRasterOpacity(raster, $event)" />
                  <span>{{ raster.opacity }}%</span>
                </div>

                <div v-if="raster.error" class="raster-error">{{ raster.error }}</div>
              </div>
            </div>
          </div>

          <div v-if="activeLayer" class="sidebar-footer">
            <div class="footer-label">ACTIVE LAYER</div>
            <div class="footer-name" :style="{ color: activeLayer.color }">{{ activeLayer.name }}</div>
            <div class="footer-count">{{ activeLayer.geojson?.features?.length || 0 }} features</div>
            <button class="export-btn" @click="exportLayer(activeLayer)">
              <svg width="12" height="12" viewBox="0 0 14 14" fill="none" style="vertical-align:middle;margin-right:5px"><polyline points="4,6 7,9 10,6" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/><line x1="7" y1="2" x2="7" y2="9" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/><path d="M2 11h10" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>
              EXPORT SHP
            </button>
          </div>
        </div>
      </transition>

      <!-- ── MAP ── -->
      <div class="map-wrap">
        <div ref="mapEl" class="map" />

        <!-- Status bar -->
        <div class="statusbar">
          <span :class="['status-dot', status !== 'Ready' ? 'active' : '']">⬤</span>
          <span>{{ status }}</span>
          <span v-if="drawMode" class="draw-mode-tag">DRAW: {{ drawMode.toUpperCase() }}</span>
        </div>
      </div>

      <!-- ── RIGHT SIDEBAR: LEGEND ── -->
      <aside class="legend-sidebar">
        <div class="sidebar-header">LEGEND</div>

        <div class="legend-body">
          <!-- Legenda SHP yang aktif (selalu paling atas) -->
          <div class="legend-block">
            <div class="legend-title">Vector Layers</div>

            <div v-for="layer in visibleVectorLayers" :key="layer.id" class="legend-item">
              <span class="legend-symbol"
                    :style="{ background: layer.color + '40', borderColor: layer.color }"></span>
              <span class="legend-label">{{ layer.name }}</span>
            </div>

            <div v-if="!visibleVectorLayers.length" class="legend-empty">No visible layers</div>
          </div>

          <!-- Legenda raster (muncul sesuai raster yang aktif) -->
          <div v-if="rasterLegend" class="legend-block">
            <div class="legend-title">{{ rasterLegend.title }}</div>

            <div v-for="item in rasterLegend.items" :key="item.label" class="legend-item">
              <span class="legend-symbol" :style="{ background: item.color }"></span>
              <span class="legend-label">{{ item.label }}</span>
            </div>
          </div>
        </div>
      </aside>



    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount, nextTick, markRaw } from 'vue'
import L from 'leaflet'
import shp from 'shpjs'
import JSZip from 'jszip'
import 'leaflet/dist/leaflet.css'

import parseGeoraster from 'georaster'
import GeoRasterLayer from 'georaster-layer-for-leaflet'

// ── DEFAULT VECTOR LAYER ──────────────────────────────────

const vectorFiles = import.meta.glob('/src/vectors/*', {
  eager: true,
  query: '?url',
  import: 'default'
})

const vectorFileMap = Object.fromEntries(
  Object.entries(vectorFiles).map(([path, url]) => [
    path.split('/').pop().toLowerCase(),
    url
  ])
)

async function loadDefaultVectorLayer() {
  try {
    showStatus('Loading default vector layer...', 0)

    const requiredFiles = ['sptharea.shp', 'sptharea.shx', 'sptharea.dbf']

    // Pastikan file utama dan file pendamping tersedia
    for (const filename of requiredFiles) {
      if (!vectorFileMap[filename]) {
        throw new Error(`Missing file: ${filename}`)
      }
    }

    // Buat ZIP sementara untuk dibaca oleh shpjs
    const zip = new JSZip()

    const filesToLoad = [
      'sptharea.shp',
      'sptharea.shx',
      'sptharea.dbf',
      'sptharea.prj',
      'sptharea.cpg'
    ]

    for (const filename of filesToLoad) {
      const url = vectorFileMap[filename]

      if (!url) continue

      const response = await fetch(url)

      if (!response.ok) {
        throw new Error(`Failed to fetch ${filename}`)
      }

      const buffer = await response.arrayBuffer()
      zip.file(filename, buffer)
    }

    const zipBuffer = await zip.generateAsync({
      type: 'arraybuffer'
    })

    const geojson = await shp(zipBuffer)

    const id = newLayerId()

    const defaultLayer = {
      id,
      name: 'SPTHAREA',
      geojson,
      visible: true,
      color: '#00e5ff',
      isDefault: true
    }

    layers.value.push(defaultLayer)

    activeLayerId.value = id

    // Zoom ke area shapefile
    const bounds = L.geoJSON(geojson).getBounds()

    if (bounds.isValid()) {
      map.fitBounds(bounds, {
        padding: [40, 40],
        maxZoom: 16
      })
    }

    showStatus('Default layer SPTHAREA loaded')

  } catch (error) {
    console.error('Default vector loading error:', error)
    showStatus(`Failed to load SPTHAREA: ${error.message}`)
  }
}
// ── RASTER MANAGER ────────────────────────────────────────

const rasterLayers = ref([])

const rasterModules = import.meta.glob('/src/rasters/*.tif', {
  eager: true,
  query: '?url',
  import: 'default'
})

const availableRasters = Object.entries(rasterModules).map(([path, url]) => ({
  name: path.split('/').pop(),
  url
}))

// Fix Leaflet default icon
delete L.Icon.Default.prototype._getIconUrl
L.Icon.Default.mergeOptions({
  iconRetinaUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon-2x.png',
  iconUrl:       'https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon.png',
  shadowUrl:     'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png',
})

// ── Constants ────────────────────────────────────────────────────────────────
const LAYER_COLORS = [
  '#00e5ff','#ff4081','#76ff03','#ff9100','#d500f9',
  '#1de9b6','#ff1744','#ffea00','#00b0ff','#69f0ae',
]

// ── State ────────────────────────────────────────────────────────────────────
const mapEl       = ref(null)
const fileInput   = ref(null)
const layers      = ref([])
const activeLayerId = ref(null)
const drawMode    = ref(null)
const editMode    = ref(false)
const sidebarOpen    = ref(true)
const basemapOpen    = ref(false)
const showVertices   = ref(false)
const status         = ref('Ready')

const CARTO_API_KEY = 'cb1_50tf_1_be0caefafd2cfc3cd17d2f77'

const basemaps = [
  {
    id: 'dark',
    label: 'Dark',
    dot: '#334',
    url: `https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png?key=${CARTO_API_KEY}`,
    attribution: '© CARTO'
  },
  {
    id: 'light',
    label: 'Light',
    dot: '#ddd',
    url: `https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png?key=${CARTO_API_KEY}`,
    attribution: '© CARTO'
  },
  {
    id: 'street',
    label: 'Streets',
    dot: '#f80',
    url: 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
    attribution: '© OpenStreetMap'
  },
  {
    id: 'topo',
    label: 'Topo',
    dot: '#4a8',
    url: 'https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png',
    attribution: '© OpenTopoMap'
  },
  {
    id: 'sat',
    label: 'Satellite',
    dot: '#246',
    url: 'https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
    attribution: '© Esri'
  },
]
const currentBasemap = ref(basemaps[0])

let map = null
let baseTileLayer = null
const leafletLayers = {} // layerId → L.geoJSON layer
let drawingPoints = []
let tempLayer = null
let tempVertexMarkers = [] // vertex dots while drawing
let editMarkers = []

const activeLayer = computed(() => layers.value.find(l => l.id === activeLayerId.value) || null)

const displayedLayers = computed(() => {
  return [...layers.value].sort((a, b) => {
    // Default layer selalu berada paling atas
    if (a.isDefault) return -1
    if (b.isDefault) return 1

    // Layer lainnya tetap menggunakan urutan terbaru di atas
    return layers.value.indexOf(b) - layers.value.indexOf(a)
  })
})

// ── Legend ────────────────────────────────────────────────
const RASTER_LEGENDS = {
  ndvi: {
    title: 'NDVI Classification',
    items: [
      { color: '#a52a2a', label: '0.0 - 0.2 : Very Low' },
      { color: '#ffff00', label: '0.2 - 0.4 : Low' },
      { color: '#90ee90', label: '0.4 - 0.6 : Moderate' },
      { color: '#008000', label: '0.6 - 0.8 : High' },
      { color: '#006400', label: '0.8 - 1.0 : Very High' },
    ],
  },
  change: {
    title: 'NDVI Change',
    items: [
      { color: '#ff0000', label: 'Vegetation Loss' },
      { color: '#ffffff', label: 'Stable' },
      { color: '#008000', label: 'Vegetation Gain' },
    ],
  },
}

const activeRaster = computed(() => rasterLayers.value.find(r => r.visible) || null)

const rasterLegend = computed(() => {
  const name = activeRaster.value?.name?.toLowerCase()
  if (!name) return null
  if (name.startsWith('ndvi')) return RASTER_LEGENDS.ndvi
  if (name.startsWith('change') || name.startsWith('cumulative')) return RASTER_LEGENDS.change
  return null
})

const visibleVectorLayers = computed(() => displayedLayers.value.filter(l => l.visible))

// ── Status helper ─────────────────────────────────────────────────────────────
let statusTimer = null
function showStatus(msg, duration = 3000) {
  status.value = msg
  if (statusTimer) clearTimeout(statusTimer)
  if (duration) statusTimer = setTimeout(() => (status.value = 'Ready'), duration)
}

// ── Map init ──────────────────────────────────────────────────────────────────
onMounted(async () => {
  await nextTick()
  map = L.map(mapEl.value, {
    zoomControl: false,
    zoomSnap: 0.25,
    zoomDelta: 0.5,
    wheelPxPerZoomLevel: 120,
  }).setView([0, 0], 3)

  baseTileLayer = L.tileLayer(currentBasemap.value.url, {
    attribution: currentBasemap.value.attribution, maxZoom: 19,
  }).addTo(map)

  L.control.zoom({ position: 'bottomright' }).addTo(map)

  // close basemap dropdown on map click
  map.on('click', () => { basemapOpen.value = false })

  initializeRasterLayers()
await loadDefaultVectorLayer()
})

onBeforeUnmount(() => {
  // batalkan semua load raster yang masih berjalan
  rasterRequestId++

  if (map) {
    map.getContainer().removeEventListener('mousedown', onMiddleClick)

    rasterLayers.value.forEach(raster => {
      if (raster.layer && map.hasLayer(raster.layer)) {
        map.removeLayer(raster.layer)
      }
    })

    map.remove()
    map = null
  }

  if (statusTimer) {
    clearTimeout(statusTimer)
  }
})

// ── Draw mode ─────────────────────────────────────────────────────────────────
let verticesBeforeDraw = false
watch(drawMode, (mode, prevMode) => {
  if (!map) return
  map.off('click', onMapClick)
  map.getContainer().removeEventListener('mousedown', onMiddleClick)
  clearTemp()
  drawingPoints = []
  map.getContainer().style.cursor = mode ? 'crosshair' : ''

  // Auto-show vertices when entering line/polygon draw; restore when leaving
  if (mode && mode !== 'point') {
    verticesBeforeDraw = showVertices.value
    showVertices.value = true
  } else if (!mode && prevMode && prevMode !== 'point') {
    showVertices.value = verticesBeforeDraw
  }

  if (mode) {
    map.on('click', onMapClick)
    if (mode !== 'point') {
      map.getContainer().addEventListener('mousedown', onMiddleClick)
    }
  }
})

watch(editMode, (active) => {
  if (!map) return
  if (active) {
    drawMode.value = null
    renderEditMarkers()
    showStatus('Edit mode: click a feature to delete it', 5000)
  } else {
    clearEditMarkers()
    showStatus('Edit mode off')
  }
})

watch(activeLayerId, () => {
  if (editMode.value) renderEditMarkers()
})

function setBasemap(bm) {
  if (!map) return
  currentBasemap.value = bm
  basemapOpen.value = false
  if (baseTileLayer) map.removeLayer(baseTileLayer)
  baseTileLayer = L.tileLayer(bm.url, { attribution: bm.attribution, maxZoom: 19 }).addTo(map)
  baseTileLayer.bringToBack()
}

// ── RASTER FUNCTIONS ──────────────────────────────────────

function initializeRasterLayers() {
  rasterLayers.value = availableRasters.map(raster => ({
  name: raster.name,
  url: raster.url,
  georaster: null,
  layer: null,
  visible: false,
  loading: false,
  error: null,
  opacity: 80
}))
}

let rasterRequestId = 0 // penanda request terbaru (untuk membatalkan load yang basi)

// Konversi piksel georaster -> L.imageOverlay (tanpa tile cache)
function georasterToOverlay(georaster, opacity) {
  const MAX = 2048
  const scale = Math.min(1, MAX / Math.max(georaster.width, georaster.height))
  const w = Math.max(1, Math.round(georaster.width * scale))
  const h = Math.max(1, Math.round(georaster.height * scale))

  const canvas = document.createElement('canvas')
  canvas.width = w
  canvas.height = h
  const ctx = canvas.getContext('2d')
  const img = ctx.createImageData(w, h)

  const bands = georaster.values // [band][row][col]
  const nb = georaster.numberOfRasters
  const nodata = georaster.noDataValue

  for (let y = 0; y < h; y++) {
    const sy = Math.min(georaster.height - 1, Math.floor(y / scale))
    for (let x = 0; x < w; x++) {
      const sx = Math.min(georaster.width - 1, Math.floor(x / scale))
      const i = (y * w + x) * 4
      const r = bands[0][sy][sx]
      if (nb >= 3) {
        img.data[i]     = r
        img.data[i + 1] = bands[1][sy][sx]
        img.data[i + 2] = bands[2][sy][sx]
        img.data[i + 3] = nb >= 4 ? bands[3][sy][sx] : 255
      } else {
        img.data[i] = img.data[i + 1] = img.data[i + 2] = r
        img.data[i + 3] = 255
      }
      if (nodata != null && r === nodata) img.data[i + 3] = 0
    }
  }
  ctx.putImageData(img, 0, 0)

  let bounds
  if (georaster.projection === 3857) {
    const R = 6378137
    const lat = y => Math.atan(Math.sinh(y / R)) * 180 / Math.PI
    const lng = x => x / R * 180 / Math.PI
    bounds = [[lat(georaster.ymin), lng(georaster.xmin)], [lat(georaster.ymax), lng(georaster.xmax)]]
  } else {
    bounds = [[georaster.ymin, georaster.xmin], [georaster.ymax, georaster.xmax]]
  }

  return L.imageOverlay(canvas.toDataURL('image/png'), bounds, { opacity })
}

function hideRaster(raster) {
  if (raster.layer && map) map.removeLayer(raster.layer)
  raster.layer = null
  raster.visible = false
}

async function toggleRaster(raster) {
  if (!map) return

  // klik raster aktif -> sembunyikan
  if (raster.visible) {
    hideRaster(raster)
    showStatus(`Hidden ${raster.name}`)
    return
  }

  // klik raster yang sedang loading -> batalkan
  if (raster.loading) {
    rasterRequestId++
    raster.loading = false
    showStatus(`Cancelled ${raster.name}`)
    return
  }

  // sembunyikan semua raster lain
  const requestId = ++rasterRequestId
  rasterLayers.value.forEach(r => {
    if (r !== raster) { hideRaster(r); r.loading = false }
  })

  try {
    raster.loading = true
    raster.error = null
    showStatus(`Loading ${raster.name}...`, 0)

    // cache hanya untuk data hasil parse
    if (!raster.georaster) {
      const response = await fetch(raster.url)
      if (!response.ok) throw new Error(`HTTP Error: ${response.status}`)
      const buf = await response.arrayBuffer()
      if (requestId !== rasterRequestId) return

      const g = await parseGeoraster(buf)
      if (requestId !== rasterRequestId) return
      if (!g.width || !g.height) throw new Error('Invalid raster dimensions')
      raster.georaster = markRaw(g)
    }

    const g = raster.georaster
    console.log('RASTER', raster.name, {
      url: raster.url,
      bands: g.numberOfRasters,
      projection: g.projection,
      bbox: [g.xmin, g.ymin, g.xmax, g.ymax],
      sample: g.values[0][Math.floor(g.height / 2)].slice(0, 8),
    })

    const layer = markRaw(georasterToOverlay(g, raster.opacity / 100))
    layer.addTo(map)
    raster.layer = layer
    raster.visible = true

    const b = layer.getBounds()
    if (b?.isValid()) map.fitBounds(b, { padding: [40, 40], maxZoom: 16 })
    showStatus(`Loaded ${raster.name}`)
  } catch (error) {
    if (requestId !== rasterRequestId) return
    console.error('Raster loading error:', error)
    raster.error = error.message
    raster.visible = false
    showStatus(`Failed: ${raster.name}`)
  } finally {
    if (requestId === rasterRequestId) raster.loading = false
  }
}

// Change opacity
function changeRasterOpacity(raster, event) {
  const value = Number(event.target.value)
  raster.opacity = value
  if (raster.layer) {
    raster.layer.setOpacity(value / 100)
  }
}

// ── Draw helpers ──────────────────────────────────────────────────────────────
function toggleDraw(mode) {
  editMode.value = false
  drawMode.value = drawMode.value === mode ? null : mode
}

function toggleEditMode() {
  editMode.value = !editMode.value
  if (editMode.value) drawMode.value = null
}

function onMapClick(e) {
  if (!drawMode.value) return
  if (drawMode.value === 'point') {
    addFeature({ type: 'Point', coordinates: [e.latlng.lng, e.latlng.lat] })
    return
  }
  drawingPoints.push([e.latlng.lat, e.latlng.lng])

  // add a tiny vertex dot
  const dot = L.circleMarker(e.latlng, {
    radius: 4, color: '#ffffff', fillColor: '#00e5ff',
    fillOpacity: 1, weight: 1.5, interactive: false,
  }).addTo(map)
  tempVertexMarkers.push(dot)

  clearTempLine()
  if (drawingPoints.length > 1) {
    const opts = { color: '#00e5ff', weight: 2, dashArray: '6,4', fillOpacity: 0.1 }
    tempLayer = (drawMode.value === 'line'
      ? L.polyline(drawingPoints, opts)
      : L.polygon(drawingPoints, opts)
    ).addTo(map)
  }
}

function clearTempLine() {
  if (tempLayer) { map.removeLayer(tempLayer); tempLayer = null }
}

function clearTemp() {
  clearTempLine()
  tempVertexMarkers.forEach(m => map.removeLayer(m))
  tempVertexMarkers = []
}

function onMiddleClick(e) {
  if (e.button !== 1) return
  e.preventDefault()
  const mode = drawMode.value
  if (mode === 'line' && drawingPoints.length >= 2) {
    const coords = drawingPoints.map(([lat, lng]) => [lng, lat])
    addFeature({ type: 'LineString', coordinates: coords })
  } else if (mode === 'polygon' && drawingPoints.length >= 3) {
    const coords = [...drawingPoints.map(([lat, lng]) => [lng, lat])]
    coords.push(coords[0])
    addFeature({ type: 'Polygon', coordinates: [coords] })
  } else {
    showStatus('Not enough points to finish')
    return
  }
  drawingPoints = []
  clearTemp()
}

// ── Edit mode ──────────────────────────────────────────────────────────────────
function renderEditMarkers() {
  clearEditMarkers()
  const layer = activeLayer.value
  if (!layer?.geojson?.features?.length) return

  layer.geojson.features.forEach((feature, idx) => {
    const geo = feature.geometry
    // Get a representative point for each feature
    let latlng
    if (geo.type === 'Point') {
      latlng = L.latLng(geo.coordinates[1], geo.coordinates[0])
    } else if (geo.type === 'LineString') {
      const mid = Math.floor(geo.coordinates.length / 2)
      latlng = L.latLng(geo.coordinates[mid][1], geo.coordinates[mid][0])
    } else if (geo.type === 'Polygon') {
      const ring = geo.coordinates[0]
      const mid = Math.floor(ring.length / 2)
      latlng = L.latLng(ring[mid][1], ring[mid][0])
    } else return

    const marker = L.circleMarker(latlng, {
      radius: 8, color: '#ff4444', fillColor: '#ff4444',
      fillOpacity: 0.7, weight: 2, className: 'edit-marker'
    }).addTo(map)

    marker.bindTooltip('Click to delete', { permanent: false, direction: 'top' })
    marker.on('click', (e) => {
      L.DomEvent.stopPropagation(e)
      deleteFeature(layer.id, idx)
    })
    editMarkers.push(marker)
  })
}

function clearEditMarkers() {
  editMarkers.forEach(m => map.removeLayer(m))
  editMarkers = []
}

function deleteFeature(layerId, featureIdx) {
  const layer = layers.value.find(l => l.id === layerId)
  if (!layer) return
  if (!confirm('Delete this feature?')) return
  layer.geojson.features.splice(featureIdx, 1)
  syncLeafletLayer(layerId)
  renderEditMarkers()
  showStatus('Feature deleted')
}

function addFeature(geometry) {
  const feature = { type: 'Feature', geometry, properties: { id: Date.now() } }
  if (!activeLayerId.value) {
    const id = newLayerId()
    const name = `${geometry.type} Layer`
    const color = LAYER_COLORS[layers.value.length % LAYER_COLORS.length]
    layers.value.push({ id, name, geojson: { type: 'FeatureCollection', features: [feature] }, visible: true, color })
    activeLayerId.value = id
    showStatus(`Created layer: ${name}`)
  } else {
    const l = layers.value.find(l => l.id === activeLayerId.value)
    if (l) l.geojson.features.push(feature)
    showStatus(`Added ${geometry.type}`)
  }
  syncLeafletLayer(activeLayerId.value)
}

// ── Layer rendering ───────────────────────────────────────────────────────────
watch(layers, () => {
  layers.value.forEach(l => syncLeafletLayer(l.id))
  // Remove layers no longer in state
  Object.keys(leafletLayers).forEach(id => {
    if (!layers.value.find(l => l.id === id)) {
      map.removeLayer(leafletLayers[id])
      delete leafletLayers[id]
    }
  })
}, { deep: true })

watch(showVertices, () => {
  layers.value.forEach(l => syncLeafletLayer(l.id))
})

function syncLeafletLayer(id) {
  const layer = layers.value.find(l => l.id === id)
  if (!layer || !map) return

  if (leafletLayers[id]) { map.removeLayer(leafletLayers[id]) }

  if (!layer.visible || !layer.geojson?.features?.length) {
    delete leafletLayers[id]
    return
  }

  const gLayer = L.geoJSON(layer.geojson, {
    style: () => ({ color: layer.color, fillColor: layer.color, weight: 2, fillOpacity: 0.25, opacity: 0.9 }),
    pointToLayer: (f, latlng) => L.circleMarker(latlng, {
      radius: 6, color: layer.color, fillColor: layer.color, fillOpacity: 0.8, weight: 2,
    }),
    onEachFeature: (f, fl) => {
      const props = f.properties || {}
      const html = Object.entries(props).map(([k, v]) => `<b>${k}</b>: ${v}`).join('<br>')
      if (html) fl.bindPopup(`<div class="popup-content">${html}</div>`)
    },
  }).addTo(map)

  // Draw tiny vertex dots on lines and polygons
  if (showVertices.value) {
    layer.geojson.features.forEach(f => {
      const geo = f.geometry
      let rings = []
      if (geo.type === 'LineString') rings = [geo.coordinates]
      else if (geo.type === 'MultiLineString') rings = geo.coordinates
      else if (geo.type === 'Polygon') rings = geo.coordinates
      else if (geo.type === 'MultiPolygon') rings = geo.coordinates.flat()

      rings.forEach(ring => {
        // for polygons skip the closing duplicate point
        const pts = (geo.type === 'Polygon' || geo.type === 'MultiPolygon')
          ? ring.slice(0, -1) : ring
        pts.forEach(([lng, lat]) => {
          L.circleMarker([lat, lng], {
            radius: 3, color: layer.color, fillColor: '#0a0e14',
            fillOpacity: 1, weight: 1.5, interactive: false,
          }).addTo(gLayer)
        })
      })
    })
  }

  leafletLayers[id] = gLayer
}

// ── Import SHP ────────────────────────────────────────────────────────────────
async function handleImport(e) {
  const files = Array.from(e.target.files)
  e.target.value = ''

  if (!files.length) return

  showStatus('Importing...')

  for (const file of files) {
    try {
      const buffer = await file.arrayBuffer()
      const geojson = await shp(buffer)

      const id = newLayerId()
      const color = LAYER_COLORS[layers.value.length % LAYER_COLORS.length]

      layers.value.push({
        id,
        name: file.name.replace('.zip', ''),
        geojson,
        visible: true,
        color
      })

      activeLayerId.value = id

      showStatus(`Imported "${file.name}" — ${geojson.features?.length || 0} features`)
    } catch (err) {
      showStatus(`Error importing "${file.name}": ${err.message}`)
    }
  }
}

// ── Export SHP ────────────────────────────────────────────────────────────────
async function exportLayer(layer) {
  if (!layer?.geojson?.features?.length) { showStatus('No features to export'); return }
  showStatus('Exporting...', 0)
  try {
    const result = geojsonToShapefile(layer.geojson)
    if (!result) { showStatus('Unsupported geometry type'); return }

    const zip = new JSZip()
    const safe = layer.name.replace(/[^a-zA-Z0-9_]/g, '_')
    zip.file(`${safe}.shp`, result.shp)
    zip.file(`${safe}.dbf`, result.dbf)
    zip.file(`${safe}.shx`, result.shx)
    zip.file(`${safe}.prj`, 'GEOGCS["GCS_WGS_1984",DATUM["D_WGS_1984",SPHEROID["WGS_1984",6378137.0,298.257223563]],PRIMEM["Greenwich",0.0],UNIT["Degree",0.0174532925199433]]')

    const blob = await zip.generateAsync({ type: 'blob' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url; a.download = `${safe}.zip`; a.click()
    URL.revokeObjectURL(url)
    showStatus(`Exported "${layer.name}"`)
  } catch (err) {
    showStatus(`Export error: ${err.message}`)
  }
}

// ── Layer management ──────────────────────────────────────────────────────────
function createNewLayer() {
  const id = newLayerId()
  const n = layers.value.length + 1
  const color = LAYER_COLORS[layers.value.length % LAYER_COLORS.length]
  layers.value.push({ id, name: `New Layer ${n}`, geojson: { type: 'FeatureCollection', features: [] }, visible: true, color })
  activeLayerId.value = id
  showStatus('New layer created')
}

function renameLayer(id, name) {
  const l = layers.value.find(l => l.id === id)
  if (l && name.trim()) { l.name = name.trim(); showStatus(`Renamed to "${l.name}"`) }
}

function removeLayer(id) {
  const layer = layers.value.find(l => l.id === id)
  if (!confirm(`Delete layer "${layer?.name}"?`)) return
  if (leafletLayers[id]) { map.removeLayer(leafletLayers[id]); delete leafletLayers[id] }
  layers.value = layers.value.filter(l => l.id !== id)
  if (activeLayerId.value === id) activeLayerId.value = null
}

function toggleLayer(id) {
  const l = layers.value.find(l => l.id === id)
  if (l) { l.visible = !l.visible; syncLeafletLayer(id) }
}

function zoomToLayer(layer) {
  if (!layer.geojson?.features?.length) return
  try {
    const bounds = L.geoJSON(layer.geojson).getBounds()
    if (bounds.isValid()) map.fitBounds(bounds, { padding: [40, 40] })
  } catch (e) {}
}

function clearActiveLayer() {
  if (!activeLayer.value) return
  if (!confirm(`Clear all features from "${activeLayer.value.name}"?`)) return
  activeLayer.value.geojson.features = []
  syncLeafletLayer(activeLayer.value.id)
  showStatus('Layer cleared')
}

// ── Helpers ───────────────────────────────────────────────────────────────────
function newLayerId() { return `layer_${Date.now()}_${Math.random().toString(36).slice(2)}` }

// ── SHP encoder ───────────────────────────────────────────────────────────────
function geojsonToShapefile(geojson) {
  const features = geojson.features || []
  if (!features.length) return null
  const firstType = features[0].geometry.type
  let shapeType
  if (firstType === 'Point' || firstType === 'MultiPoint') shapeType = 1
  else if (firstType === 'LineString' || firstType === 'MultiLineString') shapeType = 3
  else if (firstType === 'Polygon' || firstType === 'MultiPolygon') shapeType = 5
  else return null

  const records = features.map(f => {
    const g = f.geometry
    if (shapeType === 1) return g.type === 'MultiPoint' ? g.coordinates : [g.coordinates]
    if (shapeType === 3) return g.type === 'MultiLineString' ? g.coordinates : [g.coordinates]
    return g.type === 'MultiPolygon' ? g.coordinates.flat() : g.coordinates
  })

  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity
  records.forEach(parts => parts.forEach(ring => ring.forEach(([x, y]) => {
    if (x < minX) minX = x; if (y < minY) minY = y
    if (x > maxX) maxX = x; if (y > maxY) maxY = y
  })))

  let shpSize = 100
  const recData = records.map(parts => {
    if (shapeType === 1) {
      const buf = new ArrayBuffer(4 + 16)
      const dv = new DataView(buf)
      dv.setInt32(0, 1, true)
      dv.setFloat64(4, parts[0][0], true)
      dv.setFloat64(12, parts[0][1], true)
      shpSize += 8 + buf.byteLength
      return buf
    }
    const numPoints = parts.reduce((a, r) => a + r.length, 0)
    const buf = new ArrayBuffer(4 + 32 + 4 + 4 + parts.length * 4 + numPoints * 16)
    const dv = new DataView(buf)
    let off = 0
    dv.setInt32(off, shapeType, true); off += 4
    let bX0=Infinity,bY0=Infinity,bX1=-Infinity,bY1=-Infinity
    parts.forEach(r => r.forEach(([x,y]) => { if(x<bX0)bX0=x; if(y<bY0)bY0=y; if(x>bX1)bX1=x; if(y>bY1)bY1=y }))
    dv.setFloat64(off,bX0,true);off+=8; dv.setFloat64(off,bY0,true);off+=8
    dv.setFloat64(off,bX1,true);off+=8; dv.setFloat64(off,bY1,true);off+=8
    dv.setInt32(off,parts.length,true);off+=4
    dv.setInt32(off,numPoints,true);off+=4
    let pt=0; parts.forEach(r=>{dv.setInt32(off,pt,true);off+=4;pt+=r.length})
    parts.forEach(r=>r.forEach(([x,y])=>{dv.setFloat64(off,x,true);off+=8;dv.setFloat64(off,y,true);off+=8}))
    shpSize += 8 + buf.byteLength
    return buf
  })

  // Build SHP
  const shpBuf = new ArrayBuffer(shpSize)
  const shpDv = new DataView(shpBuf)
  shpDv.setInt32(0,9994); shpDv.setInt32(24,shpSize/2)
  shpDv.setInt32(28,1000,true); shpDv.setInt32(32,shapeType,true)
  shpDv.setFloat64(36,minX,true); shpDv.setFloat64(44,minY,true)
  shpDv.setFloat64(52,maxX,true); shpDv.setFloat64(60,maxY,true)
  let off = 100
  recData.forEach((rd,i)=>{
    const dv = new DataView(shpBuf,off)
    dv.setInt32(0,i+1); dv.setInt32(4,rd.byteLength/2)
    new Uint8Array(shpBuf,off+8).set(new Uint8Array(rd))
    off += 8+rd.byteLength
  })

  // Build DBF
  const hLen=65, recLen=11
  const dbfSize=hLen+records.length*recLen+1
  const dbfBuf=new ArrayBuffer(dbfSize)
  const dbfDv=new DataView(dbfBuf)
  dbfDv.setUint8(0,3)
  const now=new Date(); dbfDv.setUint8(1,now.getFullYear()-1900); dbfDv.setUint8(2,now.getMonth()+1); dbfDv.setUint8(3,now.getDate())
  dbfDv.setInt32(4,records.length,true); dbfDv.setInt16(8,hLen,true); dbfDv.setInt16(10,recLen,true)
  'ID'.split('').forEach((c,i)=>dbfDv.setUint8(32+i,c.charCodeAt(0)))
  dbfDv.setUint8(43,78); dbfDv.setUint8(48,10); dbfDv.setUint8(32+32,0x0d)
  for(let i=0;i<records.length;i++){
    const base=hLen+i*recLen
    dbfDv.setUint8(base,0x20)
    String(i+1).padStart(10,' ').split('').forEach((c,j)=>dbfDv.setUint8(base+1+j,c.charCodeAt(0)))
  }
  new DataView(dbfBuf).setUint8(dbfSize-1,0x1a)

  // Build SHX
  const shxSize=100+records.length*8
  const shxBuf=new ArrayBuffer(shxSize)
  const shxDv=new DataView(shxBuf)
  shxDv.setInt32(0,9994); shxDv.setInt32(24,shxSize/2)
  shxDv.setInt32(28,1000,true); shxDv.setInt32(32,shapeType,true)
  shxDv.setFloat64(36,minX,true); shxDv.setFloat64(44,minY,true)
  shxDv.setFloat64(52,maxX,true); shxDv.setFloat64(60,maxY,true)
  let shxOff=50
  recData.forEach(rd=>{
    const idx=(shxOff-50)*2
    shxDv.setInt32(100+idx,shxOff)
    shxDv.setInt32(104+idx,rd.byteLength/2)
    shxOff+=4+rd.byteLength/2
  })

  return { shp: shpBuf, dbf: dbfBuf, shx: shxBuf }
}
</script>

<!-- ── Child Components (defined inline for single-file portability) ── -->
<script>
// LayerItem and ToolbarBtn are defined as local components
import { defineComponent, h, ref } from 'vue'

export const LayerItem = defineComponent({
  props: ['layer','isActive','color'],
  emits: ['select','toggle','remove','zoom','rename'],
  setup(props, { emit }) {
    const typeIcon = {
      Point:`<svg width="13" height="13" viewBox="0 0 14 14" fill="none"><circle cx="7" cy="7" r="4.5" stroke="currentColor" stroke-width="1.5"/><circle cx="7" cy="7" r="1.5" fill="currentColor"/></svg>`,
      MultiPoint:`<svg width="13" height="13" viewBox="0 0 14 14" fill="none"><circle cx="7" cy="7" r="4.5" stroke="currentColor" stroke-width="1.5"/><circle cx="7" cy="7" r="1.5" fill="currentColor"/></svg>`,
      LineString:`<svg width="13" height="13" viewBox="0 0 14 14" fill="none"><line x1="2" y1="12" x2="12" y2="2" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/><circle cx="2" cy="12" r="1.5" fill="currentColor"/><circle cx="12" cy="2" r="1.5" fill="currentColor"/></svg>`,
      MultiLineString:`<svg width="13" height="13" viewBox="0 0 14 14" fill="none"><line x1="2" y1="12" x2="12" y2="2" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/><circle cx="2" cy="12" r="1.5" fill="currentColor"/><circle cx="12" cy="2" r="1.5" fill="currentColor"/></svg>`,
      Polygon:`<svg width="13" height="13" viewBox="0 0 14 14" fill="none"><polygon points="7,1.5 12.5,5 12.5,9 7,12.5 1.5,9 1.5,5" stroke="currentColor" stroke-width="1.5" fill="currentColor" fill-opacity="0.2"/></svg>`,
      MultiPolygon:`<svg width="13" height="13" viewBox="0 0 14 14" fill="none"><polygon points="7,1.5 12.5,5 12.5,9 7,12.5 1.5,9 1.5,5" stroke="currentColor" stroke-width="1.5" fill="currentColor" fill-opacity="0.2"/></svg>`,
    }
    const fallbackIcon = `<svg width="13" height="13" viewBox="0 0 14 14" fill="none"><rect x="2" y="2" width="10" height="10" rx="2" stroke="currentColor" stroke-width="1.5"/></svg>`
    const icon = () => typeIcon[props.layer.geojson?.features?.[0]?.geometry?.type] || fallbackIcon
    const editing = ref(false)
    const editVal = ref('')

    function startRename(e) {
      e.stopPropagation()
      editing.value = true
      editVal.value = props.layer.name
      // focus the input next tick
      setTimeout(() => {
        const inp = document.querySelector(`[data-layer-id="${props.layer.id}"] .rename-input`)
        if (inp) { inp.focus(); inp.select() }
      }, 30)
    }

    function commitRename() {
      editing.value = false
      emit('rename', editVal.value)
    }

    function cancelRename() {
      editing.value = false
    }

    return () => h('div', {
      class: ['layer-item', props.isActive ? 'active' : ''],
      style: { borderLeftColor: props.isActive ? props.color : 'transparent' },
      'data-layer-id': props.layer.id,
      onClick: () => emit('select'),
    }, [
      h('span', { style: { color: props.color, fontSize:'14px', minWidth:'16px', display:'flex', alignItems:'center' }, innerHTML: icon() }),
      editing.value
        ? h('input', {
            class: 'rename-input',
            value: editVal.value,
            onInput: (e) => { editVal.value = e.target.value },
            onKeydown: (e) => {
              if (e.key === 'Enter') commitRename()
              if (e.key === 'Escape') cancelRename()
              e.stopPropagation()
            },
            onBlur: commitRename,
            onClick: (e) => e.stopPropagation(),
            style: { flex:1, background:'#1c2333', border:'1px solid #00e5ff', color:'#e0e0e0',
              fontFamily:'IBM Plex Mono,monospace', fontSize:'12px', padding:'1px 4px', borderRadius:'3px' }
          })
        : h('span', {
            class:'layer-name',
            style:{ color: props.layer.visible ? '#e0e0e0' : '#555' },
            title: 'Double-click to rename',
            onDblclick: startRename,
          }, props.layer.name),
      h('span', { class:'layer-count' }, props.layer.geojson?.features?.length || 0),
      h('button', { class:'icon-btn', title:'Zoom', onClick:(e)=>{e.stopPropagation();emit('zoom')}, innerHTML:`<svg width="13" height="13" viewBox="0 0 14 14" fill="none"><circle cx="6" cy="6" r="4" stroke="currentColor" stroke-width="1.3"/><line x1="9.2" y1="9.2" x2="12" y2="12" stroke="currentColor" stroke-width="1.3" stroke-linecap="round"/><line x1="6" y1="4" x2="6" y2="8" stroke="currentColor" stroke-width="1.2" stroke-linecap="round"/><line x1="4" y1="6" x2="8" y2="6" stroke="currentColor" stroke-width="1.2" stroke-linecap="round"/></svg>` }),
      h('button', { class:'icon-btn', style:{color: props.layer.visible ? props.color : '#444'}, title:'Toggle', onClick:(e)=>{e.stopPropagation();emit('toggle')}, innerHTML: props.layer.visible ? `<svg width="13" height="13" viewBox="0 0 14 14" fill="none"><ellipse cx="7" cy="7" rx="5.5" ry="3.5" stroke="currentColor" stroke-width="1.3"/><circle cx="7" cy="7" r="2" fill="currentColor"/></svg>` : `<svg width="13" height="13" viewBox="0 0 14 14" fill="none"><ellipse cx="7" cy="7" rx="5.5" ry="3.5" stroke="currentColor" stroke-width="1.3"/><line x1="2" y1="2" x2="12" y2="12" stroke="currentColor" stroke-width="1.3" stroke-linecap="round"/></svg>` }),
      h('button', { class:'icon-btn red', title:'Remove', onClick:(e)=>{e.stopPropagation();emit('remove')}, innerHTML:`<svg width="13" height="13" viewBox="0 0 14 14" fill="none"><polyline points="2,4 12,4" stroke="currentColor" stroke-width="1.3" stroke-linecap="round"/><path d="M5 4V3h4v1" stroke="currentColor" stroke-width="1.3" stroke-linecap="round"/><path d="M3 4l1 8h6l1-8" stroke="currentColor" stroke-width="1.3" stroke-linecap="round" stroke-linejoin="round"/></svg>` }),
    ])
  }
})

export const ToolbarBtn = defineComponent({
  props: ['icon','label','active','color'],
  emits: ['click'],
  setup(props, { emit }) {
    const hover = ref(false)

    const svgIcons = {
      point: `<svg width="14" height="14" viewBox="0 0 14 14" fill="none" xmlns="http://www.w3.org/2000/svg"><circle cx="7" cy="7" r="4.5" stroke="currentColor" stroke-width="1.5"/><circle cx="7" cy="7" r="1.5" fill="currentColor"/></svg>`,
      line: `<svg width="14" height="14" viewBox="0 0 14 14" fill="none" xmlns="http://www.w3.org/2000/svg"><line x1="2" y1="12" x2="12" y2="2" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/><circle cx="2" cy="12" r="1.5" fill="currentColor"/><circle cx="12" cy="2" r="1.5" fill="currentColor"/></svg>`,
      polygon: `<svg width="14" height="14" viewBox="0 0 14 14" fill="none" xmlns="http://www.w3.org/2000/svg"><polygon points="7,1.5 12.5,5 12.5,9 7,12.5 1.5,9 1.5,5" stroke="currentColor" stroke-width="1.5" fill="currentColor" fill-opacity="0.18"/></svg>`,
      edit: `<svg width="14" height="14" viewBox="0 0 14 14" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M9.5 2.5l2 2-7 7H2.5v-2l7-7z" stroke="currentColor" stroke-width="1.3" stroke-linejoin="round"/><line x1="8" y1="4" x2="10" y2="6" stroke="currentColor" stroke-width="1.3"/></svg>`,
      clear: `<svg width="14" height="14" viewBox="0 0 14 14" fill="none" xmlns="http://www.w3.org/2000/svg"><line x1="3" y1="3" x2="11" y2="11" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/><line x1="11" y1="3" x2="3" y2="11" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>`,
      import: `<svg width="14" height="14" viewBox="0 0 14 14" fill="none" xmlns="http://www.w3.org/2000/svg"><polyline points="4,6 7,9 10,6" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/><line x1="7" y1="2" x2="7" y2="9" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/><path d="M2 11h10" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>`,
      newlayer: `<svg width="14" height="14" viewBox="0 0 14 14" fill="none" xmlns="http://www.w3.org/2000/svg"><rect x="2" y="4" width="8" height="7" rx="1" stroke="currentColor" stroke-width="1.3"/><rect x="4" y="2" width="8" height="7" rx="1" stroke="currentColor" stroke-width="1.3"/><line x1="9" y1="8" x2="9" y2="12" stroke="currentColor" stroke-width="1.3" stroke-linecap="round"/><line x1="7" y1="10" x2="11" y2="10" stroke="currentColor" stroke-width="1.3" stroke-linecap="round"/></svg>`,
      vertex: `<svg width="14" height="14" viewBox="0 0 14 14" fill="none" xmlns="http://www.w3.org/2000/svg"><polyline points="2,11 5,4 9,8 12,3" stroke="currentColor" stroke-width="1.3" stroke-linecap="round" stroke-linejoin="round"/><circle cx="2" cy="11" r="1.8" fill="currentColor"/><circle cx="5" cy="4" r="1.8" fill="currentColor"/><circle cx="9" cy="8" r="1.8" fill="currentColor"/><circle cx="12" cy="3" r="1.8" fill="currentColor"/></svg>`,
      'sidebar-open': `<svg width="14" height="14" viewBox="0 0 14 14" fill="none" xmlns="http://www.w3.org/2000/svg"><rect x="1.5" y="1.5" width="11" height="11" rx="1.5" stroke="currentColor" stroke-width="1.3"/><line x1="5" y1="1.5" x2="5" y2="12.5" stroke="currentColor" stroke-width="1.3"/><polyline points="9,5 7,7 9,9" stroke="currentColor" stroke-width="1.3" stroke-linecap="round" stroke-linejoin="round"/></svg>`,
    }

    return () => h('button', {
      class: ['toolbar-btn', props.active ? 'toolbar-btn--active' : ''],
      style: { color: props.active ? (props.color || '#00e5ff') : hover.value ? (props.color||'#c9d1d9') : '#888' },
      title: props.label,
      onClick: () => emit('click'),
      onMouseenter: () => (hover.value = true),
      onMouseleave: () => (hover.value = false),
    }, [
      h('span', { class: 'btn-icon', innerHTML: svgIcons[props.icon] || props.icon, style:{display:'flex',alignItems:'center'} }),
      h('span', { style:{fontSize:'11px',letterSpacing:'0.5px'} }, props.label),
    ])
  }
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;600&display=swap');

* { box-sizing: border-box; margin: 0; padding: 0; }

/* Mobile warning overlay */
.mobile-warning {
  display: none;
  position: fixed; inset: 0;
  background: #0a0e14;
  z-index: 99999;
  align-items: center; justify-content: center;
  font-family: 'IBM Plex Mono', 'Courier New', monospace;
}
@media (max-width: 767px) {
  .mobile-warning { display: flex; }
  .gis-root { display: none !important; }
}
.mobile-warning-box {
  display: flex; flex-direction: column; align-items: center;
  gap: 16px; padding: 40px 32px;
  border: 1px solid rgba(255,145,0,0.3);
  border-radius: 8px;
  background: rgba(255,145,0,0.04);
  max-width: 320px; text-align: center;
}
.mobile-warning-title {
  color: #ff9100; font-size: 16px; font-weight: 700; letter-spacing: 3px;
}
.mobile-warning-msg {
  color: #888; font-size: 12px; line-height: 1.8;
}
.mobile-warning-sub {
  color: #444; font-size: 10px; letter-spacing: 1px;
  border-top: 1px solid #1c2333; padding-top: 12px; width: 100%;
}

/* Sidebar action buttons stacked vertically */
.sidebar-actions {
  display: flex; flex-direction: column; gap: 2px;
  padding: 6px 8px; border-bottom: 1px solid #1c2333;
}
.sidebar-actions .toolbar-btn {
  width: 100%; justify-content: flex-start;
  border: 1px solid rgba(255,255,255,0.06) !important;
  border-radius: 4px;
  padding: 5px 10px;
}
.sidebar-actions .toolbar-btn:hover {
  border-color: rgba(255,255,255,0.12) !important;
  background: rgba(255,255,255,0.04);
}

.gis-root {
  display: flex; flex-direction: column;
  height: 100vh; width: 100%;
  background: #0a0e14;
  font-family: 'IBM Plex Mono', 'Courier New', monospace;
  color: #c9d1d9; overflow: hidden;
}

/* Toolbar */
.toolbar {
  display: flex; align-items: center; gap: 6px;
  padding: 0 16px; height: 48px;
  background: #0d1117;
  border-bottom: 1px solid #1c2333;
  flex-shrink: 0; z-index: 1000;
}
.brand { color: #00e5ff; font-weight: 700; font-size: 15px; letter-spacing: 2px; margin-right: 8px; }
.divider { width: 1px; height: 24px; background: #1c2333; margin: 0 4px; }
.draw-hint { font-size: 11px; color: #666; margin-left: 6px; }

/* Body */
.body { display: flex; flex: 1; overflow: hidden; }

/* Sidebar */
.sidebar {
  width: 260px; background: #0d1117;
  border-right: 1px solid #1c2333;
  display: flex; flex-direction: column; flex-shrink: 0;
  overflow-y: auto;
}
.sidebar-header {
  padding: 10px 12px 6px; font-size: 10px;
  color: #444; letter-spacing: 2px;
  border-bottom: 1px solid #1c2333;
}
.layer-list { flex: 1; min-height: 80px; overflow-y: auto; }
.empty-hint {
  padding: 20px 12px; font-size: 11px;
  color: #333; text-align: center; line-height: 1.8;
}
.sidebar-footer {
  padding: 8px 10px; border-top: 1px solid #1c2333; font-size: 11px;
}
.footer-label { color: #444; letter-spacing: 1px; margin-bottom: 4px; font-size: 10px; }
.footer-name { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.footer-count { color: #555; margin-top: 2px; }
.export-btn {
  margin-top: 8px; width: 100%; padding: 5px 0;
  background: rgba(255,64,129,0.1); border: 1px solid rgba(255,64,129,0.3);
  border-radius: 3px; color: #ff4081; font-size: 11px; cursor: pointer;
  letter-spacing: 1px; font-family: inherit;
}
.export-btn:hover { background: rgba(255,64,129,0.2); }

/* Map */
.map-wrap { flex: 1; position: relative; }
.map { height: 100%; width: 100%; background: #0a0e14; }
.statusbar {
  position: absolute; bottom: 0; left: 0; right: 0;
  display: flex; align-items: center; gap: 10px;
  padding: 4px 12px; background: rgba(13,17,23,0.85);
  border-top: 1px solid #1c2333; font-size: 11px; color: #555;
  z-index: 1000; backdrop-filter: blur(4px);
}
.status-dot { color: #1c2333; }
.status-dot.active { color: #00e5ff; }
.draw-mode-tag { margin-left: auto; color: #ff9100; }

/* Sidebar slide transition */
.slide-enter-active, .slide-leave-active { transition: width 0.2s ease, opacity 0.2s ease; }
.slide-enter-from, .slide-leave-to { width: 0; opacity: 0; overflow: hidden; }
.slide-enter-to, .slide-leave-from { width: 260px; opacity: 1; }

/* Basemap picker */
.basemap-picker { position: relative; }
.basemap-dropdown {
  position: absolute; top: calc(100% + 4px); right: 0;
  background: #0d1117; border: 1px solid #1c2333;
  border-radius: 6px; min-width: 140px; z-index: 2000;
  box-shadow: 0 8px 24px rgba(0,0,0,0.6);
  overflow: hidden;
}
.basemap-item {
  display: flex; align-items: center; gap: 8px;
  padding: 7px 12px; font-size: 12px; cursor: pointer;
  color: #888; transition: background 0.12s, color 0.12s;
  font-family: 'IBM Plex Mono', monospace;
}
.basemap-item:hover { background: rgba(255,255,255,0.05); color: #c9d1d9; }
.basemap-item.active { color: #00e5ff; background: rgba(0,229,255,0.07); }
.bm-dot { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; border: 1px solid #333; }
</style>

<!-- Global styles for layer items, toolbar btns, leaflet popup -->
<style>
.layer-item {
  display: flex; align-items: center; gap: 8px;
  padding: 7px 10px; cursor: pointer; transition: all 0.15s;
  border-left: 3px solid transparent;
  border-radius: 0 4px 4px 0;
}
.layer-item:hover { background: rgba(255,255,255,0.03); }
.layer-item.active { background: rgba(0,229,255,0.06); }
.layer-name {
  flex: 1; font-size: 12px; font-family: 'IBM Plex Mono', monospace;
  overflow: hidden; text-overflow: ellipsis; white-space: nowrap;
}
.layer-count { font-size: 10px; color: #444; min-width: 20px; text-align: right; }
.icon-btn {
  background: none; border: none; cursor: pointer;
  color: #666; font-size: 13px; padding: 0 2px;
  font-family: inherit; line-height: 1;
}
.icon-btn:hover { color: #aaa; }
.icon-btn.red:hover { color: #ff4444; }

.toolbar-btn {
  display: flex; align-items: center; gap: 5px;
  padding: 4px 10px; border: 1px solid rgba(255,255,255,0.07); border-radius: 4px;
  background: transparent; cursor: pointer; font-size: 12px;
  font-family: 'IBM Plex Mono', monospace; transition: all 0.12s;
}
.toolbar-btn:hover { background: rgba(255,255,255,0.05); border-color: rgba(255,255,255,0.13); }
.toolbar-btn--active { background: rgba(0,229,255,0.1); border-color: rgba(0,229,255,0.35) !important; outline: none; }

.edit-marker { cursor: pointer !important; }

.rename-input:focus { outline: none; }

.popup-content {
  font-family: 'IBM Plex Mono', monospace;
  font-size: 12px; background: #0d1117;
  color: #c9d1d9; padding: 4px;
}
.leaflet-popup-content-wrapper,
.leaflet-popup-tip { background: #0d1117; border: 1px solid #1c2333; color: #c9d1d9; }

/* ───────────── RASTER MANAGER ───────────── */

.raster-section {
  display: flex;
  flex-direction: column;
  border-top: 1px solid #1c2333;
  border-bottom: 1px solid #1c2333;
}

.raster-list {
  max-height: 340px;
  overflow-y: auto;
}

.raster-item {
  padding: 8px;
  border-bottom: 1px solid rgba(255,255,255,0.04);
}
.raster-item.active { background: rgba(0,229,255,0.06); }

/* Tombol besar untuk aktivasi raster */
.raster-btn {
  width: 100%;
  display: flex; align-items: center; gap: 12px;
  padding: 14px 12px;
  min-height: 56px;
  background: #111822;
  border: 1px solid #1c2333;
  border-radius: 6px;
  color: #c9d1d9;
  font-family: 'IBM Plex Mono', monospace;
  text-align: left;
  cursor: pointer;
  transition: all 0.15s;
}
.raster-btn:hover { background: #172030; border-color: rgba(0,229,255,0.35); }
.raster-item.active .raster-btn { border-color: #00e5ff; background: rgba(0,229,255,0.1); }
.raster-item.loading .raster-btn { border-color: #ff9100; background: rgba(255,145,0,0.06); }

.raster-btn-icon {
  width: 28px; height: 28px; flex-shrink: 0;
  display: flex; align-items: center; justify-content: center;
  font-size: 22px; color: #555;
}
.raster-item.active .raster-btn-icon { color: #00e5ff; }

.raster-btn-text { display: flex; flex-direction: column; gap: 3px; min-width: 0; flex: 1; }
.raster-name {
  font-size: 12px; font-weight: 600; color: #c9d1d9;
  overflow: hidden; text-overflow: ellipsis; white-space: nowrap;
}
.raster-state { font-size: 10px; color: #666; }
.raster-item.active .raster-state  { color: #00e5ff; }
.raster-item.loading .raster-state { color: #ff9100; }

/* Spinner loading */
.spinner {
  width: 20px; height: 20px;
  border: 3px solid rgba(255,145,0,0.25);
  border-top-color: #ff9100;
  border-radius: 50%;
  animation: raster-spin 0.8s linear infinite;
}
@keyframes raster-spin { to { transform: rotate(360deg); } }

/* Progress bar indeterminate */
.raster-progress {
  height: 3px; margin-top: 6px;
  background: #1c2333; border-radius: 2px; overflow: hidden;
}
.raster-progress-bar {
  width: 40%; height: 100%; background: #ff9100;
  animation: raster-slide 1.1s ease-in-out infinite;
}
@keyframes raster-slide {
  0%   { transform: translateX(-100%); }
  100% { transform: translateX(250%); }
}

.raster-opacity {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 9px;
  font-size: 9px;
  color: #555;
}
.raster-opacity input {
  flex: 1;
  min-width: 0;
  height: 3px;
  accent-color: #00e5ff;
  cursor: pointer;
}
.raster-opacity span:last-child {
  min-width: 30px;
  text-align: right;
  color: #888;
}

.raster-error {
  margin-top: 6px;
  font-size: 10px;
  color: #ff6464;
  line-height: 1.5;
}

.brand {
  display: flex;
  align-items: center;
  
  height: 100%;
  gap: 8px;
}

.brand img {
  
  height: 100%;
  object-fit: contain;
  min-width: 0;
}

.map-legend {
  position: absolute;
  top: 20px;
  right: 20px;
  width: 220px;
  max-height: calc(100% - 40px);
  overflow-y: auto;
  z-index: 1000;

  background: #172030;
  color: #222222;
  border: 1px solid #e5e5e5;
  border-radius: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.12);
}

.legend-header {
  padding: 12px 14px;
  font-size: 13px;
  font-weight: 600;
  border-bottom: 1px solid #eeeeee;
}

.legend-content {
  padding: 12px 14px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 12px;
}

.legend-symbol {
  width: 14px;
  height: 14px;
  flex-shrink: 0;
  border: 1px solid rgba(0, 0, 0, 0.2);
  border-radius: 2px;
}

.legend-label {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.legend-empty {
  font-size: 12px;
  color: #999999;
}

/* ───────────── LEGEND SIDEBAR ───────────── */
.legend-sidebar {
  width: 240px; flex-shrink: 0;
  background: #0d1117;
  border-left: 1px solid #1c2333;
  display: flex; flex-direction: column;
  overflow-y: auto;
}
.legend-body { display: flex; flex-direction: column; }
.legend-block {
  padding: 12px;
  border-bottom: 1px solid #1c2333;
  display: flex; flex-direction: column; gap: 8px;
}
.legend-title {
  font-size: 12px; font-weight: 600; color: #c9d1d9;
  margin-bottom: 2px;
}
.legend-item {
  display: flex; align-items: center; gap: 10px;
  font-size: 11px; color: #9aa4b2;
}
.legend-symbol {
  width: 16px; height: 16px; flex-shrink: 0;
  border: 1px solid rgba(255,255,255,0.35);
  border-radius: 2px;
}
.legend-label {
  overflow: hidden; text-overflow: ellipsis; white-space: nowrap;
}
.legend-empty { font-size: 11px; color: #444; }
</style>