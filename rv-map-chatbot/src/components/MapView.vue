<template>
  <div class="map-wrapper">
    <div v-if="!(isMobile && mobileViewMode === 'list')" ref="mapContainer" class="map-container"></div>

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
          <span v-else class="move-paused-text">{{ servicePointText }}</span>
        </div>
      </template>
    </div>

    <!-- Right-hand shop list: shows closest shops and supports infinite scroll -->
    <div v-if="!(isMobile && mobileViewMode === 'map')" class="shop-list">
      <div class="shop-list-header">
          <div class="shop-header-title inline">
            Service point near
            <button
              type="button"
              class="inline-btn"
              :class="{ active: listMode === 'nearby' }"
              @click="setListMode('nearby')"
            >
              current
            </button>
            <span class="sep">/</span>
            <button
              type="button"
              class="inline-btn"
              :class="{ active: listMode === 'searched', disabled: !canUseSearchCoordinates }"
              :disabled="!canUseSearchCoordinates"
              @click="setListMode('searched')"
            >
              searched
            </button>
            location
          </div>
        </div>
      <ul class="shop-list-items">
        <li v-for="(shop, idx) in visibleShops" :key="idx" class="shop-list-item" @click="openShopFromList(idx)">
          <div class="shop-item-left">
            <div class="shop-name">{{ shop.name }}</div>
            <div class="shop-desc">{{ shop.description }}</div>
          </div>
          <div class="shop-item-right">{{ shop.distance != null ? Number(shop.distance).toFixed(2) + ' km' : '-' }}</div>
        </li>
      </ul>
      <div v-if="visibleShops.length < latestShops.length" class="shop-list-loading">
        <button type="button" class="load-more-btn" @click="loadMore">Load more</button>
      </div>
    </div>

    <!-- Mobile mode switch (visible on small screens) -->
    <div class="mobile-switch" v-if="isMobile">
      <button :class="{ active: mobileViewMode === 'map' }" @click="setMobileMode('map')">Map</button>
      <button :class="{ active: mobileViewMode === 'list' }" @click="setMobileMode('list')">List</button>
    </div>

  </div>
</template>

<script setup>
import { ref, watch, onMounted, onBeforeUnmount, computed } from 'vue'
import mapboxgl from 'mapbox-gl'
import 'mapbox-gl/dist/mapbox-gl.css' // 必须引入样式，否则地图会错位

// 1. 填入你的 Mapbox Access Token
mapboxgl.accessToken = 'pk.eyJ1IjoiZHQwMzI0IiwiYSI6ImNtbnlvNW0zbDAzeTAycnB2eXg0MXNldmoifQ.6gAAqjLYlQUXD-TXmSjcLg'

const mapContainer = ref(null)
const map = ref(null)
const centerLngLat = ref('正在获取定位...')
const deviceLngLat = ref('正在获取定位...')
const deviceCoords = ref(null)
let deviceMarker = null
let targetMarker = null
let pendingFlyTarget = null
let shopMarkers = []
const latestShops = ref([])
const visibleCount = ref(5)
const visibleShops = computed(() => latestShops.value.slice(0, visibleCount.value))
const listMode = ref('nearby')
const searchedLocation = ref(null)
const searchedPlaceName = ref('')
const props = defineProps({
  flyToTarget: Object,
})

// Mobile view state: 'map' or 'list'
const isMobile = ref(false)
const mobileViewMode = ref('map')

const setMobileMode = (mode) => {
  if (mode !== 'map' && mode !== 'list') return
  mobileViewMode.value = mode
}

const updateIsMobile = () => {
  isMobile.value = window.matchMedia('(max-width: 640px)').matches
  // when switching to desktop size, ensure map/list are visible
  if (!isMobile.value) mobileViewMode.value = 'map'
}

const servicePointText = ref('Searching for service points...')
const isSearching = ref(false)
const updateOnMove = ref(false)
const showMoveAlert = ref(false)

const canUseSearchCoordinates = computed(() => Array.isArray(searchedLocation.value) && searchedLocation.value.length === 2)
const listHeaderText = computed(() => {
  if (listMode.value === 'searched' && canUseSearchCoordinates.value) {
    return 'Service Point Near the Searched Address'
  }
  return 'Nearby Service Points'
})
const listToggleLabel = computed(() => (listMode.value === 'searched' ? 'Use device location' : 'Use searched address'))

const toggleUpdateOnMove = () => {
  updateOnMove.value = !updateOnMove.value
  if (updateOnMove.value) {
    showMoveAlert.value = false
  }
}

const toggleListMode = () => {
  if (!canUseSearchCoordinates.value) return
  listMode.value = listMode.value === 'nearby' ? 'searched' : 'nearby'
  if (map.value) {
    executeMoveLogic()
  }
}

const setListMode = (mode) => {
  if (mode === 'searched' && !canUseSearchCoordinates.value) return
  if (mode !== 'searched' && mode !== 'nearby') return
  listMode.value = mode
  if (map.value) executeMoveLogic()
}

watch(canUseSearchCoordinates, (available) => {
  if (!available && listMode.value === 'searched') {
    listMode.value = 'nearby'
  }
})

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
    let url = `${AWS_API_BASE}?swLng=${encodeURIComponent(swLng)}&swLat=${encodeURIComponent(swLat)}&neLng=${encodeURIComponent(neLng)}&neLat=${encodeURIComponent(neLat)}`
    // 如果已知用户坐标，附加到请求以便后端计算距离和排序
    const sourceCoords = listMode.value === 'searched' && canUseSearchCoordinates.value ? searchedLocation.value : deviceCoords.value
    if (sourceCoords && Array.isArray(sourceCoords) && sourceCoords.length === 2) {
      const [uLng, uLat] = sourceCoords
      url += `&userLng=${encodeURIComponent(uLng)}&userLat=${encodeURIComponent(uLat)}`
    }

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

    // ensure shops are stored and presented sorted by distance when available
    if (shops && shops.length) {
      const shopsSorted = shops.slice().sort((a, b) => {
        const da = a.distance != null ? Number(a.distance) : Infinity
        const db = b.distance != null ? Number(b.distance) : Infinity
        return da - db
      })
      latestShops.value = shopsSorted
      visibleCount.value = Math.min(5, shopsSorted.length)
      renderShopMarkers(shopsSorted)
    } else {
      latestShops.value = []
      visibleCount.value = 0
      renderShopMarkers([])
    }

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
  shopMarkers.forEach((entry) => {
    if (!entry) return
    const m = entry.marker || entry
    if (m && typeof m.remove === 'function') m.remove()
  })
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

    const distanceText = shop.distance != null ? `${Number(shop.distance).toFixed(2)} km` : ''
    const htmlString = `
      <div class="shop-popup">
        <h3>${shop.name || ''}</h3>
        <p>${shop.description || ''}</p>
        <div class="shop-distance">${distanceText}</div>
        <a href="${shop.link || '#'}" target="_blank">查看地图</a>
      </div>
    `
    const popup = new mapboxgl.Popup({ offset: 12, closeButton: true, closeOnClick: true })
      .setHTML(htmlString)

    marker.setPopup(popup)
    pinEl.addEventListener('click', () => marker.togglePopup())

    shopMarkers.push({ marker, shop })
  })
}

// List scrolling: load more when near bottom
const onListScroll = (e) => {
  const el = e.target
  if (el.scrollTop + el.clientHeight >= el.scrollHeight - 20) {
    visibleCount.value = Math.min(latestShops.value.length, visibleCount.value + 5)
  }
}

// Click handler for loading more items
const loadMore = () => {
  visibleCount.value = Math.min(latestShops.value.length, visibleCount.value + 5)
}

// Open a shop from the right-hand list: fly to and open its popup
const openShopFromList = (index) => {
  const shop = visibleShops.value[index]
  if (!shop || !map.value) return
  const lat = Number(shop.latitude ?? shop.lat)
  const lng = Number(shop.longitude ?? shop.lng)
  if (isFinite(lat) && isFinite(lng)) {
    map.value.flyTo({ center: [lng, lat], zoom: 14, speed: 1.2, curve: 1.4, essential: true })
  }
  const entry = shopMarkers.find((e) => e.shop && Number(e.shop.latitude) === Number(shop.latitude) && Number(e.shop.longitude) === Number(shop.longitude) && e.shop.name === shop.name)
  if (entry && entry.marker) entry.marker.togglePopup()
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
      searchedLocation.value = next.location
      searchedPlaceName.value = next.name || ''
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
  updateIsMobile()
  const mq = window.matchMedia('(max-width: 640px)')
  try {
    mq.addEventListener('change', updateIsMobile)
  } catch (e) {
    mq.addListener(updateIsMobile)
  }
  // 2. 尝试获取浏览器当前用户的真实定位
  if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(
      (position) => {
        const { longitude, latitude } = position.coords
        deviceCoords.value = [longitude, latitude]
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
  map.value.addControl(new mapboxgl.NavigationControl(), 'bottom-left')

  map.value.on('load', () => {
    formatCenterText(map.value.getCenter())
    if (deviceCoordinates) {
      deviceCoords.value = deviceCoordinates
      formatDeviceText(deviceCoordinates)
      addDeviceMarker(deviceCoordinates)
    }
    if (pendingFlyTarget && pendingFlyTarget.location?.length === 2) {
      flyToLocation(pendingFlyTarget)
      pendingFlyTarget = null
    }
    // Trigger an initial search once the map has finished loading
    executeMoveLogic()
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
  try {
    const mq = window.matchMedia('(max-width: 640px)')
    mq.removeEventListener('change', updateIsMobile)
  } catch (e) {
    try { mq.removeListener(updateIsMobile) } catch (er) { /* ignore */ }
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

/* Right-hand shop list styles */
.shop-list {
  position: absolute;
  top: var(--map-overlay-top);
  right: 20px;
  width: 420px;
  max-height: calc(100vh - 7rem);
  background: rgba(255,255,255,0.95);
  border-radius: 12px;
  box-shadow: 0 18px 30px rgba(0,0,0,0.08);
  overflow: auto;
  z-index: 16;
  padding: 10px;
}
.shop-list-header {
  font-weight: 700;
  margin-bottom: 8px;
}
.shop-list-items { list-style: none; margin: 0; padding: 0; }
.shop-list-item { display: flex; align-items: center; justify-content: space-between; padding: 8px; border-bottom: 1px solid rgba(0,0,0,0.04); cursor: pointer; }
.shop-list-item:hover { background: rgba(0,0,0,0.02); }
.shop-name { font-size: 13px; font-weight: 700; }
.shop-desc { font-size: 12px; color: #6b7280; margin-top: 4px; }
.shop-item-left { max-width: 70%; }
.shop-item-right { font-weight: 700; color: var(--accent); }
.shop-list-loading { padding: 8px; text-align: center; color: #666; font-size: 13px; }

/* Inline title with small pill buttons */
.shop-header-title.inline { display: flex; align-items: center; gap: 10px; justify-content: center; font-size: 16px; font-weight: 700; }
.inline-btn { appearance: none; -webkit-appearance: none; border: none; padding: 6px 12px; border-radius: 999px; background: rgba(0,0,0,0.08); color: #111827; font-weight: 700; cursor: pointer; }
.inline-btn.active { background: var(--accent); color: white; box-shadow: 0 6px 18px rgba(17,24,39,0.08); }
.inline-btn.disabled { opacity: 0.45; cursor: not-allowed; }
.sep { color: rgba(0,0,0,0.35); margin: 0 6px; }
.shop-header-title .highlight { color: var(--accent); }

.load-more-btn {
  background: transparent;
  border: 1px solid rgba(0,0,0,0.06);
  padding: 8px 14px;
  border-radius: 8px;
  font-weight: 700;
  color: var(--accent);
  cursor: pointer;
}
.load-more-btn:hover { background: rgba(0,0,0,0.03); }

/* Mobile switch buttons */
.mobile-switch { position: fixed; left: 50%; transform: translateX(-50%); bottom: 18px; z-index: 60; display: none; gap: 8px; }
.mobile-switch button { padding: 8px 14px; border-radius: 999px; border: 1px solid rgba(0,0,0,0.06); background: white; font-weight: 700; }
.mobile-switch button.active { background: var(--accent); color: white; }

@media (max-width: 640px) {
  .mobile-switch { display: flex; }
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