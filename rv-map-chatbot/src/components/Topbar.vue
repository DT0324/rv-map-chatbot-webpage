<template>
  <div class="topbar">
    <div class="search-box">
      <span class="search-icon">🔍</span>
      <input
        v-model="searchQuery"
        @input="onSearchInput"
        @focus="openSuggestions"
        @keydown.enter.prevent="applyActiveSuggestion"
        @keydown.arrow-down.prevent="moveActive(1)"
        @keydown.arrow-up.prevent="moveActive(-1)"
        type="text"
        placeholder="Search address"
        aria-label="Search address"
      />
    </div>

    <button class="filter-button" @click="toggleFilter">
      Filter
    </button>

    <ul v-if="isOpen && suggestions.length" class="suggestions" role="listbox">
      <li
        v-for="(item, index) in suggestions"
        :key="item.id"
        :class="{ active: index === activeIndex }"
        @click="selectSuggestion(item)"
      >
        {{ item.place_name }}
      </li>
    </ul>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const MAPBOX_TOKEN = 'pk.eyJ1IjoiZHQwMzI0IiwiYSI6ImNtbnlvNW0zbDAzeTAycnB2eXg0MXNldmoifQ.6gAAqjLYlQUXD-TXmSjcLg'
const searchQuery = ref('')
const suggestions = ref([])
const isOpen = ref(false)
const activeIndex = ref(-1)
let searchTimeout = null

const emit = defineEmits(['select-location', 'toggle-filter'])

const toggleFilter = () => {
  emit('toggle-filter')
}

const openSuggestions = () => {
  if (suggestions.value.length) {
    isOpen.value = true
  }
}

const onSearchInput = () => {
  if (searchTimeout) {
    clearTimeout(searchTimeout)
  }

  if (!searchQuery.value.trim()) {
    suggestions.value = []
    isOpen.value = false
    activeIndex.value = -1
    return
  }

  searchTimeout = setTimeout(fetchPlaces, 250)
}

const fetchPlaces = async () => {
  const query = searchQuery.value.trim()
  if (!query) {
    suggestions.value = []
    isOpen.value = false
    return
  }

  const url = `https://api.mapbox.com/geocoding/v5/mapbox.places/${encodeURIComponent(query)}.json?access_token=${MAPBOX_TOKEN}&autocomplete=true&limit=5&country=au`

  try {
    const response = await fetch(url)
    const data = await response.json()
    suggestions.value = data.features || []
    isOpen.value = suggestions.value.length > 0
    activeIndex.value = suggestions.value.length ? 0 : -1
  } catch (err) {
    console.warn('Geocoding error', err)
    suggestions.value = []
    isOpen.value = false
  }
}

const selectSuggestion = (item) => {
  console.log('Selected geocoding item:', item)
  searchQuery.value = item.place_name
  suggestions.value = []
  isOpen.value = false
  activeIndex.value = -1

  const rawLocation = item.center || item.geometry?.coordinates
  if (!rawLocation || rawLocation.length !== 2) {
    console.warn('Selected item does not contain valid coordinates:', item)
    return
  }

  const location = [rawLocation[0], rawLocation[1]]
  console.log('Emitting select-location with coordinates:', location)

  emit('select-location', {
    name: item.place_name,
    location,
    zoom: 14,
  })
}

const applyActiveSuggestion = () => {
  if (activeIndex.value >= 0 && activeIndex.value < suggestions.value.length) {
    selectSuggestion(suggestions.value[activeIndex.value])
  }
}

const moveActive = (direction) => {
  if (!suggestions.value.length) return
  activeIndex.value = (activeIndex.value + direction + suggestions.value.length) % suggestions.value.length
}
</script>

<style scoped>
.topbar {
  position: absolute;
  top: 16px;
  left: 16px;
  right: 16px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 12px 16px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 18px;
  box-shadow: 0 24px 60px rgba(15, 23, 42, 0.12);
  backdrop-filter: blur(12px);
  z-index: 20;
}

.search-box {
  position: relative;
  display: flex;
  align-items: center;
  gap: 10px;
  flex: 1;
  min-width: 220px;
  padding: 10px 14px;
  border-radius: 14px;
  background: #f5f7fb;
  border: 1px solid #e5e7eb;
}

.search-icon {
  font-size: 16px;
  color: #64748b;
}

.search-box input {
  width: 100%;
  border: none;
  background: transparent;
  outline: none;
  font-size: 14px;
  color: #111827;
}

.filter-button {
  border: none;
  border-radius: 14px;
  padding: 10px 18px;
  font-size: 13px;
  font-weight: 600;
  color: #ffffff;
  background: #111827;
  cursor: pointer;
  transition: opacity 0.2s ease, transform 0.2s ease;
}

.filter-button:hover {
  opacity: 0.92;
  transform: translateY(-1px);
}

.suggestions {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  margin: 10px 0 0;
  padding: 8px 0;
  background: rgba(255, 255, 255, 0.98);
  border-radius: 16px;
  box-shadow: 0 18px 40px rgba(15, 23, 42, 0.12);
  list-style: none;
  max-height: 260px;
  overflow-y: auto;
  z-index: 30;
}

.suggestions li {
  padding: 10px 16px;
  font-size: 14px;
  color: #111827;
  cursor: pointer;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.suggestions li:hover,
.suggestions li.active {
  background: #f8fafc;
}
</style>
