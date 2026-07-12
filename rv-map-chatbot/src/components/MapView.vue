<template>
  <div class="map-wrapper">
    <div ref="mapContainer" class="map-container"></div>

    <button
      type="button"
      class="map-button map-button-left"
      :class="{ active: updateOnMove }"
      @click="toggleUpdateOnMove"
    >
      <span class="toggle-icon" aria-hidden="true">
        <span class="toggle-track"></span>
        <span class="toggle-thumb"></span>
      </span>
      <span class="toggle-text">Update Map as It Moves</span>
    </button>

    <div class="map-button map-button-center" :class="{ alert: !updateOnMove && showMoveAlert }">
      <template v-if="updateOnMove">
        <span class="search-status" :class="{ active: isSearching }">
          <span class="search-dot"></span>
          {{ servicePointText }}
        </span>
      </template>
      <template v-else>
        <div class="move-status">
          <span v-if="showMoveAlert" class="move-alert-text">Your map position has changed.</span>
          <button
            v-if="showMoveAlert"
            type="button"
            class="search-area-button"
            @click="searchThisArea"
          >
            Search this area
          </button>
          <span v-else class="move-paused-text">Move the map to search this area.</span>
        </div>
      </template>
    </div>

    <div class="coord-overlay">
      <div class="coord-title">当前设备位置</div>
      <div class="coord-value">{{ deviceLngLat }}</div>
    </div>

    <div class="status-bar">
      <p><strong>当前地图中心：</strong> {{ centerLngLat }}</p>
      <p class="tip">提示：拖拽或放大缩小地图，即可触发范围更新</p>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onBeforeUnmount } from 'vue'
import mapboxgl from 'mapbox-gl'
import 'mapbox-gl/dist/mapbox-gl.css' // 必须引入样式，否则地图会错位

// 1. 填入你的 Mapbox Access Token
mapboxgl.accessToken = 'pk.eyJ1IjoiZHQwMzI0IiwiYSI6ImNtbnlvNW0zbDAzeTAycnB2eXg0MXNldmoifQ.6gAAqjLYlQUXD-TXmSjcLg'

const mapContainer = ref(null)
const map = ref(null)
const centerLngLat = ref('正在获取定位...')
const deviceLngLat = ref('正在获取定位...')
let deviceMarker = null
let targetMarker = null
let pendingFlyTarget = null
let shopMarkers = []
const props = defineProps({
  flyToTarget: Object,
})

const servicePointText = ref('Searching for service points...')
const isSearching = ref(false)
const updateOnMove = ref(true)
const showMoveAlert = ref(false)

const toggleUpdateOnMove = () => {
  updateOnMove.value = !updateOnMove.value
  if (updateOnMove.value) {
    showMoveAlert.value = false
  }
}

const executeMoveLogic = () => {
  if (!map.value) return

  const currentCenter = map.value.getCenter()
  formatCenterText(currentCenter)

  const bounds = map.value.getBounds()
  const sw = bounds.getSouthWest()
  const ne = bounds.getNorthEast()

  console.log('当前屏幕可视边界，准备发送给 AWS 后端：')
  console.log(`西南角: [${sw.lng}, ${sw.lat}], 东北角: [${ne.lng}, ${ne.lat}]`)
  fetchShopsFromAWS(sw.lng, sw.lat, ne.lng, ne.lat)
}


const AWS_API_BASE = 'https://ukvberwf88.execute-api.ap-southeast-2.amazonaws.com/dev/getShops'
const AWS_API_KEY = '7HDLyOvfOj3FUz8H7K5V72NQt10YAnpk1JJJ93QK' 
/**
 * 使用 GET 请求从 AWS API 获取店铺数据
 * 参数名采用 API 要求的查询字符串：swLng, swLat, neLng, neLat
 */
const fetchShopsFromAWS = async (swLng, swLat, neLng, neLat) => {
  try {
    const url = `${AWS_API_BASE}?swLng=${encodeURIComponent(swLng)}&swLat=${encodeURIComponent(swLat)}&neLng=${encodeURIComponent(neLng)}&neLat=${encodeURIComponent(neLat)}`

    isSearching.value = true
    const res = await fetch(url, {
      method: 'GET',
      headers: {
        'x-api-key': AWS_API_KEY,
      },
    })

    if (!res.ok) {
      console.error('AWS API 返回错误：', res.status, res.statusText)
      return null
    }

    const data = await res.json()
    console.log('从 AWS 获取到的店铺数据：', data)

    // API may return { data: [...], count } or plain array or { shops: [...] }
    let shops = []
    if (Array.isArray(data.shops)) shops = data.shops
    else if (Array.isArray(data.data)) shops = data.data
    else if (Array.isArray(data)) shops = data

    const totalShops = Number.isFinite(Number(data.count)) ? Number(data.count) : shops.length
    if (shops.length === 0) {
      servicePointText.value = 'There is no available service point near this location, please move the map to search for more'
    } else {
      servicePointText.value = `Showing ${shops.length} of ${totalShops} Service Point${shops.length !== 1 ? 's' : ''}`
    }

    console.log('normalized shops length:', shops.length)
    renderShopMarkers(shops)
    return data
  } catch (err) {
    console.error('fetchShopsFromAWS 错误：', err)
    return null
  } finally {
    isSearching.value = false
  }
}

// Remove existing shop markers from the map
const removeShopMarkers = () => {
  if (!shopMarkers || !shopMarkers.length) return
  shopMarkers.forEach((m) => m.remove())
  shopMarkers = []
}

// Create a simple circular marker element using the site's accent color
const createCircleElement = (fillColor = '#aa3bff') => {
  const el = document.createElement('div')
  el.className = 'shop-marker'
  // apply inline background color so each marker can use the accent
  el.style.background = fillColor
  return el
}

// Render shop markers from API results
const renderShopMarkers = (shops = []) => {
  if (!map.value) return
  console.log('renderShopMarkers called with', shops.length, 'shops')
  removeShopMarkers()

  // try to read accent color from CSS variables
  const cssAccent = getComputedStyle(document.documentElement).getPropertyValue('--accent') || ''
  const accent = cssAccent.trim() || '#aa3bff'

    shops.forEach((shop) => {
    const lat = Number(shop.latitude ?? shop.lat)
    const lng = Number(shop.longitude ?? shop.lng)
    if (!isFinite(lat) || !isFinite(lng)) return
    const pinEl = createCircleElement(accent)

    const marker = new mapboxgl.Marker({ element: pinEl, anchor: 'center' })
      .setLngLat([lng, lat])
      .addTo(map.value)

    const htmlString = `
      <div class="shop-popup">
        <h3>${shop.name || ''}</h3>
        <p>${shop.description || ''}</p>
        <a href="${shop.link || '#'}" target="_blank">Check in Google Maps</a>
      </div>
    `

    const popup = new mapboxgl.Popup({ offset: 12, closeButton: true, closeOnClick: true })
      .setHTML(htmlString)

    marker.setPopup(popup)
    pinEl.addEventListener('click', () => marker.togglePopup())

    shopMarkers.push(marker)
  })
}

const searchThisArea = () => {
  showMoveAlert.value = false
  executeMoveLogic()
}

const formatCenterText = (center) => {
  centerLngLat.value = `经度: ${center.lng.toFixed(4)}, 纬度: ${center.lat.toFixed(4)}`
}

const flyToLocation = (target) => {
  if (!map.value || !target?.location?.length || !map.value.loaded()) {
    pendingFlyTarget = target
    return
  }

  const coords = [Number(target.location[0]), Number(target.location[1])]

  setTargetMarker(coords)

  map.value.flyTo({
    center: coords,
    zoom: target.zoom || 14,
    speed: 1.2,
    curve: 1.4,
    essential: true,
  })


}

watch(
  () => props.flyToTarget,
  (next, prev) => {
    if (next && next.location?.length === 2) {
      flyToLocation(next)
    }
  }
)

const formatDeviceText = (location) => {
  deviceLngLat.value = `经度: ${location[0].toFixed(4)}, 纬度: ${location[1].toFixed(4)}`
}

const addDeviceMarker = (location) => {
  if (!map.value) return

  if (deviceMarker) {
    deviceMarker.remove()
  }

  const el = document.createElement('div')
  el.className = 'device-marker'
  el.innerHTML = '<span class="marker-dot map-marker"></span><span class="pulse-ring"></span>'

  deviceMarker = new mapboxgl.Marker(el)
    .setLngLat(location)
    .addTo(map.value)
}

const setTargetMarker = (location) => {
  if (!map.value || !location?.length) return

  if (targetMarker) {
    targetMarker.remove()
  }

  const el = document.createElement('div')
  el.className = 'map-marker target-marker'

  targetMarker = new mapboxgl.Marker(el)
    .setLngLat(location)
    .addTo(map.value)
}

// 默认中心点（如果你在家 Spotswood 附近测试，默认设在墨尔本）
const defaultCenter = [144.9631, -37.8136] 

onMounted(() => {
  // 2. 尝试获取浏览器当前用户的真实定位
  if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(
      (position) => {
        const { longitude, latitude } = position.coords
        initMap([longitude, latitude], [longitude, latitude])
      },
      (error) => {
        console.warn("无法获取精准定位，使用默认中心点:", error.message)
        deviceLngLat.value = '定位失败，请允许定位'
        initMap(defaultCenter)
      }
    )
  } else {
    deviceLngLat.value = '浏览器不支持定位'
    initMap(defaultCenter)
  }
})

// 3. 初始化地图函数
const initMap = (centerCoordinates, deviceCoordinates = null) => {
  map.value = new mapboxgl.Map({
    container: mapContainer.value,      // 挂载的 DOM 元素
    style: 'mapbox://styles/dt0324/cmrg6z4mz003001rh8i5tcm45', 
    center: centerCoordinates,          // 起始中心点 [lng, lat]
    zoom: 12,                           // 初始缩放层级
  })

  // 添加地图自带的导航控制条（放大、缩小、旋转）
  map.value.addControl(new mapboxgl.NavigationControl(), 'bottom-right')

  map.value.on('load', () => {
    formatCenterText(map.value.getCenter())
    if (deviceCoordinates) {
      formatDeviceText(deviceCoordinates)
      addDeviceMarker(deviceCoordinates)
    }
    if (pendingFlyTarget && pendingFlyTarget.location?.length === 2) {
      flyToLocation(pendingFlyTarget)
      pendingFlyTarget = null
    }
  })

  // 4. 核心事件监听：当地图停止移动（拖拽、缩放结束）时触发
  map.value.on('moveend', () => {
    if (updateOnMove.value) {
      executeMoveLogic()
    } else {
      showMoveAlert.value = true
    }
  })
}

// 销毁组件时释放地图资源，防止内存泄漏
onBeforeUnmount(() => {
  if (map.value) {
    map.value.remove()
  }
})
</script>

<style scoped>
.map-wrapper {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}

.map-container {
  width: 100%;
  height: 100%;
}

.status-bar {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 10;
  background: rgba(255, 255, 255, 0.95);
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  font-family: sans-serif;
  color: #333;
  max-width: 300px;
}

.search-status {
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

.search-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  border: 2px solid rgba(255, 255, 255, 0.95);
  background: var(--accent);
  box-shadow: 0 0 0 0 rgba(170, 59, 255, 0.4);
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.search-status.active .search-dot {
  animation: searchPulse 1s infinite ease-in-out;
}

@keyframes searchPulse {
  0%, 100% {
    transform: scale(1);
    box-shadow: 0 0 0 0 rgba(170, 59, 255, 0.4);
  }
  50% {
    transform: scale(1.2);
    box-shadow: 0 0 0 4px rgba(170, 59, 255, 0.12);
  }
}

.coord-overlay {
  position: absolute;
  top: 20px;
  right: 20px;
  z-index: 12;
  background: rgba(245, 245, 245, 0.9);
  color: #2f3744;
  padding: 14px 18px;
  border-radius: 16px;
  box-shadow: 0 18px 30px rgba(0, 0, 0, 0.08);
  backdrop-filter: blur(12px);
  animation: fadeInUp 0.45s ease-out both;
}

.coord-title {
  margin: 0 0 6px;
  font-size: 11px;
  color: #6b7280;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

.coord-value {
  margin: 0;
  font-size: 14px;
  font-weight: 600;
  color: #111827;
}

@keyframes fadeInUp {
  0% {
    opacity: 0;
    transform: translateY(14px);
  }
  100% {
    opacity: 1;
    transform: translateY(0);
  }
}

.map-wrapper {
  --map-overlay-top: clamp(4.8rem, 6vw, 6.5rem);
}

.map-button {
  position: absolute;
  z-index: 14;
  padding: 10px 16px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.95);
  border: 1px solid rgba(31, 41, 55, 0.12);
  color: #111827;
  font-size: 14px;
  font-weight: 600;
  box-shadow: 0 12px 24px rgba(15, 23, 42, 0.08);
  cursor: pointer;
  transition: transform 0.24s ease, box-shadow 0.24s ease, background-color 0.24s ease;
  display: inline-flex;
  align-items: center;
  gap: 10px;
  border-radius: 1.75rem;
}

.map-button:hover {
  box-shadow: 0 18px 32px rgba(15, 23, 42, 0.14);
}

.map-button-left:hover {
  transform: translateY(-1px);
}

.map-button-center:hover {
  transform: translateX(-50%) translateY(-1px);
}

.map-button-left {
  top: var(--map-overlay-top);
  left: 20px;
}

.map-button-center {
  top: var(--map-overlay-top);
  left: 50%;
  transform: translateX(-50%);
  min-width: 240px;
  padding: 12px 18px;
  justify-content: center;
}

.map-button-center.alert {
  background: #eff6ff;
  border-color: rgba(59, 130, 246, 0.25);
}

.map-button-center .move-status {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  flex-wrap: wrap;
  justify-content: center;
}

.move-alert-text {
  font-size: 14px;
  font-weight: 600;
  color: #111827;
}

.move-paused-text {
  font-size: 13px;
  color: rgba(17, 24, 39, 0.7);
}

.search-area-button {
  border: none;
  border-radius: 999px;
  padding: 9px 16px;
  background: #1d4ed8;
  color: #ffffff;
  font-size: 13px;
  font-weight: 700;
  cursor: pointer;
  transition: transform 0.18s ease, background-color 0.18s ease;
}

.search-area-button:hover {
  background: #2563eb;
  transform: translateY(-1px);
}

.map-button-left .toggle-icon {
  position: relative;
  width: 42px;
  height: 24px;
  display: inline-flex;
  align-items: center;
}

.map-button-left .toggle-track {
  position: absolute;
  inset: 0;
  border-radius: 999px;
  background: rgba(15, 23, 42, 0.08);
  transition: background-color 0.24s ease;
}

.map-button-left .toggle-thumb {
  position: absolute;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #fff;
  border: 1px solid rgba(31, 41, 55, 0.16);
  box-shadow: 0 6px 18px rgba(15, 23, 42, 0.12);
  left: 4px;
  transition: transform 0.24s ease, background-color 0.24s ease, border-color 0.24s ease;
}

.map-button-left.active .toggle-track {
  background: #1d4ed8;
}

.map-button-left.active .toggle-thumb {
  transform: translateX(18px);
  background: #eff6ff;
  border-color: #bfdbfe;
}

.map-button-left .toggle-text {
  display: inline-flex;
  align-items: center;
}

.status-bar {
  top: calc(var(--map-overlay-top) + 3.5rem);
}

.coord-overlay {
  top: calc(var(--map-overlay-top) + 7rem);
}

@media (max-width: 640px) {
  .map-button {
    padding: 10px 14px;
    font-size: 13px;
    width: calc(100% - 20px);
    max-width: none;
    justify-content: flex-start;
    box-sizing: border-box;
  }

  .map-button-left,
  .map-button-center {
    left: 10px;
    transform: none;
    right: 10px;
  }

  .map-button-left {
    top: calc(var(--map-overlay-top) + 1.5rem);
  }

  .map-button-center {
    top: calc(var(--map-overlay-top) + 4.8rem);
    min-width: 0;
    margin-top: 0;
    justify-content: flex-start;
  }

  .status-bar,
  .coord-overlay {
    display: none;
  }
}

.tip {
  font-size: 11px !important;
  color: #666;
}
</style>

<style>
.device-marker {
  position: relative;
  width: 20px;
  height: 20px;
}

.map-marker {
  position: absolute;
  inset: 0;
  border-radius: 50%;
  border: 2px solid rgba(255, 255, 255, 0.95);
  box-shadow: 0 0 0 4px rgba(15, 23, 42, 0.24);
}

.target-marker {
  width: 16px;
  height: 16px;
  background: #121317;
  box-shadow: 0 0 0 6px rgba(18, 19, 23, 0.22);
}

.device-marker .marker-dot {
  position: absolute;
  inset: 0;
  border-radius: 50%;
  background: #676666;
  border: 1px solid rgba(31, 41, 55, 0.16);
  box-shadow: inset 0 0 0 2px rgba(148, 163, 184, 0.24);
}

.device-marker .pulse-ring {
  position: absolute;
  inset: 0;
  border-radius: 50%;
  background: rgba(148, 163, 184, 0.24);
  animation: pulse 1.8s ease-out infinite;
}

@keyframes pulse {
  0% {
    transform: scale(0.8);
    opacity: 0.85;
  }
  70% {
    transform: scale(2.05);
    opacity: 0;
  }
  100% {
    opacity: 0;
  }
}

.shop-marker {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  display: inline-block;
  line-height: 0;
  cursor: pointer;
  will-change: transform;
  border: 2px solid rgba(255,255,255,0.95);
  box-shadow: 0 6px 12px rgba(0,0,0,0.18);
}

.shop-popup {
  font-family: sans-serif;
  color: #111;
  max-width: 220px;
}

.shop-popup h3 {
  margin: 0 0 8px;
  font-size: 14px;
  line-height: 1.2;
}

.shop-popup p {
  margin: 0 0 10px;
  font-size: 13px;
  line-height: 1.5;
  color: #4b5563;
}

.shop-popup a {
  color: var(--accent);
  text-decoration: none;
  font-weight: 600;
}
</style>