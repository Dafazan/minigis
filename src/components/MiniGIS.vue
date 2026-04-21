<template>
  <div class="gis-root">

    <!-- ── TOP TOOLBAR ── -->
    <div class="toolbar">
      <span class="brand">DAFAZAN's SHP VIEWER</span>

      <input ref="fileInput" type="file" multiple accept=".shp,.dbf,.shx,.prj" @change="handleImport" style="display:none" />
      <ToolbarBtn icon="↑" label="Import SHP" @click="fileInput.click()" />
      <ToolbarBtn icon="+" label="New Layer" @click="createNewLayer" />

      <div class="divider" />

      <ToolbarBtn icon="◉" label="Point"   :active="drawMode === 'point'"   color="#76ff03" @click="toggleDraw('point')" />
      <ToolbarBtn icon="╱" label="Line"    :active="drawMode === 'line'"    color="#00e5ff" @click="toggleDraw('line')" />
      <ToolbarBtn icon="▭" label="Polygon" :active="drawMode === 'polygon'" color="#ff9100" @click="toggleDraw('polygon')" />

      <span v-if="drawMode" class="draw-hint">
        {{ drawMode === 'point' ? 'Click to place point' : `Click vertices • Double-click to finish ${drawMode}` }}
      </span>

      <div style="flex:1" />

      <template v-if="activeLayer">
        <ToolbarBtn icon="↓" :label="`Export &quot;${activeLayer.name}&quot;`" color="#ff4081" @click="exportLayer(activeLayer)" />
        <ToolbarBtn icon="✕" label="Clear" color="#ff4444" @click="clearActiveLayer" />
      </template>

      <ToolbarBtn :icon="sidebarOpen ? '◧' : '▣'" label="Layers" @click="sidebarOpen = !sidebarOpen" />
    </div>

    <!-- ── BODY ── -->
    <div class="body">

      <!-- ── SIDEBAR ── -->
      <transition name="slide">
        <div v-if="sidebarOpen" class="sidebar">
          <div class="sidebar-header">LAYERS {{ layers.length ? `(${layers.length})` : '' }}</div>

          <div class="layer-list">
            <div v-if="!layers.length" class="empty-hint">
              No layers yet.<br>Import a shapefile<br>or create a new layer.
            </div>
            <LayerItem
              v-for="layer in [...layers].reverse()"
              :key="layer.id"
              :layer="layer"
              :isActive="layer.id === activeLayerId"
              :color="layer.color"
              @select="activeLayerId = layer.id"
              @toggle="toggleLayer(layer.id)"
              @remove="removeLayer(layer.id)"
              @zoom="zoomToLayer(layer)"
            />
          </div>

          <div v-if="activeLayer" class="sidebar-footer">
            <div class="footer-label">ACTIVE LAYER</div>
            <div class="footer-name" :style="{ color: activeLayer.color }">{{ activeLayer.name }}</div>
            <div class="footer-count">{{ activeLayer.geojson?.features?.length || 0 }} features</div>
            <button class="export-btn" @click="exportLayer(activeLayer)">↓ EXPORT SHP</button>
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
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount, nextTick } from 'vue'
import L from 'leaflet'
import shp from 'shpjs'
import JSZip from 'jszip'
import 'leaflet/dist/leaflet.css'

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
const sidebarOpen = ref(true)
const status      = ref('Ready')

let map = null
const leafletLayers = {} // layerId → L.geoJSON layer
let drawingPoints = []
let tempLayer = null

const activeLayer = computed(() => layers.value.find(l => l.id === activeLayerId.value) || null)

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
  map = L.map(mapEl.value, { zoomControl: false }).setView([0, 0], 3)
  L.tileLayer('https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png', {
    attribution: '© CARTO', maxZoom: 19,
  }).addTo(map)
  L.control.zoom({ position: 'bottomright' }).addTo(map)
})

onBeforeUnmount(() => {
  if (map) map.remove()
})

// ── Draw mode ─────────────────────────────────────────────────────────────────
watch(drawMode, (mode) => {
  if (!map) return
  map.off('click', onMapClick)
  map.off('dblclick', onMapDblClick)
  clearTemp()
  drawingPoints = []
  map.getContainer().style.cursor = mode ? 'crosshair' : ''
  if (mode) {
    map.on('click', onMapClick)
    if (mode !== 'point') map.on('dblclick', onMapDblClick)
  }
})

function toggleDraw(mode) {
  drawMode.value = drawMode.value === mode ? null : mode
}

function onMapClick(e) {
  if (!drawMode.value) return
  if (drawMode.value === 'point') {
    addFeature({ type: 'Point', coordinates: [e.latlng.lng, e.latlng.lat] })
    return
  }
  drawingPoints.push([e.latlng.lat, e.latlng.lng])
  clearTemp()
  if (drawingPoints.length > 1) {
    const opts = { color: '#00e5ff', weight: 2, dashArray: '6,4', fillOpacity: 0.1 }
    tempLayer = (drawMode.value === 'line'
      ? L.polyline(drawingPoints, opts)
      : L.polygon(drawingPoints, opts)
    ).addTo(map)
  }
}

function onMapDblClick(e) {
  const mode = drawMode.value
  if (mode === 'line' && drawingPoints.length >= 2) {
    const coords = drawingPoints.map(([lat, lng]) => [lng, lat])
    addFeature({ type: 'LineString', coordinates: coords })
  } else if (mode === 'polygon' && drawingPoints.length >= 3) {
    const coords = [...drawingPoints.map(([lat, lng]) => [lng, lat])]
    coords.push(coords[0])
    addFeature({ type: 'Polygon', coordinates: [coords] })
  }
  drawingPoints = []
  clearTemp()
}

function clearTemp() {
  if (tempLayer) { map.removeLayer(tempLayer); tempLayer = null }
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

function removeLayer(id) {
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

function readFileBuffer(file) {
  return new Promise((res, rej) => {
    const r = new FileReader()
    r.onload = e => res(e.target.result)
    r.onerror = rej
    r.readAsArrayBuffer(file)
  })
}

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
  emits: ['select','toggle','remove','zoom'],
  setup(props, { emit }) {
    const typeIcon = { Point:'◉',MultiPoint:'◉',LineString:'╱',MultiLineString:'╱',Polygon:'▭',MultiPolygon:'▭' }
    const icon = () => typeIcon[props.layer.geojson?.features?.[0]?.geometry?.type] || '◈'
    return () => h('div', {
      class: ['layer-item', props.isActive ? 'active' : ''],
      style: { borderLeftColor: props.isActive ? props.color : 'transparent' },
      onClick: () => emit('select'),
    }, [
      h('span', { style: { color: props.color, fontSize:'14px', minWidth:'16px' } }, icon()),
      h('span', { class:'layer-name', style:{ color: props.layer.visible ? '#e0e0e0' : '#555' } }, props.layer.name),
      h('span', { class:'layer-count' }, props.layer.geojson?.features?.length || 0),
      h('button', { class:'icon-btn', title:'Zoom', onClick:(e)=>{e.stopPropagation();emit('zoom')} }, '⊕'),
      h('button', { class:'icon-btn', style:{color: props.layer.visible ? props.color : '#444'}, title:'Toggle', onClick:(e)=>{e.stopPropagation();emit('toggle')} }, props.layer.visible?'◉':'○'),
      h('button', { class:'icon-btn red', title:'Remove', onClick:(e)=>{e.stopPropagation();emit('remove')} }, '✕'),
    ])
  }
})

export const ToolbarBtn = defineComponent({
  props: ['icon','label','active','color'],
  emits: ['click'],
  setup(props, { emit }) {
    const hover = ref(false)
    return () => h('button', {
      class: ['toolbar-btn', props.active ? 'toolbar-btn--active' : ''],
      style: { color: props.active ? '#00e5ff' : hover.value ? (props.color||'#c9d1d9') : '#888' },
      title: props.label,
      onClick: () => emit('click'),
      onMouseenter: () => (hover.value = true),
      onMouseleave: () => (hover.value = false),
    }, [
      h('span', { style:{fontSize:'14px'} }, props.icon),
      h('span', { style:{fontSize:'11px',letterSpacing:'0.5px'} }, props.label),
    ])
  }
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;600&display=swap');

* { box-sizing: border-box; margin: 0; padding: 0; }

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
  width: 240px; background: #0d1117;
  border-right: 1px solid #1c2333;
  display: flex; flex-direction: column; flex-shrink: 0;
}
.sidebar-header {
  padding: 10px 12px 6px; font-size: 10px;
  color: #444; letter-spacing: 2px;
  border-bottom: 1px solid #1c2333;
}
.layer-list { flex: 1; overflow-y: auto; }
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
.slide-enter-to, .slide-leave-from { width: 240px; opacity: 1; }
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
  padding: 4px 10px; border: none; border-radius: 4px;
  background: transparent; cursor: pointer; font-size: 12px;
  font-family: 'IBM Plex Mono', monospace; transition: all 0.12s;
}
.toolbar-btn:hover { background: rgba(255,255,255,0.05); }
.toolbar-btn--active { background: rgba(0,229,255,0.1); outline: 1px solid rgba(0,229,255,0.3); }

.popup-content {
  font-family: 'IBM Plex Mono', monospace;
  font-size: 12px; background: #0d1117;
  color: #c9d1d9; padding: 4px;
}
.leaflet-popup-content-wrapper,
.leaflet-popup-tip { background: #0d1117; border: 1px solid #1c2333; color: #c9d1d9; }
</style>