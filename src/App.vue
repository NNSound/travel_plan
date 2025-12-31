<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import DayView from './components/DayView.vue'
import ActivityCard from './components/ActivityCard.vue'

// Initial state is null since we removed defaults
const selectedItinerary = ref(null)
const itinerary = ref([])
const currentDayIndex = ref(0)
const loading = ref(true)
const isSidebarOpen = ref(true)
const pocketList = ref([])
const isPocketModalOpen = ref(false)
const isDocModalOpen = ref(false)
const docTab = ref('preview') // 'preview' or 'raw'
const pocketFilter = ref('All') // 'All', 'Visited', or specific tag

// Custom itineraries state
const customItineraries = ref([]) // Array of { name: string, url: null }
const customDataMap = ref({}) // Map of filename -> itinerary data

const allItineraries = computed(() => {
  return customItineraries.value
})

const pocketTags = computed(() => {
  const tags = new Set()
  pocketList.value.forEach(item => {
    if (item.tags) item.tags.forEach(t => tags.add(t))
  })
  return ['All', 'Visited', ...Array.from(tags)]
})

const filteredPocketList = computed(() => {
  let list = [...pocketList.value]
  
  // Filter
  if (pocketFilter.value !== 'All') {
    if (pocketFilter.value === 'Visited') {
      list = list.filter(item => item.visited)
    } else {
      list = list.filter(item => item.tags && item.tags.includes(pocketFilter.value))
    }
  }

  // Sort: Visited items at the bottom
  return list.sort((a, b) => {
    if (a.visited === b.visited) return 0
    return a.visited ? 1 : -1
  })
})

const processItineraryData = async (data) => {
  if (Array.isArray(data)) {
    itinerary.value = data
    pocketList.value = []
  } else {
    itinerary.value = data.days || []
    pocketList.value = data.pocketList || []
  }
  
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
  // Remove global pocket list sync as it's now trip-specific
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

watch(selectedItinerary, () => {
  // Reset states if needed
})

watch(itinerary, () => {
  if (selectedItinerary.value?.name) {
    const tripData = {
      days: itinerary.value,
      pocketList: pocketList.value
    }
    customDataMap.value[selectedItinerary.value.name] = tripData
    localStorage.setItem('customDataMap', JSON.stringify(customDataMap.value))
  }
}, { deep: true })

watch(isPocketModalOpen, (newVal) => {
  if (!newVal) {
    // Reset all items to non-editing mode when modal closes
    pocketList.value.forEach(item => {
      item.isEditing = false
    })
  }
})

watch(pocketList, () => {
  savePocketList()
}, { deep: true })

const savePocketList = () => {
  // Update in memory first
  const currentTripName = selectedItinerary.value?.name
  if (currentTripName) {
    const tripData = {
      days: itinerary.value,
      pocketList: pocketList.value
    }
    customDataMap.value[currentTripName] = tripData
    localStorage.setItem('customDataMap', JSON.stringify(customDataMap.value))
  } else {
    // Fallback to a temp global storage if no trip selected
    localStorage.setItem('pocketList_temp', JSON.stringify(pocketList.value))
  }
}

const addToPocket = () => {
  pocketList.value.unshift({
    id: Date.now(), // Unique ID for stable rendering
    title: 'New Destination',
    description: '',
    location: '',
    mapLink: '',
    tags: ['想去'],
    isEditing: true // New items start in editing mode
  })
  savePocketList()
}

const removeFromPocket = (index) => {
  if (confirm('Remove from pocket list?')) {
    pocketList.value.splice(index, 1)
    savePocketList()
  }
}

const addItemToCurrentDay = (item) => {
  if (!currentDay.value) {
    alert('Please select a trip first!')
    return
  }
  // Deep clone the item (excluding time as it doesn't have it)
  const newItem = JSON.parse(JSON.stringify(item))
  newItem.time = '09:00' // Default time for new items added from pocket
  
  currentDay.value.activities.push(newItem)
  isPocketModalOpen.value = false
  alert(`"${newItem.title}" added to ${currentDay.value.date}! Remember to save your trip changes.`)
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
    const tripData = {
      days: itinerary.value,
      pocketList: pocketList.value
    }
    const data = JSON.stringify(tripData, null, 2)
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

const copyToClipboard = async (text, message = 'Copied!') => {
  try {
    await navigator.clipboard.writeText(text);
    alert(message);
  } catch (err) {
    console.error('Failed to copy: ', err);
    alert('Failed to copy to clipboard');
  }
}

const templateJson = `{
  "name": "Taiwan Trip Example",
  "days": [
    {
      "day": 1,
      "date": "2024-12-31",
      "title": "New Year Arrival",
      "activities": [
        {
          "time": "14:00",
          "title": "Arrive at TAIPEI 101",
          "location": "Taipei 101",
          "description": "Check-in and view city lights.",
          "mapLink": "https://www.google.com/maps/search/?api=1&query=Taipei+101",
          "tags": ["Sightseeing", "Shopping"]
        }
      ]
    }
  ],
  "pocketList": [
    {
      "title": "Din Tai Fung",
      "location": "Xinyi District",
      "description": "Famous xiaolongbao.",
      "mapLink": "https://www.google.com/maps/search/?api=1&query=Din+Tai+Fung+Xinyi",
      "tags": ["Food"],
      "visited": false
    }
  ]
}`

const rawAgentMd = `# Travel Plan JSON Specification

This guide explains how to generate or create JSON files that are compatible with the Travel Planner application. These files can be imported via the "Import JSON" button in the sidebar.

## JSON Structure Overview

The root of the JSON file must contain the following fields:

| Field | Type | Description |
| :--- | :--- | :--- |
| \`name\` | String | The overall name of the trip. |
| \`days\` | Array | A list of daily itineraries. |
| \`pocketList\` | Array | (Optional) A list of inspirations/spots not yet assigned to a specific day. |

---

## Component Details

### 1. Day Object
Each item in the \`days\` array should have:
- \`day\` (Number): The sequence number of the day.
- \`date\` (String): The date in \`YYYY-MM-DD\` format.
- \`title\` (String): A brief theme for the day.
- \`activities\` (Array): A list of activity objects.

### 2. Activity Object
Used in both \`days.activities\` and \`pocketList\`.
- \`time\` (String): 24-hour format \`HH:mm\`.
- \`title\` (String): Name of the activity or spot.
- \`location\` (String): Human-readable address or landmark name.
- \`description\` (String): Details or notes.
- \`mapLink\` (String): Google Maps search URL (see below).
- \`tags\` (Array): Category labels (e.g., \`["Food", "Sightseeing"]\`).
- \`visited\` (Boolean): (Optional, used in Pocket List) Set to \`true\` if already visited.

---

## Guidelines for Quality Content

### 📍 Google Maps Links
For better reliability and readability, **ALWAYS** use the Google Maps Search API format instead of raw coordinates:
- **Format**: \`https://www.google.com/maps/search/?api=1&query=Landmark+Name\`
- **Example**: \`https://www.google.com/maps/search/?api=1&query=Taipei+101+Observatory\`

### 🏷️ Recommended Tags
The UI provides special styling for the following tags:
- \`Transport\`
- \`Food\`
- \`Sightseeing\`
- \`Shopping\`
- \`Ski\`
- \`Accommodation\`
- \`Free\`

---

## Example Template

\`\`\`json
{
  "name": "Taiwan Trip Example",
  "days": [
    {
      "day": 1,
      "date": "2024-12-31",
      "title": "New Year Arrival",
      "activities": [
        {
          "time": "14:00",
          "title": "Arrive at TAIPEI 101",
          "location": "Taipei 101",
          "description": "Check-in and view city lights.",
          "mapLink": "https://www.google.com/maps/search/?api=1&query=Taipei+101",
          "tags": ["Sightseeing", "Shopping"]
        }
      ]
    }
  ],
  "pocketList": [
    {
      "title": "Din Tai Fung",
      "location": "Xinyi District",
      "description": "Famous xiaolongbao.",
      "mapLink": "https://www.google.com/maps/search/?api=1&query=Din+Tai+Fung+Xinyi",
      "tags": ["Food"],
      "visited": false
    }
  ]
}
\`\`\`
`
</script>

<template>
  <div class="layout">
    <div 
      v-if="isSidebarOpen" 
      class="mobile-backdrop" 
      @click="toggleSidebar"
    ></div>

    <!-- Pocket List Modal (Revamped) -->
    <div v-if="isPocketModalOpen" class="modal-overlay" @click.self="isPocketModalOpen = false">
      <div class="modal-content pocket-modal">
        <header class="modal-header">
          <div class="header-text">
            <h2>👜 Pocket List</h2>
            <p class="pocket-subtitle">Your travel inspirations bucket</p>
          </div>
          <button class="close-btn" @click="isPocketModalOpen = false">✕</button>
        </header>

        <div class="filter-section">
          <div class="filter-chips">
            <button 
              v-for="tag in pocketTags" 
              :key="tag"
              class="filter-chip"
              :class="{ active: pocketFilter === tag }"
              @click="pocketFilter = tag"
              :data-agent="'pocket-filter-' + tag"
            >
              {{ tag }}
            </button>
          </div>
        </div>

        <div class="modal-body">
          <div v-if="pocketList.length === 0" class="empty-pocket">
            <div class="empty-icon">📍</div>
            <h3>Your Pocket is Empty</h3>
            <p>Save interesting spots here to plan your journey easier.</p>
          </div>
          
          <div class="pocket-list">
            <ActivityCard 
              v-for="(item, index) in filteredPocketList" 
              :key="item.id || index" 
              :activity="item" 
              :isEditing="!!item.isEditing" 
              noTime
              @remove="removeFromPocket(index)"
              @add-to-trip="addItemToCurrentDay(item)"
              @edit="item.isEditing = true"
              @save="item.isEditing = false"
              @toggle-visited="item.visited = !item.visited"
            />
          </div>
        </div>

        <footer class="modal-footer">
          <button class="add-pocket-btn primary" @click="addToPocket" data-agent="add-pocket-item">
            ➕ Add New Destination
          </button>
        </footer>
      </div>
    </div>

    <button 
      class="sidebar-toggle" 
      @click="toggleSidebar"
      :aria-label="isSidebarOpen ? 'Close Sidebar' : 'Open Sidebar'"
      data-agent="sidebar-toggle"
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
            :data-agent="'trip-item-' + trip.name"
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
          <button class="upload-btn" @click="triggerFileUpload" data-agent="upload-json">
            📂 Upload JSON
          </button>
        </div>
        <div class="sidebar-footer">
          <button class="doc-link-btn" @click="isDocModalOpen = true" data-agent="json-guide-toggle">
            📖 JSON Guide
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
          <button class="action-btn download" title="Download JSON" @click="downloadTrip" data-agent="export-trip">
            💾 Export
          </button>
          <button class="action-btn delete" title="Delete Trip" @click="deleteTrip(selectedItinerary)" data-agent="delete-trip">
            🗑️ Delete
          </button>
          <button class="action-btn pocket" title="Pocket List" @click="isPocketModalOpen = true" data-agent="pocket-list-toggle">
            👜 Pocket List
          </button>
        </div>
        <DayView :day="currentDay" />
        
        <div class="navigation-controls">
          <button 
            class="nav-btn prev" 
            @click="prevDay" 
            :disabled="currentDayIndex === 0"
            :class="{ hidden: currentDayIndex === 0 }"
            data-agent="prev-day"
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
            data-agent="next-day"
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

    <div v-if="isDocModalOpen" class="modal-overlay" @click.self="isDocModalOpen = false">
      <div class="modal-content doc-modal">
        <header class="modal-header doc-modal-header">
          <div class="header-text">
            <h2>📖 JSON Guide</h2>
            <p class="doc-subtitle">Use these specifications to generate compatible travel plans with AI assistants.</p>
            <div class="doc-tabs">
              <button 
                class="doc-tab-btn" 
                :class="{ active: docTab === 'preview' }"
                @click="docTab = 'preview'"
              >Preview</button>
              <button 
                class="doc-tab-btn" 
                :class="{ active: docTab === 'raw' }"
                @click="docTab = 'raw'"
              >Raw Markdown</button>
            </div>
          </div>
          <button class="close-btn" @click="isDocModalOpen = false">✕</button>
        </header>
        
        <div v-if="docTab === 'preview'" class="modal-body doc-body">
          <div class="doc-section">
            <h1>Travel Plan JSON Specification</h1>
            <p>This guide explains how to generate or create JSON files that are compatible with the Travel Planner application. These files can be imported via the <strong>"Import JSON"</strong> button in the sidebar.</p>
          </div>

          <div class="doc-section">
            <h3>JSON Structure Overview</h3>
            <p>The root of the JSON file must contain the following fields:</p>
            <table class="spec-table">
              <thead>
                <tr>
                  <th>Field</th>
                  <th>Type</th>
                  <th>Description</th>
                </tr>
              </thead>
              <tbody>
                <tr>
                  <td><code>name</code></td>
                  <td>String</td>
                  <td>The overall name of the trip.</td>
                </tr>
                <tr>
                  <td><code>days</code></td>
                  <td>Array</td>
                  <td>A list of daily itineraries.</td>
                </tr>
                <tr>
                  <td><code>pocketList</code></td>
                  <td>Array</td>
                  <td>(Optional) inspirations/spots.</td>
                </tr>
              </tbody>
            </table>
          </div>

          <div class="doc-section">
            <h3>Component Details</h3>
            <h4>1. Day Object</h4>
            <ul>
              <li><code>day</code>: Sequence number (Number)</li>
              <li><code>date</code>: <code>YYYY-MM-DD</code> format</li>
              <li><code>title</code>: Day theme</li>
              <li><code>activities</code>: List of activity objects</li>
            </ul>
            <h4>2. Activity Object</h4>
            <ul>
              <li><code>time</code>: 24h format <code>HH:mm</code></li>
              <li><code>title</code>: Name of activity</li>
              <li><code>location</code>: Landmark name</li>
              <li><code>description</code>: Details/Notes</li>
              <li><code>mapLink</code>: Google Maps URL</li>
              <li><code>tags</code>: e.g., <code>["Food", "Sightseeing"]</code></li>
            </ul>
          </div>

          <div class="doc-section">
            <h3>Guidelines for Quality Content</h3>
            <div class="tip-box">
              <strong>📍 Google Maps Links</strong>
              <p>ALWAYS use the Search API format for reliability:</p>
              <code>https://www.google.com/maps/search/?api=1&query=Landmark+Name</code>
            </div>
            <div class="tip-box" style="margin-top: 1rem;">
              <strong>🏷️ Recommended Tags</strong>
              <p>Styled tags: Transport, Food, Sightseeing, Shopping, Ski, Accommodation, Free</p>
            </div>
          </div>
          
          <div class="doc-section">
            <h3>Full Example Template</h3>
            <p>Copy this structure to your AI assistant:</p>
            <div class="code-block">
              <button class="copy-hint" @click="copyToClipboard(templateJson, 'Template copied!')">Copy Template</button>
              <pre>{{ templateJson }}</pre>
            </div>
          </div>
          <!-- CLEANED SECTION -->
        </div>

        <div v-else class="modal-body doc-body raw-body">
          <div class="raw-container">
            <button class="copy-raw-btn" @click="copyToClipboard(rawAgentMd, 'Markdown copied!')">Copy Raw MD</button>
            <pre class="raw-markdown">{{ rawAgentMd }}</pre>
          </div>
        </div>
      </div>
    </div>
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

.action-btn.edit:hover {
  background: #ebf8ff;
  color: #2b6cb0;
  border-color: #90cdf4;
}

.action-btn.save {
  background: #4299e1;
  color: white;
  border: none;
}

.action-btn.save:hover {
  background: #3182ce;
}

.action-btn.cancel:hover {
  background: #fff5f5;
  color: #e53e3e;
  border-color: #feb2b2;
}

.action-btn.pocket:hover {
  background: #fff5f5;
  color: #c53030;
  border-color: #feb2b2;
}

/* Modal Styles - Premium Revamp */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(45, 55, 72, 0.4);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  backdrop-filter: blur(8px);
  padding: 1.5rem;
}

.modal-content {
  background: white;
  width: 100%;
  max-height: 90vh;
  border-radius: 32px;
  box-shadow: 0 40px 100px -20px rgba(0, 0, 0, 0.25);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  border: 1px solid rgba(255, 255, 255, 0.5);
  animation: modalPop 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
}

.modal-content.pocket-modal {
  max-width: 650px;
}

.modal-content.doc-modal {
  max-width: 800px;
}

.modal-header {
  padding: 2rem 2.5rem;
  border-bottom: 1px solid #edf2f7;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  background: white;
  position: sticky;
  top: 0;
  z-index: 10;
}

.header-text h2 {
  margin: 0;
  font-size: 1.75rem;
  font-weight: 800;
  color: #2d3748; /* Simplified color for reliability */
}

.pocket-subtitle {
  margin: 0.25rem 0 0;
  color: #718096;
  font-size: 0.95rem;
}

.close-btn {
  background: #f7fafc;
  border: none;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  font-size: 1.2rem;
  color: #a0aec0;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}

.close-btn:hover {
  background: #edf2f7;
  color: #e53e3e;
  transform: rotate(90deg);
}

.modal-body {
  padding: 1.5rem 2.5rem;
  overflow-y: auto;
  flex: 1;
  background: #f8fafc;
}

.pocket-list {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  padding-bottom: 2rem;
  max-width: 600px;
  margin: 0 auto;
}

.empty-pocket {
  text-align: center;
  padding: 5rem 2rem;
  color: #a0aec0;
}

.empty-icon {
  font-size: 4rem;
  margin-bottom: 1.5rem;
  opacity: 0.5;
}

.empty-pocket h3 {
  color: #4a5568;
  margin-bottom: 0.5rem;
}

.modal-footer {
  padding: 1.5rem 2.5rem;
  border-top: 1px solid #edf2f7;
  background: white;
  display: flex;
  justify-content: center;
}

.add-pocket-btn.primary {
  width: 100%;
  max-width: 400px;
  padding: 1rem 2rem;
  background: #4299e1;
  color: white;
  border: none;
  border-radius: 16px;
  font-weight: 700;
  font-size: 1.1rem;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 10px 15px -3px rgba(66, 153, 225, 0.3);
}

.add-pocket-btn.primary:hover {
  background: #3182ce;
  transform: translateY(-2px);
  box-shadow: 0 15px 25px -5px rgba(66, 153, 225, 0.4);
}

@media (max-width: 768px) {
  .pocket-grid {
    grid-template-columns: 1fr;
  }
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

.filter-section {
  padding: 0.5rem 2rem;
  background: white;
  border-bottom: 1px solid #edf2f7;
}

.filter-chips {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
  padding: 0.5rem 0;
}

.filter-chip {
  padding: 0.4rem 1rem;
  border-radius: 100px;
  background: #edf2f7;
  color: #4a5568;
  border: 1px solid transparent;
  font-size: 0.85rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.filter-chip:hover {
  background: #e2e8f0;
}

.filter-chip.active {
  background: #4299e1;
  color: white;
  box-shadow: 0 4px 6px rgba(66, 153, 225, 0.3);
}

@media (max-width: 768px) {
  .layout {
    flex-direction: row;
  }
  
  .sidebar {
    position: fixed;
    top: 0;
    left: 0;
    height: 100%;
    width: 280px;
    z-index: 1000;
    background: white;
    box-shadow: 10px 0 30px rgba(0,0,0,0.1);
    transform: translateX(0);
    transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }

  .sidebar.collapsed {
    transform: translateX(-100%);
    border: none;
  }

  .sidebar-content {
    padding: 5rem 1.5rem 2rem;
    width: 100%;
  }

  .trip-list {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .trip-item {
    margin-bottom: 0.5rem;
    padding: 0.75rem 1rem;
    font-size: 1rem;
    white-space: normal;
  }

  .main-content {
    position: relative;
    width: 100%;
    padding: 1rem;
    padding-top: 5rem;
    flex: none; /* Occupy full screen independently */
  }

  .top-actions {
    position: static;
    flex-direction: row;
    width: 100%;
    justify-content: flex-end;
    margin-bottom: 1.5rem;
    padding: 0;
    gap: 0.5rem;
  }

  .mobile-backdrop {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background: rgba(0,0,0,0.4);
    z-index: 999;
    backdrop-filter: blur(2px);
    transition: opacity 0.3s ease;
  }

  .welcome-box {
    padding: 2rem 1.5rem;
    margin: 1rem;
    width: auto;
  }
}
.sidebar-footer {
  margin-top: auto;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.doc-link-btn {
  width: 100%;
  padding: 0.75rem;
  background: #f7fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  color: #4a5568;
  font-weight: 600;
  font-size: 0.9rem;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
}

.doc-link-btn:hover {
  background: white;
  border-color: #4299e1;
  color: #4299e1;
  box-shadow: 0 2px 4px rgba(66, 153, 225, 0.1);
}

.doc-modal {
  max-height: 85vh !important;
}

.doc-body {
  padding: 2.5rem;
  overflow-y: auto;
  background: #ffffff;
}

.doc-section {
  margin-bottom: 2rem;
}

.doc-section h3 {
  font-size: 1.1rem;
  color: #2d3748;
  margin-bottom: 0.75rem;
  border-left: 4px solid #4299e1;
  padding-left: 0.75rem;
}

.doc-section p, .doc-section li {
  color: #4a5568;
  line-height: 1.6;
}

.doc-modal-header {
  flex-direction: row;
  align-items: center;
  padding: 1.5rem 2.5rem;
}

.doc-subtitle {
  margin: 0.25rem 0 0.5rem;
  color: #718096;
  font-size: 0.9rem;
}

.doc-tabs {
  display: flex;
  gap: 1rem;
  margin-top: 0.5rem;
}

.doc-tab-btn {
  background: transparent;
  border: none;
  font-size: 0.9rem;
  font-weight: 600;
  color: #a0aec0;
  cursor: pointer;
  padding: 0.25rem 0;
  border-bottom: 2px solid transparent;
  transition: all 0.2s;
}

.doc-tab-btn:hover {
  color: #718096;
}

.doc-tab-btn.active {
  color: #4299e1;
  border-bottom-color: #4299e1;
}

.raw-body {
  background: #1a202c !important;
  color: #e2e8f0;
  padding: 0 !important;
}

.raw-container {
  position: relative;
  height: 100%;
}

.copy-raw-btn {
  position: absolute;
  top: 1rem;
  right: 1.5rem;
  background: rgba(255,255,255,0.1);
  border: 1px solid rgba(255,255,255,0.2);
  color: #a0aec0;
  padding: 0.4rem 0.8rem;
  border-radius: 6px;
  font-size: 0.8rem;
  cursor: pointer;
  z-index: 5;
}

.copy-raw-btn:hover {
  background: rgba(255,255,255,0.2);
  color: white;
}

.raw-markdown {
  margin: 0;
  padding: 2.5rem;
  font-family: 'Monaco', 'Consolas', monospace;
  font-size: 0.9rem;
  line-height: 1.5;
  white-space: pre-wrap;
  overflow-y: auto;
}

.doc-section h1 {
  font-size: 1.75rem;
  color: #2d3748;
  margin-bottom: 1.5rem;
  border-bottom: 2px solid #edf2f7;
  padding-bottom: 0.75rem;
}

.doc-section h4 {
  font-size: 1rem;
  color: #4a5568;
  margin: 1rem 0 0.5rem;
}

.spec-table {
  width: 100%;
  border-collapse: collapse;
  margin: 1rem 0;
  font-size: 0.9rem;
}

.spec-table th, .spec-table td {
  text-align: left;
  padding: 0.75rem;
  border-bottom: 1px solid #edf2f7;
}

.spec-table th {
  color: #718096;
  font-weight: 600;
}

.tip-box {
  background: #ebf8ff;
  border-left: 4px solid #4299e1;
  padding: 1rem;
  border-radius: 0 8px 8px 0;
}

.tip-box strong {
  display: block;
  color: #2b6cb0;
  margin-bottom: 0.25rem;
}

.tip-box p, .tip-box code {
  margin: 0;
  font-size: 0.85rem;
}

.code-block {
  position: relative;
  background: #1a202c;
  color: #e2e8f0;
  padding: 1.5rem;
  border-radius: 12px;
  margin: 1rem 0;
  font-size: 0.85rem;
}

.copy-hint {
  position: absolute;
  top: 0.75rem;
  right: 0.75rem;
  background: rgba(255,255,255,0.1);
  border: 1px solid rgba(255,255,255,0.2);
  color: #a0aec0;
  padding: 0.25rem 0.6rem;
  border-radius: 6px;
  font-size: 0.7rem;
  cursor: pointer;
  transition: all 0.2s;
}

.copy-hint:hover {
  background: rgba(255,255,255,0.2);
  color: white;
}

.code-block pre {
  margin: 0;
  white-space: pre-wrap;
  font-family: 'Monaco', 'Consolas', monospace;
}
</style>
