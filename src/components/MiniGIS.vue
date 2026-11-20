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

  <div class="gis-root" :data-theme="theme">

    <!-- ── TOP BAR ── -->
    <header class="topbar">
      <span class="brand">
        <img src="https://res.cloudinary.com/djm8hc5vg/image/upload/v1793903697/a0a3d6c55c13ebbfe324b4d6115d7751.png" alt="">
        <img src="https://res.cloudinary.com/djm8hc5vg/image/upload/v1793903697/cropped-The-Education-University.png" alt="">
      </span>

      <input ref="fileInput" type="file" multiple accept=".shp,.dbf,.shx,.prj" @change="handleImport" style="display:none" />

      <div class="topbar-spacer" />

      <div class="theme-switch" role="group" aria-label="Tema tampilan">
        <button type="button" :class="['theme-opt', theme === 'light' ? 'active' : '']" :aria-pressed="theme === 'light'" title="Mode terang" @click="setTheme('light')">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><circle cx="12" cy="12" r="4"/><path d="M12 2v2M12 20v2M4.9 4.9l1.4 1.4M17.7 17.7l1.4 1.4M2 12h2M20 12h2M4.9 19.1l1.4-1.4M17.7 6.3l1.4-1.4"/></svg>
          <span>Terang</span>
        </button>
        <button type="button" :class="['theme-opt', theme === 'dark' ? 'active' : '']" :aria-pressed="theme === 'dark'" title="Mode gelap" @click="setTheme('dark')">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path d="M20 14.5A8 8 0 019.5 4 8 8 0 1020 14.5z"/></svg>
          <span>Gelap</span>
        </button>
      </div>

      <div class="basemap-picker">
        <button
          type="button"
          class="toolbar-btn toolbar-btn--solid"
          title="Ganti basemap"
          aria-haspopup="true"
          :aria-expanded="basemapOpen"
          @click="basemapOpen = !basemapOpen"
        >
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path d="M9 4L3 6v14l6-2 6 2 6-2V4l-6 2-6-2zM9 4v14M15 6v14"/></svg>
          <span>Ganti basemap</span>
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true" style="color:var(--muted)"><path d="M6 9l6 6 6-6"/></svg>
        </button>
        <div v-if="basemapOpen" class="basemap-dropdown">
          <button
            type="button"
            v-for="bm in basemaps" :key="bm.id"
            :class="['basemap-item', currentBasemap.id === bm.id ? 'active' : '']"
            @click="setBasemap(bm)"
          >
            <span class="bm-dot" :style="{ background: bm.dot }"></span>
            {{ bm.label }}
          </button>
        </div>
      </div>
    </header>

    <!-- ── BODY ── -->
    <div class="body">

      <!-- ── SIDEBAR ── -->
      <transition name="slide">
        <aside v-if="sidebarOpen" class="sidebar">
          <div class="sidebar-scroll">

            <!-- VECTOR LAYERS -->
            <section class="panel-section">
              <div class="section-head">
                <span class="section-title">Layer vektor</span>
                <span class="count-pill">{{ layers.length }}</span>
              </div>

              <div class="action-grid">
                <ToolbarBtn icon="import" label="Import SHP" @click="fileInput.click()" />
                <ToolbarBtn icon="newlayer" label="Layer baru" @click="createNewLayer" />
              </div>

              <div class="layer-list">
                <div v-if="!layers.length" class="empty-hint">
                  Belum ada layer.<br>Import shapefile atau buat layer baru.
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
            </section>
<!-- BLOK -->
<section class="panel-section">
  <div class="section-head">
    <span class="section-title">Blok</span>
    <span class="count-pill">{{ blok.count }}</span>
  </div>

  <div v-if="!blok.geojson" class="empty-hint">Memuat data BLOK…</div>

  <div v-else class="layer-item" style="cursor:default">
    <span class="layer-chip" :style="{ borderColor: BLOK_COLOR, background: 'transparent' }"></span>
    <div class="layer-text">
      <span :class="['layer-name', blok.visible ? '' : 'is-hidden']">BLOK</span>
      <span class="layer-meta">{{ blok.count }} fitur · poligon · klik untuk lihat data</span>
    </div>
    <button type="button" class="icon-btn" title="Zoom ke BLOK" aria-label="Zoom ke BLOK" @click="zoomToBlok">
      <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><circle cx="11" cy="11" r="6"/><path d="M20 20l-4.5-4.5"/></svg>
    </button>
    <button type="button" :class="['icon-btn', blok.visible ? 'on' : '']"
            :title="blok.visible ? 'Sembunyikan BLOK' : 'Tampilkan BLOK'"
            :aria-label="blok.visible ? 'Sembunyikan BLOK' : 'Tampilkan BLOK'"
            @click="toggleBlok">
      <svg v-if="blok.visible" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path d="M2 12s4-7 10-7 10 7 10 7-4 7-10 7S2 12 2 12z"/><circle cx="12" cy="12" r="3"/></svg>
      <svg v-else width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path d="M2 12s4-7 10-7 10 7 10 7-4 7-10 7S2 12 2 12z"/><circle cx="12" cy="12" r="3"/><path d="M3 3l18 18"/></svg>
    </button>
  </div>
</section>
            <!-- RASTER MANAGER -->
            <section class="panel-section raster-section">
              <div class="section-head">
                <span class="section-title">Layer raster</span>
                <span class="count-pill">{{ rasterLayers.length }}</span>
              </div>

              <div v-if="!rasterLayers.length" class="empty-hint">
                File raster tidak ditemukan.
              </div>

              <div v-for="cat in rasterCategories" :key="cat.name" class="raster-category">
                <button
                  type="button"
                  class="category-header"
                  :aria-expanded="!!expandedCategories[cat.name]"
                  @click="toggleCategory(cat.name)"
                >
                  <svg
                    :class="['category-arrow', expandedCategories[cat.name] ? 'open' : '']"
                    width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"
                  ><path d="M6 9l6 6 6-6"/></svg>
                  <span class="category-name">{{ cat.name }}</span>
                  <span v-if="cat.hasLoading" class="category-dot loading"></span>
                  <span v-else-if="cat.hasActive" class="category-dot active"></span>
                  <span class="category-count">{{ cat.items.length }} layer</span>
                </button>

                <div v-show="expandedCategories[cat.name]" class="category-body">
                  <div class="year-grid">
                    <button
                      type="button"
                      v-for="raster in cat.items"
                      :key="raster.id"
                      :class="['year-chip', raster.visible ? 'active' : '', raster.loading ? 'loading' : '']"
                      :aria-pressed="raster.visible"
                      :title="raster.name + (raster.loading ? ' — memuat… (klik untuk batal)' : raster.visible ? ' — aktif, klik untuk sembunyikan' : ' — klik untuk tampilkan')"
                      @click="toggleRaster(raster)"
                    >
                      <span v-if="raster.loading" class="spinner"></span>
                      <span class="chip-text">{{ chipLabel(raster) }}</span>
                    </button>
                  </div>

                  <div v-if="cat.activeItem" :class="['raster-active', cat.activeItem.loading ? 'loading' : '']">
                    <div class="raster-active-head">
                      <span class="raster-active-name">
                        <span class="raster-dot"></span>{{ cat.activeItem.name }}
                      </span>
                      <span class="raster-active-state">
                        {{ cat.activeItem.loading ? 'Memuat…' : 'Aktif' }}
                      </span>
                    </div>

                    <div v-if="cat.activeItem.loading" class="raster-progress">
                      <div class="raster-progress-bar"></div>
                    </div>

                    <label class="raster-opacity">
                      <span class="raster-opacity-label">Opasitas</span>
                      <input
                        type="range" min="0" max="100"
                        :value="cat.activeItem.opacity"
                        aria-label="Opasitas raster"
                        @input="changeRasterOpacity(cat.activeItem, $event)"
                      />
                      <span class="raster-opacity-val">{{ cat.activeItem.opacity }}%</span>
                    </label>
                  </div>

                  <div
                    v-for="raster in cat.items.filter(r => r.error)"
                    :key="raster.id + '-error'"
                    class="raster-error"
                  >
                    {{ raster.name }}: {{ raster.error }}
                  </div>
                </div>
              </div>
            </section>
          </div>

          <footer v-if="activeLayer" class="sidebar-footer">
            <div class="footer-row">
              <span class="section-title">Layer aktif</span>
              <span class="footer-info">
                <span class="footer-name" :style="{ color: activeLayer.color }">{{ activeLayer.name }}</span>
                · {{ activeLayer.geojson?.features?.length || 0 }} fitur
              </span>
            </div>
            <button type="button" class="export-btn" @click="exportLayer(activeLayer)">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path d="M12 3v12m0 0l-4-4m4 4l4-4M4 17v3h16v-3"/></svg>
              Ekspor SHP
            </button>
          </footer>
        </aside>
      </transition>

      <!-- ── MAP ── -->
      <div class="map-wrap">
        <div ref="mapEl" class="map" />

        <!-- Floating drawing toolbar -->
        <div class="map-toolbar" role="toolbar" aria-label="Alat gambar">
          <div class="tool-group">
            <ToolbarBtn icon="point"   label="Titik"   :active="drawMode === 'point'"   @click="toggleDraw('point')" />
            <ToolbarBtn icon="line"    label="Garis"   :active="drawMode === 'line'"    @click="toggleDraw('line')" />
            <ToolbarBtn icon="polygon" label="Poligon" :active="drawMode === 'polygon'" @click="toggleDraw('polygon')" />

            <div class="tool-sep" />

            <ToolbarBtn icon="vertex" label="Vertex"      :active="showVertices" @click="showVertices = !showVertices" />
            <ToolbarBtn icon="edit"   label="Hapus fitur" :active="editMode"     @click="toggleEditMode" />
            <ToolbarBtn v-if="activeLayer" icon="clear" label="Bersihkan" @click="clearActiveLayer" />
          </div>

          <span v-if="drawMode" class="draw-hint">
            {{ drawMode === 'point' ? 'Klik peta untuk menaruh titik' : 'Klik untuk menambah vertex • Klik tengah mouse untuk menyelesaikan' }}
          </span>

          <div class="tool-spacer" />

          <div v-if="activeRaster" class="layer-badge">
            <span class="raster-dot"></span>
            <span>{{ activeRaster.category }} · {{ chipLabel(activeRaster) }}</span>
            <small>opasitas {{ activeRaster.opacity }}%</small>
          </div>
        </div>

        <!-- Floating legend -->
        <div v-if="rasterLegend || visibleVectorLayers.length || (blok.geojson && blok.visible)" class="map-legend">
          <div>LEGENDA</div>
          <div v-if="rasterLegend" class="legend-block">
            
            <div class="legend-title">{{ rasterLegend.title }}</div>
            <div v-for="item in rasterLegend.items" :key="item.label" class="legend-item">
              <span class="legend-symbol" :style="{ background: item.color }"></span>
              <span class="legend-label">{{ item.label }}</span>
            </div>
            
          </div>

          <div v-if="visibleVectorLayers.length" class="legend-block">
            <div v-for="layer in visibleVectorLayers" :key="layer.id" class="legend-item">
              <span class="legend-symbol legend-symbol--vector"
                    :style="{ background: layer.color + '33', borderColor: layer.color }"></span>
              <span class="legend-label">{{ layer.name }}</span>
              
            </div>
          </div>

          <div v-if="blok.geojson && blok.visible" class="legend-block">
    <div class="legend-item">
      <span class="legend-symbol legend-symbol--vector" :style="{ background: 'transparent', borderColor: BLOK_COLOR }"></span>
      <span class="legend-label">BLOK</span>
    </div>
  </div>
  
        </div>

        <!-- Status bar -->
        <div class="statusbar">
          <span :class="['status-dot', status !== 'Ready' ? 'active' : '']"></span>
          <span>{{ status }}</span>
          <span v-if="drawMode" class="draw-mode-tag">DRAW: {{ drawMode.toUpperCase() }}</span>
        </div>
      </div>

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
      name: 'Area SPTH',
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

// ── BLOK (shapefile khusus, terpisah dari Layer vektor) ───────────────────
const BLOK_COLOR = '#ffd740'
const blok = ref({ geojson: null, visible: true, count: 0 })
let blokLeaflet = null

const escapeHtml = (s) =>
  String(s).replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]))

// Ambil atribut tanpa peduli huruf besar/kecil (Blok / BLOK / blok)
function getProp(props, name) {
  const key = Object.keys(props || {}).find(k => k.toLowerCase() === name.toLowerCase())
  return key !== undefined ? props[key] : null
}

function blokPopupHtml(props) {
  const nama = getProp(props, 'Blok')
  const luas = getProp(props, 'Luas_Ha')
  const luasNum = Number(luas)
  const luasText = luas === null || luas === '' ? '-'
    : Number.isNaN(luasNum) ? escapeHtml(luas)
    : luasNum.toLocaleString('id-ID', { maximumFractionDigits: 2 })

  return `<div class="popup-content blok-popup">
    <div class="blok-row"><b>${nama === null ? '-' : escapeHtml(nama)}</b></div>
    <div class="blok-row"><span>Luas: </span><b>${luasText}</b> Ha</div>
  </div>`
}

async function loadBlokLayer() {
  try {
    const files = ['blok.shp', 'blok.shx', 'blok.dbf', 'blok.prj', 'blok.cpg']
    for (const f of ['blok.shp', 'blok.shx', 'blok.dbf']) {
      if (!vectorFileMap[f]) throw new Error(`Missing file: ${f}`)
    }

    const zip = new JSZip()
    for (const f of files) {
      const url = vectorFileMap[f]
      if (!url) continue
      const res = await fetch(url)
      if (!res.ok) throw new Error(`Failed to fetch ${f}`)
      zip.file(f, await res.arrayBuffer())
    }

    const geojson = await shp(await zip.generateAsync({ type: 'arraybuffer' }))
    blok.value = {
      geojson: markRaw(geojson),
      visible: true,
      count: geojson.features?.length || 0,
    }
    renderBlok()
  } catch (error) {
    console.error('Blok loading error:', error)
    showStatus(`Failed to load BLOK: ${error.message}`)
  }
}

function renderBlok() {
  if (!map) return
  if (blokLeaflet) { map.removeLayer(blokLeaflet); blokLeaflet = null }
  if (!blok.value.geojson || !blok.value.visible) return

  const baseStyle = { color: BLOK_COLOR, weight: 2, opacity: 1, fill: true, fillColor: BLOK_COLOR, fillOpacity: 0 }

  blokLeaflet = L.geoJSON(blok.value.geojson, {
    pane: 'blokPane',
    style: () => baseStyle,
    onEachFeature: (f, fl) => {
      fl.on('mouseover', () => fl.setStyle({ weight: 3.5, fillOpacity: 0.08 }))
      fl.on('mouseout',  () => fl.setStyle(baseStyle))
      fl.on('click', (e) => {
        // jangan ganggu saat menggambar / menghapus fitur
        if (drawMode.value || editMode.value) return
        L.popup({ closeButton: true })
          .setLatLng(e.latlng)
          .setContent(blokPopupHtml(f.properties))
          .openOn(map)
      })
    },
  }).addTo(map)
}

function toggleBlok() {
  blok.value.visible = !blok.value.visible
  renderBlok()
}

function zoomToBlok() {
  if (!blokLeaflet) return
  const b = blokLeaflet.getBounds()
  if (b.isValid()) map.fitBounds(b, { padding: [40, 40] })
}
// ── RASTER MANAGER ────────────────────────────────────────

const rasterLayers = ref([])

const rasterModules = import.meta.glob('/src/rasters/*/*.{tif,tiff}', {
  eager: true,
  query: '?url',
  import: 'default'
})

// path: /src/rasters/[kategori]/[file].tif
const availableRasters = Object.entries(rasterModules).map(([path, url]) => {
  const parts = path.split('/')
  const name = parts[parts.length - 1]
  const category = parts[parts.length - 2]
  return { id: `${category}/${name}`, category, name, url }
})

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

// ── Tema (terang / gelap) ─────────────────────────────────
const THEME_KEY = 'spth-theme'
function getInitialTheme() {
  try {
    const saved = localStorage.getItem(THEME_KEY)
    if (saved === 'light' || saved === 'dark') return saved
  } catch (e) {}
  return 'dark'
}
const theme = ref(getInitialTheme())
function setTheme(t) {
  theme.value = t
  try { localStorage.setItem(THEME_KEY, t) } catch (e) {}
}

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
const currentBasemap = ref(basemaps.find(b => b.id === theme.value) || basemaps[0])

// Ikuti tema: kalau basemap yang dipakai Dark/Light, ikut berganti
watch(theme, (t, prev) => {
  if (map && currentBasemap.value.id === prev) {
    const bm = basemaps.find(b => b.id === t)
    if (bm) setBasemap(bm)
  }
  if (showVertices.value) layers.value.forEach(l => syncLeafletLayer(l.id))
})

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

const activeRaster = computed(() => rasterLayers.value.find(r => r.visible) || null)

const RASTER_LEGENDS = {
  ndvi: {
    title: 'Klasifikasi NDVI',
    items: [
      { color: '#a52a2a', label: 'Non Vegetasi' },
      { color: '#ffff00', label: 'Vegetasi Sangat Rendah' },
      { color: '#90ee90', label: 'Vegetasi Rendah' },
      { color: '#008000', label: 'Vegetasi Sedang' },
      { color: '#006400', label: 'Vegetasi Tinggi' },
    ],
  },
  change: {
    title: 'Perubahan Berdasarkan NDVI',
    items: [
      { color: '#ff0000', label: 'Vegetasi Hilang' },
      { color: '#ffffff', label: 'Vegetasi Stabil' },
      { color: '#008000', label: 'Vegetasi Bertambah' },
    ],
  },
  tutupan_lahan: {
    title: 'Tutupan Lahan',
    items: [
      { color: '#1a5bab', label: 'Air' },
      { color: '#358221', label: 'Pohon' },
      { color: '#87d19e', label: 'Vegetasi Tergenang' },
      { color: '#ffdb5c', label: 'Lahan Pertanian' },
      { color: '#ed022a', label: 'Area Terbangun' },
      { color: '#ede9e3', label: 'Lahan Terbuka' },
    ],
  },
}

// Urutan penting: "change" dicek lebih dulu, karena nama seperti
// "ndvi_change_2023" mengandung kata "ndvi" juga.
const LEGEND_RULES = [
  { key: 'tutupan_lahan', keywords: ['tutupan lahan', 'tutupan', 'landcover', 'land cover', 'lulc'] },
  { key: 'change',        keywords: ['change', 'cumulative', 'perubahan'] },
  { key: 'ndvi',          keywords: ['ndvi'] },
]

const rasterLegend = computed(() => {
  const r = activeRaster.value
  if (!r) return null

  // Samakan underscore/strip jadi spasi, supaya "tutupan_lahan" dan
  // "Tutupan-Lahan" tetap cocok dengan keyword "tutupan lahan"
  const text = `${r.category} ${r.name}`
    .toLowerCase()
    .replace(/[_-]+/g, ' ')

  const rule = LEGEND_RULES.find(rule =>
    rule.keywords.some(k => text.includes(k))
  )
  return rule ? RASTER_LEGENDS[rule.key] : null
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

  // pane khusus BLOK (di atas overlayPane = 400)
  map.createPane('blokPane').style.zIndex = 450

  baseTileLayer = L.tileLayer(currentBasemap.value.url, {
    attribution: currentBasemap.value.attribution, maxZoom: 19,
  }).addTo(map)

  L.control.zoom({ position: 'bottomright' }).addTo(map)

  // close basemap dropdown on map click
  map.on('click', () => { basemapOpen.value = false })

  initializeRasterLayers()
  await loadDefaultVectorLayer()
  await loadBlokLayer()
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

const expandedCategories = ref({})

function initializeRasterLayers() {
  rasterLayers.value = availableRasters.map(raster => ({
    id: raster.id,
    category: raster.category,
    name: raster.name,
    url: raster.url,
    georaster: null,
    layer: null,
    visible: false,
    loading: false,
    error: null,
    opacity: 80
  }))

  // buka kategori pertama secara default
  const first = rasterLayers.value[0]?.category
  if (first) expandedCategories.value[first] = true
}

// Kelompokkan raster per kategori (folder)
const rasterCategories = computed(() => {
  const groups = {}
  rasterLayers.value.forEach(r => {
    (groups[r.category] ||= []).push(r)
  })
  return Object.entries(groups)
    .sort(([a], [b]) => a.localeCompare(b))
    .map(([name, items]) => ({
      name,
      items: items.sort((a, b) => a.name.localeCompare(b.name)),
      hasActive: items.some(i => i.visible),
      hasLoading: items.some(i => i.loading),
      activeItem: items.find(i => i.visible || i.loading) || null,
    }))
})

function toggleCategory(name) {
  expandedCategories.value[name] = !expandedCategories.value[name]
}

// Label chip: nama file tanpa ekstensi (2021.tif -> 2021)
function chipLabel(raster) {
  return raster.name.replace(/\.(tif|tiff)$/i, '')
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
            radius: 3, color: layer.color, fillColor: theme.value === 'light' ? '#ffffff' : '#0a0e14',
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

const svg = (inner, size = 16) =>
  `<svg width="${size}" height="${size}" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">${inner}</svg>`

export const LayerItem = defineComponent({
  props: ['layer','isActive','color'],
  emits: ['select','toggle','remove','zoom','rename'],
  setup(props, { emit }) {
    const geomLabel = {
      Point: 'titik', MultiPoint: 'titik',
      LineString: 'garis', MultiLineString: 'garis',
      Polygon: 'poligon', MultiPolygon: 'poligon',
    }
    const count = () => props.layer.geojson?.features?.length || 0
    const meta = () => {
      const t = geomLabel[props.layer.geojson?.features?.[0]?.geometry?.type]
      return t ? `${count()} fitur · ${t}` : `${count()} fitur`
    }

    const icons = {
      zoom:   svg('<circle cx="11" cy="11" r="6"/><path d="M20 20l-4.5-4.5"/>'),
      eye:    svg('<path d="M2 12s4-7 10-7 10 7 10 7-4 7-10 7S2 12 2 12z"/><circle cx="12" cy="12" r="3"/>'),
      eyeOff: svg('<path d="M2 12s4-7 10-7 10 7 10 7-4 7-10 7S2 12 2 12z"/><circle cx="12" cy="12" r="3"/><path d="M3 3l18 18"/>'),
      trash:  svg('<path d="M4 7h16M9 7V4h6v3M6 7l1 13h10l1-13"/>'),
    }

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
      'data-layer-id': props.layer.id,
      onClick: () => emit('select'),
    }, [
      h('span', {
        class: 'layer-chip',
        style: { borderColor: props.color, background: props.color + '40' },
      }),
      h('div', { class: 'layer-text' }, [
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
            })
          : h('span', {
              class: ['layer-name', props.layer.visible ? '' : 'is-hidden'],
              title: 'Klik dua kali untuk ganti nama',
              onDblclick: startRename,
            }, props.layer.name),
        h('span', { class: 'layer-meta' }, meta()),
      ]),
      h('button', {
        type: 'button', class: 'icon-btn',
        title: 'Zoom ke layer', 'aria-label': `Zoom ke ${props.layer.name}`,
        onClick: (e) => { e.stopPropagation(); emit('zoom') },
        innerHTML: icons.zoom,
      }),
      h('button', {
        type: 'button', class: ['icon-btn', props.layer.visible ? 'on' : ''],
        title: props.layer.visible ? 'Sembunyikan layer' : 'Tampilkan layer',
        'aria-label': props.layer.visible ? `Sembunyikan ${props.layer.name}` : `Tampilkan ${props.layer.name}`,
        onClick: (e) => { e.stopPropagation(); emit('toggle') },
        innerHTML: props.layer.visible ? icons.eye : icons.eyeOff,
      }),
      h('button', {
        type: 'button', class: 'icon-btn red',
        title: 'Hapus layer', 'aria-label': `Hapus ${props.layer.name}`,
        onClick: (e) => { e.stopPropagation(); emit('remove') },
        innerHTML: icons.trash,
      }),
    ])
  }
})

export const ToolbarBtn = defineComponent({
  props: ['icon','label','active','color'],
  emits: ['click'],
  setup(props, { emit }) {
    const svgIcons = {
      point:   svg('<circle cx="12" cy="12" r="3"/><path d="M12 3v3M12 18v3M3 12h3M18 12h3"/>'),
      line:    svg('<path d="M5 19L19 5"/><circle cx="5" cy="19" r="1.5"/><circle cx="19" cy="5" r="1.5"/>'),
      polygon: svg('<path d="M12 3l8 6-3 11H7L4 9l8-6z"/>'),
      edit:    svg('<path d="M7 20h13M5 14l8-8 6 6-6 6H8l-3-4z"/>'),
      clear:   svg('<path d="M6 6l12 12M18 6L6 18"/>'),
      import:  svg('<path d="M12 3v12m0 0l-4-4m4 4l4-4M4 17v3h16v-3"/>'),
      newlayer: svg('<path d="M12 5v14M5 12h14"/>'),
      vertex:  svg('<path d="M4 18l5-9 5 6 6-9"/><circle cx="4" cy="18" r="1.5"/><circle cx="9" cy="9" r="1.5"/><circle cx="14" cy="15" r="1.5"/><circle cx="20" cy="6" r="1.5"/>'),
      'sidebar-open': svg('<rect x="3" y="4" width="18" height="16" rx="2"/><path d="M9 4v16M15 9l-3 3 3 3"/>'),
    }

    return () => h('button', {
      type: 'button',
      class: ['toolbar-btn', props.active ? 'toolbar-btn--active' : ''],
      title: props.label,
      'aria-pressed': props.active === undefined ? undefined : !!props.active,
      onClick: () => emit('click'),
    }, [
      h('span', { class: 'btn-icon', innerHTML: svgIcons[props.icon] || props.icon }),
      h('span', { class: 'btn-label' }, props.label),
    ])
  }
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;500&family=IBM+Plex+Sans:wght@400;500;600&display=swap');

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

/* Root + design tokens */
.gis-root {
  --bg: #0b110f;
  --panel: #121916;
  --panel-2: #18211e;
  --line: #25312d;
  --text: #e7f0ea;
  --muted: #93a69c;
  --accent: #2dd4bf;
  --accent-hi: #5eead4;
  --accent-bg: #0f2f2a;
  --accent-line: #1f6e63;
  --glass: rgba(18, 25, 22, 0.92);
  --glass-strong: rgba(18, 25, 22, 0.94);
  --card: #151d1a;
  --card-hover: #35453f;
  --hover: rgba(255, 255, 255, 0.06);
  --dot-idle: #3a4a44;
  --on-accent: #04201b;
  --warn: #ffb84d;
  --danger: #ff6b6b;
  --err: #ff8a8a;
  --shadow: 0 12px 32px rgba(0, 0, 0, 0.5);
  --legend-border: rgba(255, 255, 255, 0.2);
  --font: 'IBM Plex Sans', system-ui, sans-serif;
  --mono: 'IBM Plex Mono', 'Courier New', monospace;

  display: flex; flex-direction: column;
  height: 100vh; width: 100%;
  background: var(--bg);
  font-family: var(--font);
  font-size: 13px; line-height: 1.4;
  color: var(--text); overflow: hidden;
}

/* Tema terang */
.gis-root[data-theme='light'] {
  --bg: #eef3f0;
  --panel: #ffffff;
  --panel-2: #f1f5f3;
  --line: #d5dfda;
  --text: #14201b;
  --muted: #566a60;
  --accent: #0f766e;
  --accent-hi: #0b5f58;
  --accent-bg: #d9f3ee;
  --accent-line: #7cc9bd;
  --glass: rgba(255, 255, 255, 0.92);
  --glass-strong: rgba(255, 255, 255, 0.95);
  --card: #f7faf8;
  --card-hover: #bccbc4;
  --hover: rgba(20, 32, 27, 0.06);
  --dot-idle: #b7c4be;
  --on-accent: #ffffff;
  --warn: #b45309;
  --danger: #dc2626;
  --err: #b91c1c;
  --shadow: 0 12px 32px rgba(20, 32, 27, 0.18);
  --legend-border: rgba(0, 0, 0, 0.2);
}

/* Top bar */
.topbar {
  position: relative; z-index: 1100;
  display: flex; align-items: center; gap: 16px;
  padding: 0 16px; height: 56px;
  background: var(--panel);
  border-bottom: 1px solid var(--line);
  flex-shrink: 0;
}
.brand {
  display: flex; align-items: center;
  height: 100%; gap: 8px; padding: 8px 0;
}
.brand img { height: 100%; object-fit: contain; min-width: 0; }
.topbar-spacer { flex: 1; }

/* Body */
.body { display: flex; flex: 1; min-height: 0; overflow: hidden; }

/* Sidebar */
.sidebar {
  width: 320px; background: var(--panel);
  border-right: 1px solid var(--line);
  display: flex; flex-direction: column; flex-shrink: 0;
  min-height: 0;
}
.sidebar-scroll { flex: 1; min-height: 0; overflow-y: auto; }

.panel-section {
  padding: 16px;
  display: flex; flex-direction: column; gap: 12px;
  border-bottom: 1px solid var(--line);
}
.section-head { display: flex; align-items: center; justify-content: space-between; }
.section-title {
  font-size: 11px; font-weight: 600;
  letter-spacing: 0.08em; text-transform: uppercase;
  color: var(--muted);
}
.count-pill {
  font-family: var(--mono); font-size: 11px; color: var(--muted);
  background: var(--panel-2); border-radius: 10px; padding: 1px 8px;
}
.action-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 8px; }
.layer-list { display: flex; flex-direction: column; gap: 8px; }
.empty-hint {
  padding: 14px 12px; font-size: 12px;
  color: var(--muted); text-align: center; line-height: 1.7;
  border: 1px dashed var(--line); border-radius: 10px;
}

.sidebar-footer {
  padding: 16px; border-top: 1px solid var(--line);
  display: flex; flex-direction: column; gap: 10px; flex-shrink: 0;
}
.footer-row { display: flex; align-items: baseline; justify-content: space-between; gap: 8px; min-width: 0; }
.footer-info {
  font-size: 12px; color: var(--text); min-width: 0;
  overflow: hidden; text-overflow: ellipsis; white-space: nowrap;
}
.footer-name { font-weight: 500; }
.export-btn {
  display: flex; align-items: center; justify-content: center; gap: 8px;
  height: 40px; border: 0; border-radius: 8px;
  background: var(--accent); color: var(--on-accent);
  font-family: inherit; font-size: 13px; font-weight: 600; cursor: pointer;
  transition: background 0.12s;
}
.export-btn:hover { background: var(--accent-hi); }

/* Map */
.map-wrap { flex: 1; position: relative; min-width: 0; }
.map { height: 100%; width: 100%; background: var(--bg); }

.map-toolbar {
  position: absolute; top: 16px; left: 16px; right: 16px;
  z-index: 1000;
  display: flex; align-items: flex-start; flex-wrap: wrap; gap: 8px;
  pointer-events: none;
}
.map-toolbar > * { pointer-events: auto; }
.tool-group {
  display: flex; align-items: center; gap: 2px; padding: 4px;
  border-radius: 12px; background: var(--glass);
  border: 1px solid var(--line); backdrop-filter: blur(6px);
}
.tool-sep { width: 1px; height: 22px; background: var(--line); margin: 0 6px; }
.tool-spacer { flex: 1 1 0; }
.draw-hint, .layer-badge {
  display: flex; align-items: center; gap: 10px;
  height: 44px; padding: 0 14px; border-radius: 12px;
  background: var(--glass); border: 1px solid var(--line);
  backdrop-filter: blur(6px);
}
.draw-hint { font-size: 12px; color: var(--muted); }
.layer-badge { font-weight: 500; }
.layer-badge small { font-family: var(--mono); font-size: 11px; font-weight: 400; color: var(--muted); }

.statusbar {
  position: absolute; bottom: 0; left: 0; right: 0;
  display: flex; align-items: center; gap: 8px;
  height: 32px; padding: 0 16px;
  background: var(--glass);
  border-top: 1px solid var(--line);
  font-family: var(--mono); font-size: 12px; color: var(--muted);
  z-index: 1000; backdrop-filter: blur(4px);
}
.status-dot { width: 8px; height: 8px; border-radius: 50%; background: var(--dot-idle); flex-shrink: 0; }
.status-dot.active { background: var(--accent); }
.draw-mode-tag { margin-left: auto; color: var(--warn); }

/* Sidebar slide transition */
.slide-enter-active, .slide-leave-active { transition: width 0.2s ease, opacity 0.2s ease; }
.slide-enter-from, .slide-leave-to { width: 0; opacity: 0; overflow: hidden; }
.slide-enter-to, .slide-leave-from { width: 320px; opacity: 1; }

/* Basemap picker */
.basemap-picker { position: relative; }
.basemap-dropdown {
  position: absolute; top: calc(100% + 6px); right: 0;
  background: var(--panel); border: 1px solid var(--line);
  border-radius: 10px; min-width: 168px; z-index: 2000; padding: 4px;
  box-shadow: var(--shadow);
  display: flex; flex-direction: column;
}
.basemap-item {
  display: flex; align-items: center; gap: 10px;
  padding: 8px 10px; border: 0; border-radius: 6px;
  background: transparent; text-align: left;
  font-family: inherit; font-size: 13px; cursor: pointer;
  color: var(--muted); transition: background 0.12s, color 0.12s;
}
.basemap-item:hover { background: var(--hover); color: var(--text); }
.basemap-item.active { color: var(--accent-hi); background: var(--accent-bg); }
.bm-dot { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; border: 1px solid var(--line); }
</style>

<!-- Global styles for layer items, toolbar btns, raster manager, legend, leaflet -->
<style>
/* Buttons & inputs inherit the app font */
.gis-root button, .gis-root input { font-family: inherit; }

.toolbar-btn {
  display: flex; align-items: center; gap: 8px;
  height: 36px; padding: 0 12px;
  border: 1px solid transparent; border-radius: 8px;
  background: transparent; color: var(--text);
  font-size: 13px; white-space: nowrap; cursor: pointer;
  transition: background 0.12s, border-color 0.12s, color 0.12s;
}
.toolbar-btn:hover { background: var(--hover); }
.toolbar-btn--active { background: var(--accent-bg); border-color: var(--accent-line); color: var(--accent-hi); }
.toolbar-btn--solid,
.action-grid .toolbar-btn { background: var(--panel-2); border-color: var(--line); }
.action-grid .toolbar-btn { justify-content: center; }
.btn-icon { display: flex; align-items: center; }

.toolbar-btn:focus-visible, .year-chip:focus-visible, .icon-btn:focus-visible,
.export-btn:focus-visible, .category-header:focus-visible, .basemap-item:focus-visible {
  outline: 2px solid var(--accent); outline-offset: 2px;
}

/* Vector layer items */
.layer-item {
  display: flex; align-items: center; gap: 12px;
  padding: 10px 12px; cursor: pointer;
  border-radius: 10px; background: var(--card);
  border: 1px solid var(--line);
  transition: background 0.15s, border-color 0.15s;
}
.layer-item, .layer-item * { box-sizing: border-box; }
.layer-item:hover { border-color: var(--card-hover); }
.layer-item.active { background: var(--accent-bg); border-color: var(--accent-line); }
.layer-chip { width: 14px; height: 14px; border-radius: 4px; border: 2px solid; flex-shrink: 0; }
.layer-text { flex: 1; min-width: 0; display: flex; flex-direction: column; }
.layer-name {
  font-size: 13px; font-weight: 500; color: var(--text);
  overflow: hidden; text-overflow: ellipsis; white-space: nowrap;
}
.layer-name.is-hidden { color: var(--muted); }
.layer-meta { font-size: 11px; color: var(--muted); }
.rename-input {
  width: 100%; min-width: 0; padding: 2px 6px;
  background: var(--panel-2); border: 1px solid var(--accent);
  border-radius: 6px; color: var(--text); font-size: 13px; outline: none;
}
.icon-btn {
  width: 28px; height: 28px; padding: 0; flex-shrink: 0;
  display: flex; align-items: center; justify-content: center;
  background: none; border: 0; border-radius: 6px;
  color: var(--muted); cursor: pointer; line-height: 1;
}
.icon-btn:hover { background: var(--hover); color: var(--text); }
.icon-btn.on { color: var(--accent); }
.icon-btn.red:hover { color: var(--danger); }

/* Raster manager */
.raster-category {
  border: 1px solid var(--line); border-radius: 10px;
  background: var(--card); overflow: hidden;
}
.category-header {
  width: 100%; height: 40px; padding: 0 12px;
  display: flex; align-items: center; gap: 8px;
  background: transparent; border: 0;
  color: var(--text); font-size: 13px; font-weight: 500;
  cursor: pointer; text-align: left;
}
.category-header:hover { background: var(--hover); }
.category-arrow { color: var(--muted); flex-shrink: 0; transform: rotate(-90deg); transition: transform 0.15s; }
.category-arrow.open { transform: none; }
.category-name { flex: 1; min-width: 0; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.category-count { font-family: var(--mono); font-size: 11px; color: var(--muted); }
.category-dot { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; }
.category-dot.active  { background: var(--accent); box-shadow: 0 0 6px var(--accent); }
.category-dot.loading { background: #ff9100; }

.category-body { padding: 0 12px 12px; display: flex; flex-direction: column; gap: 12px; }

.year-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 8px; }
.year-chip {
  display: flex; align-items: center; justify-content: center; gap: 6px;
  height: 36px; min-width: 0; padding: 0 6px;
  border-radius: 8px; background: var(--panel-2);
  border: 1px solid var(--line); color: var(--text);
  font-family: var(--mono); font-size: 13px; cursor: pointer;
  transition: background 0.12s, border-color 0.12s, color 0.12s;
}
.chip-text { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.year-chip:hover { border-color: var(--accent-line); }
.year-chip.active { background: var(--accent-bg); border-color: var(--accent); color: var(--accent-hi); }
.year-chip.loading { border-color: #ff9100; color: var(--warn); }

.raster-active {
  display: flex; flex-direction: column; gap: 8px;
  padding: 10px 12px; border-radius: 8px;
  background: var(--accent-bg); border: 1px solid var(--accent-line);
}
.raster-active.loading { background: rgba(255,145,0,0.08); border-color: rgba(255,145,0,0.45); }
.raster-active-head { display: flex; align-items: center; justify-content: space-between; gap: 8px; }
.raster-active-name {
  display: flex; align-items: center; gap: 8px; min-width: 0;
  font-family: var(--mono); overflow: hidden; text-overflow: ellipsis; white-space: nowrap;
}
.raster-dot { width: 8px; height: 8px; border-radius: 50%; background: var(--accent); flex-shrink: 0; }
.raster-active.loading .raster-dot { background: #ff9100; }
.raster-active-state { font-size: 11px; color: var(--accent-hi); }
.raster-active.loading .raster-active-state { color: var(--warn); }

.raster-opacity { display: flex; align-items: center; gap: 10px; font-size: 11px; color: var(--muted); }
.raster-opacity-label { letter-spacing: 0.06em; text-transform: uppercase; }
.raster-opacity input { flex: 1; min-width: 0; accent-color: var(--accent); cursor: pointer; }
.raster-opacity-val { min-width: 32px; text-align: right; font-family: var(--mono); color: var(--text); }

.raster-error { font-size: 11px; color: var(--err); line-height: 1.5; }

/* Spinner loading */
.spinner {
  width: 12px; height: 12px; flex-shrink: 0;
  border: 2px solid rgba(255,145,0,0.25);
  border-top-color: #ff9100;
  border-radius: 50%;
  animation: raster-spin 0.8s linear infinite;
}
@keyframes raster-spin { to { transform: rotate(360deg); } }

/* Progress bar indeterminate */
.raster-progress { height: 3px; background: var(--line); border-radius: 2px; overflow: hidden; }
.raster-progress-bar {
  width: 40%; height: 100%; background: #ff9100;
  animation: raster-slide 1.1s ease-in-out infinite;
}
@keyframes raster-slide {
  0%   { transform: translateX(-100%); }
  100% { transform: translateX(250%); }
}

/* Floating legend */
.map-legend {
  position: absolute; left: 16px; bottom: 48px; z-index: 1000;
  width: 224px; max-height: calc(100% - 140px); overflow-y: auto;
  padding: 12px 14px; border-radius: 12px;
  background: var(--glass-strong); border: 1px solid var(--line);
  display: flex; flex-direction: column; gap: 10px;
}
.legend-block { display: flex; flex-direction: column; gap: 8px; }
.legend-block + .legend-block { padding-top: 10px; border-top: 1px solid var(--line); }
.legend-title {
  font-size: 11px; font-weight: 600; letter-spacing: 0.08em;
  text-transform: uppercase; color: var(--muted); margin-bottom: 2px;
}
.legend-item { display: flex; align-items: center; gap: 10px; font-size: 12px; color: var(--text); }
.legend-symbol {
  width: 14px; height: 14px; flex-shrink: 0;
  border-radius: 4px; border: 1px solid var(--legend-border);
}
.legend-symbol--vector { border-width: 2px; }
.legend-label { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

/* Leaflet controls in the same style */
.gis-root .leaflet-bottom { margin-bottom: 40px; }
.gis-root .leaflet-bar { border: 1px solid var(--line) !important; border-radius: 12px; overflow: hidden; box-shadow: none; }
.gis-root .leaflet-bar a,
.gis-root .leaflet-touch .leaflet-bar a {
  width: 40px; height: 40px; line-height: 40px; font-size: 18px;
  background: var(--glass-strong); color: var(--text);
  border-bottom: 1px solid var(--line);
}
.gis-root .leaflet-bar a:last-child { border-bottom: 0; }
.gis-root .leaflet-bar a:hover { background: var(--panel-2); color: var(--accent-hi); }
.gis-root .leaflet-control-attribution { background: var(--glass) !important; color: var(--muted); }
.gis-root .leaflet-control-attribution a { color: var(--muted); }

.edit-marker { cursor: pointer !important; }

.popup-content {
  font-family: 'IBM Plex Mono', monospace;
  font-size: 12px; background: var(--panel);
  color: var(--text); padding: 4px;
}
.leaflet-popup-content-wrapper,
.leaflet-popup-tip { background: var(--panel); border: 1px solid var(--line); color: var(--text); }

/* Pilihan tema */
.theme-switch {
  display: flex; gap: 2px; padding: 2px;
  background: var(--panel-2); border: 1px solid var(--line); border-radius: 10px;
}
.theme-opt {
  display: flex; align-items: center; gap: 6px;
  height: 30px; padding: 0 10px;
  border: 0; border-radius: 8px; background: transparent;
  color: var(--muted); font-size: 12px; cursor: pointer;
  transition: background 0.12s, color 0.12s;
}
.theme-opt:hover { color: var(--text); }
.theme-opt.active { background: var(--accent-bg); color: var(--accent-hi); box-shadow: inset 0 0 0 1px var(--accent-line); }
.theme-opt:focus-visible { outline: 2px solid var(--accent); outline-offset: 2px; }

/* Raster tampil dengan pixel tajam (tanpa smoothing) */
.leaflet-image-layer {
  image-rendering: pixelated;
  image-rendering: -moz-crisp-edges;
  image-rendering: crisp-edges;

  /* Popup BLOK */
.blok-popup { display: flex; flex-direction: column; gap: 4px; min-width: 130px; }
.blok-row { display: flex; justify-content: space-between; gap: 16px; }
.blok-row span { color: var(--muted); }
.blok-row b { font-weight: 600; text-align: right; }
}
</style>