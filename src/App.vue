<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import DayView from './components/DayView.vue'

// Initial state is null since we removed defaults
const selectedItinerary = ref(null)
const itinerary = ref([])
const currentDayIndex = ref(0)
const loading = ref(true)
const isSidebarOpen = ref(true)

// Custom itineraries state
const customItineraries = ref([]) // Array of { name: string, url: null }
const customDataMap = ref({}) // Map of filename -> itinerary data

const allItineraries = computed(() => {
  return customItineraries.value
})

const processItineraryData = async (data) => {
  itinerary.value = data
  // Default to first day
  currentDayIndex.value = 0

  // Jump to Today Logic
  const today = new Date()
  const todayStr = today.toISOString().split('T')[0] // YYYY-MM-DD
  
  const todayIndex = itinerary.value.findIndex(day => day.date === todayStr)
  
  if (todayIndex !== -1) {
    currentDayIndex.value = todayIndex
    
    // Wait for DOM to update with new day
    await nextTick()
    
    // Find nearest activity to current time
    const currentHours = today.getHours()
    const currentMinutes = today.getMinutes()
    const currentTimeInMinutes = currentHours * 60 + currentMinutes
    
    const activities = itinerary.value[todayIndex].activities
    let closestActivityIndex = -1
    let minDiff = Infinity
    
    activities.forEach((activity, index) => {
      const [h, m] = activity.time.split(':').map(Number)
      const activityTimeInMinutes = h * 60 + m
      const diff = Math.abs(currentTimeInMinutes - activityTimeInMinutes)
      
      if (diff < minDiff) {
        minDiff = diff
        closestActivityIndex = index
      }
    })
    
    if (closestActivityIndex !== -1) {
      setTimeout(() => {
          const cards = document.querySelectorAll('.activity-card')
          if (cards && cards[closestActivityIndex]) {
              cards[closestActivityIndex].scrollIntoView({ behavior: 'smooth', block: 'center' })
          }
      }, 100)
    }
  }
}

const fetchItinerary = async (url) => {
  loading.value = true
  try {
    const response = await fetch(url)
    const data = await response.json()
    await processItineraryData(data)
  } catch (error) {
    console.error('Failed to load itinerary:', error)
  } finally {
    loading.value = false
  }
}

const toggleSidebar = () => {
  isSidebarOpen.value = !isSidebarOpen.value
}

onMounted(async () => {
  // Load custom itineraries list and data map
  const savedCustomList = localStorage.getItem('customItinerariesList')
  const savedCustomData = localStorage.getItem('customDataMap')
  
  if (savedCustomList) customItineraries.value = JSON.parse(savedCustomList)
  if (savedCustomData) customDataMap.value = JSON.parse(savedCustomData)

  const savedTrip = localStorage.getItem('lastSelectedTrip')
  if (savedTrip) {
    const trip = JSON.parse(savedTrip)
    selectedItinerary.value = trip
    
    if (trip.url) {
      // url-based trips are removed, we don't fetch anymore
    } else {
      // Load custom data from map
      const data = customDataMap.value[trip.name]
      if (data) {
        await processItineraryData(data)
        loading.value = false
      }
    }
  } else if (allItineraries.value.length > 0) {
    // If no last trip but we have custom ones, pick the first
    selectedItinerary.value = allItineraries.value[0]
  } else {
    loading.value = false
  }

  // Auto-close on small screens
  if (window.innerWidth < 768) {
    isSidebarOpen.value = false
  }
})

watch(selectedItinerary, (newVal) => {
  if (!newVal) return
  localStorage.setItem('lastSelectedTrip', JSON.stringify(newVal))
  
  if (newVal.url) {
    fetchItinerary(newVal.url)
  } else {
    // Selection changed to a custom trip
    const data = customDataMap.value[newVal.name]
    if (data) {
      processItineraryData(data)
    }
  }
  
  if (window.innerWidth < 768) {
    isSidebarOpen.value = false
  }
})

const currentDay = computed(() => {
  if (itinerary.value.length === 0) return null
  return itinerary.value[currentDayIndex.value]
})

const nextDay = () => {
  if (currentDayIndex.value < itinerary.value.length - 1) {
    currentDayIndex.value++
  }
}

const prevDay = () => {
  if (currentDayIndex.value > 0) {
    currentDayIndex.value--
  }
}

const deleteTrip = (trip) => {
  if (!confirm(`Are you sure you want to delete "${trip.name}"?`)) return

  // 1. Remove from data map
  delete customDataMap.value[trip.name]
  localStorage.setItem('customDataMap', JSON.stringify(customDataMap.value))

  // 2. Remove from list
  customItineraries.value = customItineraries.value.filter(t => t.name !== trip.name)
  localStorage.setItem('customItinerariesList', JSON.stringify(customItineraries.value))

  // 3. Update selection if we deleted the active one
  if (selectedItinerary.value?.name === trip.name) {
    if (customItineraries.value.length > 0) {
      selectedItinerary.value = customItineraries.value[0]
    } else {
      selectedItinerary.value = null
      itinerary.value = []
    }
  }
}

const downloadTrip = () => {
  if (!itinerary.value || itinerary.value.length === 0) return
  
  try {
    const data = JSON.stringify(itinerary.value, null, 2)
    const blob = new Blob([data], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = url
    const baseName = (selectedItinerary.value?.name || 'trip').replace(/\.json$/i, '')
    link.download = `${baseName}.json`
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
    URL.revokeObjectURL(url)
  } catch (error) {
    console.error('Download failed:', error)
    alert('Failed to download itinerary.')
  }
}

const fileInput = ref(null)

const triggerFileUpload = () => {
  fileInput.value.click()
}

const handleFileUpload = (event) => {
  const file = event.target.files[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = async (e) => {
    try {
      const json = JSON.parse(e.target.result)
      const fileName = file.name

      // 1. Update Data Map
      customDataMap.value[fileName] = json
      localStorage.setItem('customDataMap', JSON.stringify(customDataMap.value))

      // 2. Update List if not already there
      if (!customItineraries.value.find(t => t.name === fileName)) {
        customItineraries.value.push({ name: fileName, url: null })
        localStorage.setItem('customItinerariesList', JSON.stringify(customItineraries.value))
      }

      // 3. Process and Select
      await processItineraryData(json)
      selectedItinerary.value = { name: fileName, url: null }
      
      if (window.innerWidth < 768) {
        isSidebarOpen.value = false
      }
    } catch (error) {
      console.error('Error parsing JSON:', error)
      alert('Invalid JSON file')
    }
  }
  reader.readAsText(file)
  event.target.value = ''
}
</script>

<template>
  <div class="layout">
    <button 
      class="sidebar-toggle" 
      @click="toggleSidebar"
      :aria-label="isSidebarOpen ? 'Close Sidebar' : 'Open Sidebar'"
    >
      <span v-if="isSidebarOpen">←</span>
      <span v-else>☰</span>
    </button>

    <aside class="sidebar" :class="{ collapsed: !isSidebarOpen }">
      <div class="sidebar-content">
        <h2 class="sidebar-title">My Trips</h2>
        <ul class="trip-list">
          <li 
            v-for="trip in allItineraries" 
            :key="trip.url || trip.name"
            class="trip-item"
            :class="{ active: selectedItinerary?.name === trip.name }"
            @click="selectedItinerary = trip"
          >
            {{ trip.name.replace(/\.json$/i, '') }}
          </li>
        </ul>
        
        <div v-if="allItineraries.length === 0" class="empty-sidebar-hint">
          No trips yet. Upload your first itinerary to get started!
        </div>
        
        <div class="upload-section">
          <input 
            type="file" 
            ref="fileInput" 
            accept=".json" 
            style="display: none" 
            @change="handleFileUpload"
          >
          <button class="upload-btn" @click="triggerFileUpload">
            📂 Upload JSON
          </button>
        </div>
      </div>
    </aside>

    <main class="main-content">
      <div v-if="loading" class="loading">
        <div class="spinner"></div>
        <p>Loading your journey...</p>
      </div>

      <div v-else-if="currentDay" class="content-wrapper">
        <div class="top-actions">
          <button class="action-btn download" title="Download JSON" @click="downloadTrip">
            � Export
          </button>
          <button class="action-btn delete" title="Delete Trip" @click="deleteTrip(selectedItinerary)">
            🗑️ Delete
          </button>
        </div>
        <DayView :day="currentDay" />
        
        <div class="navigation-controls">
          <button 
            class="nav-btn prev" 
            @click="prevDay" 
            :disabled="currentDayIndex === 0"
            :class="{ hidden: currentDayIndex === 0 }"
          >
            ← Prev Day
          </button>
          
          <div class="dots">
            <span 
              v-for="(_, index) in itinerary" 
              :key="index"
              class="dot"
              :class="{ active: index === currentDayIndex }"
              @click="currentDayIndex = index"
            ></span>
          </div>

          <button 
            class="nav-btn next" 
            @click="nextDay" 
            :disabled="currentDayIndex === itinerary.length - 1"
            :class="{ hidden: currentDayIndex === itinerary.length - 1 }"
          >
            Next Day →
          </button>
        </div>
      </div>

      <div v-else class="error-state">
        <div class="welcome-box">
          <h1>Welcome to Travel Plan</h1>
          <p>Get started by uploading your trip JSON file.</p>
          <button class="upload-btn primary" @click="triggerFileUpload">
            📂 Upload JSON
          </button>
        </div>
      </div>
    </main>
  </div>
</template>

<style scoped>
.layout {
  display: flex;
  height: 100vh; /* Fixed height to prevent body scroll */
  overflow: hidden; /* Hide overflow on the layout container */
  position: relative;
}

.sidebar-toggle {
  position: fixed;
  top: 1rem;
  left: 1rem;
  z-index: 100;
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.2rem;
  color: #4a5568;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  transition: all 0.3s ease;
}

.sidebar-toggle:hover {
  transform: scale(1.1);
  background: #f7fafc;
}

.sidebar {
  width: 250px;
  background: #f7fafc;
  border-right: 1px solid #e2e8f0;
  flex-shrink: 0;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  overflow: hidden;
  position: relative;
  height: 100%; /* Full height */
}

.sidebar.collapsed {
  width: 0;
  border-right: none;
}

.sidebar-content {
  padding: 4rem 1.5rem 2rem;
  width: 250px;
}

.sidebar-title {
  font-size: 1.2rem;
  font-weight: 700;
  color: #2d3748;
  margin-bottom: 1.5rem;
  white-space: nowrap;
}

.trip-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.trip-item {
  padding: 0.75rem 1rem;
  border-radius: 8px;
  color: #4a5568;
  cursor: pointer;
  transition: all 0.2s;
  margin-bottom: 0.5rem;
  font-weight: 500;
  white-space: nowrap;
}

.trip-item:hover {
  background: #edf2f7;
  color: #2d3748;
}

.trip-item.active {
  background: #4299e1;
  color: white;
  box-shadow: 0 4px 6px rgba(66, 153, 225, 0.3);
}

.upload-section {
  margin-top: 2rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.upload-btn {
  width: 100%;
  padding: 0.75rem 1rem;
  background: white;
  border: 1px dashed #cbd5e0;
  border-radius: 8px;
  color: #4a5568;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  font-weight: 500;
}

.upload-btn:hover {
  background: #edf2f7;
  border-color: #4299e1;
  color: #2d3748;
}

.upload-btn.primary {
  background: #4299e1;
  color: white;
  border: none;
  padding: 1rem 2rem;
  font-size: 1.1rem;
  margin-top: 1.5rem;
}

.upload-btn.primary:hover {
  background: #3182ce;
  transform: translateY(-2px);
  box-shadow: 0 4px 6px rgba(66, 153, 225, 0.4);
}

.content-wrapper {
  width: 100%;
  max-width: 600px;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding-bottom: 4rem; /* Add space for bottom/scrolling */
  position: relative; /* Added for absolute positioning of children */
}

.top-actions {
  position: absolute;
  top: 0;
  right: -100px; /* Move it slightly outside the 600px container on wide screens */
  display: flex;
  flex-direction: column; /* Stacked look often feels more premium in top-right corners */
  gap: 0.5rem;
  z-index: 10;
}

/* Adjust for smaller screens where buttons might overlap content */
@media (max-width: 900px) {
  .top-actions {
    position: static;
    flex-direction: row;
    width: 100%;
    justify-content: flex-end;
    margin-bottom: 1rem;
    padding: 0 1rem;
  }
}

.action-btn {
  padding: 0.5rem 1rem;
  border-radius: 8px;
  font-size: 0.85rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  gap: 0.4rem;
  border: 1px solid #e2e8f0;
  background: white;
  color: #4a5568;
}

.action-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 6px rgba(0,0,0,0.05);
}

.action-btn.delete:hover {
  background: #fff5f5;
  color: #e53e3e;
  border-color: #feb2b2;
}

.action-btn.download:hover {
  background: #f0fff4;
  color: #38a169;
  border-color: #9ae6b4;
}

.empty-sidebar-hint {
  font-size: 0.85rem;
  color: #718096;
  font-style: italic;
  padding: 1rem 0.5rem;
  line-height: 1.4;
}

.welcome-box {
  background: white;
  padding: 3rem;
  border-radius: 20px;
  box-shadow: 0 10px 25px rgba(0,0,0,0.05);
  text-align: center;
  max-width: 450px;
}

.welcome-box h1 {
  color: #2d3748;
  margin-bottom: 1rem;
}

.welcome-box p {
  color: #718096;
  margin-bottom: 0.5rem;
}

.main-content {
  flex: 1;
  padding: 2rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  overflow-y: auto; /* Allow scrolling within the main content */
  height: 100%; /* Full height */
  transition: all 0.3s ease;
}

.content-wrapper {
  width: 100%;
  max-width: 600px;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding-bottom: 4rem; /* Add space for bottom/scrolling */
}

.loading, .error-state {
  color: #4a5568;
  text-align: center;
  margin-top: 20vh;
  font-size: 1.2rem;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #e2e8f0;
  border-top-color: #4299e1;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin: 0 auto 1rem;
}

.navigation-controls {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  margin-top: 2rem;
  padding: 1rem;
  background: #f7fafc;
  border-radius: 100px;
}

.nav-btn {
  background: white;
  border: 1px solid #e2e8f0;
  padding: 0.75rem 1.5rem;
  border-radius: 50px;
  font-weight: 700;
  color: #4a5568;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 0.9rem;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}

.nav-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  background: white;
  border-color: #cbd5e0;
}

.nav-btn:disabled, .nav-btn.hidden {
  opacity: 0;
  pointer-events: none;
}

.dots {
  display: flex;
  gap: 0.5rem;
}

.dot {
  width: 10px;
  height: 10px;
  background: #cbd5e0;
  border-radius: 50%;
  cursor: pointer;
  transition: all 0.3s;
}

.dot.active {
  background: #4299e1;
  transform: scale(1.2);
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

@media (max-width: 768px) {
  .layout {
    flex-direction: column;
  }
  
  .sidebar {
    width: 100%;
    padding: 1rem;
    overflow-x: auto;
    border-right: none;
    border-bottom: 1px solid #e2e8f0;
  }

  .trip-list {
    display: flex;
    gap: 0.5rem;
  }

  .trip-item {
    margin-bottom: 0;
    white-space: nowrap;
  }

  .main-content {
    padding: 1rem;
  }
}
</style>
