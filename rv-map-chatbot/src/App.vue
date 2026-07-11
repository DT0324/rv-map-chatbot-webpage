<script setup>
import { ref } from 'vue'
import MapView from './components/MapView.vue'
import Topbar from './components/Topbar.vue'

const targetLocation = ref(null)

const handleSelectLocation = (location) => {
  console.log('App received select-location:', location)
  targetLocation.value = location
}
</script>

<template>
  <Topbar @select-location="handleSelectLocation" />
  <MapView :flyToTarget="targetLocation" />
</template>

<style>
/* 重置默认边距，让地图完美铺满全屏 */
html, body, #app {
  margin: 0;
  padding: 0;
  width: 100vw;
  height: 100vh;
  overflow: hidden; /* 防止出现多余的滚动条 */
}
</style>
