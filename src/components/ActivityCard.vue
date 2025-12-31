<script setup>
import { computed } from 'vue'

const props = defineProps({
  activity: {
    type: Object,
    required: true
  },
  isEditing: {
    type: Boolean,
    default: false
  },
  noTime: {
    type: Boolean,
    default: false
  }
})

const localTags = computed({
  get: () => (props.activity.tags || []).join(', '),
  set: (val) => {
    props.activity.tags = val.split(',').map(t => t.trim()).filter(t => t)
  }
})

const getTagClass = (tag) => {
  const map = {
    '交通': 'transport',
    '美食': 'food',
    '觀光': 'sightseeing',
    '逛街': 'shopping',
    '滑雪': 'ski',
    '住宿': 'accommodation',
    '自由': 'free',
    '其他': 'other'
  }
  return map[tag] || ''
}

defineEmits(['remove', 'edit', 'save', 'toggle-visited'])
</script>

<template>
  <div class="activity-card" :class="{ 'is-editing': isEditing, 'no-time': noTime, 'visited': activity.visited }">
    <div v-if="isEditing" class="drag-handle" title="Drag to reorder">
      ⋮⋮
    </div>

    <button v-if="isEditing" class="remove-activity-btn" @click="$emit('remove')" title="Remove Activity">
      ✕
    </button>

    <div v-if="!noTime" class="time-container">
      <div v-if="!isEditing" class="time-badge">{{ activity.time }}</div>
      <input v-else type="text" v-model="activity.time" class="edit-input time-input" placeholder="HH:mm">
    </div>
    
    <div class="card-content">
      <div v-if="isEditing" class="edit-form">
        <div class="input-row">
          <input type="text" v-model="activity.title" class="edit-input title-input" placeholder="Activity Title">
        </div>

        
        <div class="input-row">
          <span class="edit-label">Tags (comma separated)</span>
          <input type="text" v-model="localTags" class="edit-input tags-input" placeholder="e.g. 交通, 美食">
        </div>

        <div class="input-row">
          <span class="edit-label">Location Name</span>
          <input type="text" v-model="activity.location" class="edit-input" placeholder="Location">
        </div>

        <div class="input-row">
          <span class="edit-label">Map URL</span>
          <input type="text" v-model="activity.mapLink" class="edit-input" placeholder="https://google.com/maps/...">
        </div>

        <div class="input-row">
          <span class="edit-label">Description</span>
          <textarea v-model="activity.description" class="edit-input desc-input" placeholder="Details..."></textarea>
        </div>

        <div class="edit-actions-bottom">
          <button class="save-card-btn" @click="$emit('save')">
            Done
          </button>
        </div>
      </div>

      <template v-else>
        <div class="header-row">
          <div class="title-group">
            <button 
              v-if="noTime"
              class="visited-toggle-btn" 
              :class="{ 'is-visited': activity.visited }" 
              @click.stop="$emit('toggle-visited')" 
              title="Mark as visited"
            >
              {{ activity.visited ? '✅' : '✔️' }}
            </button>
            <h3 class="activity-title">{{ activity.title }}</h3>
            <button class="inline-edit-btn" @click="$emit('edit')" title="Edit Item">
              ✏️
            </button>
          </div>
          <div v-if="activity.tags && activity.tags.length" class="tags-container">
            <span 
              v-for="tag in activity.tags" 
              :key="tag" 
              class="tag"
              :class="getTagClass(tag)"
            >
              {{ tag }}
            </span>
          </div>
        </div>
        <div class="location-row" v-if="activity.location">
          <span class="icon">📍</span>
          <a :href="activity.mapLink" target="_blank" rel="noopener noreferrer" class="location-link">
            {{ activity.location }}
          </a>
        </div>
        <div v-if="activity.links && activity.links.length" class="links-container">
          <a 
            v-for="link in activity.links" 
            :key="link.url" 
            :href="link.url"
            target="_blank"
            class="external-link"
          >
            <span class="icon">🔗</span>
            {{ link.text }}
          </a>
        </div>
        <p class="description">{{ activity.description }}</p>
      </template>
    </div>
  </div>
</template>

<style scoped>
.activity-card {
  position: relative;
  background: #ffffff;
  border-radius: 20px;
  padding: 1.5rem;
  margin-bottom: 1.5rem;
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
  border: 1px solid #e2e8f0;
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  overflow: hidden;
  display: flex;
  gap: 1rem;
}

.activity-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 12px 40px 0 rgba(31, 38, 135, 0.15);
}

.time-badge {
  background: #edf2f7;
  color: #4a5568;
  padding: 0.25rem 0.75rem;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.9rem;
  height: fit-content;
  white-space: nowrap;
  font-family: 'Monaco', 'Consolas', monospace; /* Monospace for alignment */
}

.card-content {
  flex: 1;
}

.header-row {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.activity-title {
  margin: 0;
  color: #2d3748;
  font-size: 1.25rem;
  font-weight: 700;
  letter-spacing: -0.025em;
}

.location-row {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  margin-bottom: 0.75rem;
}

.icon {
  font-size: 1rem;
}

.location-link {
  color: #5a67d8;
  text-decoration: none;
  font-weight: 500;
  font-size: 0.95rem;
  transition: color 0.2s;
}

.location-link:hover {
  color: #434190;
  text-decoration: underline;
}

.description {
  margin: 0;
  color: #4a5568;
  line-height: 1.6;
  font-size: 1rem;
}

@media (max-width: 480px) {
  .activity-card {
    flex-direction: column;
    padding: 1.25rem;
  }
  
  .time-badge {
    align-self: flex-start;
    margin-bottom: 0.5rem;
  }
}


.tags-container {
  display: flex;
  gap: 0.4rem;
  flex-wrap: wrap;
}

.tag {
  font-size: 0.75rem;
  padding: 0.2rem 0.6rem;
  border-radius: 9999px;
  font-weight: 600;
  background: #edf2f7;
  color: #4a5568;
}

/* Tag Specific Colors */
.tag.transport {
  background: #e6fffa;
  color: #2c7a7b; /* Teal */
}

.tag.food {
  background: #fff5f5;
  color: #c53030; /* Red */
}

.tag.sightseeing {
  background: #ebf8ff;
  color: #2b6cb0; /* Blue */
}

.tag.shopping {
  background: #faf5ff;
  color: #6b46c1; /* Purple */
}

.tag.ski {
  background: #f0fff4;
  color: #2f855a; /* Green */
}

.tag.accommodation {
  background: #fffff0;
  color: #b7791f; /* Yellow/Orange */
}

.tag.free {
  background: #f7fafc;
  color: #718096; /* Gray/Cool Gray for neutral/flexible */
  border: 1px dashed #cbd5e0; /* Dashed border to indicate flexibility */
}

.links-container {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 0.75rem;
  flex-wrap: wrap;
}

.external-link {
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
  color: #805ad5; /* Purple */
  background: #faf5ff;
  padding: 0.3rem 0.8rem;
  border-radius: 6px;
  text-decoration: none;
  font-size: 0.9rem;
  font-weight: 500;
  border: 1px solid #e9d8fd;
  transition: all 0.2s;
}

.external-link:hover {
  background: #f3e8ff;
  border-color: #d6bcfa;
}

/* Editing Styles */
.edit-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.input-row {
  display: flex;
  flex-direction: column;
  gap: 0.2rem;
}

.edit-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #718096;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.edit-input {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.95rem;
  color: #2d3748;
  transition: border-color 0.2s;
}

.edit-input:focus {
  outline: none;
  border-color: #4299e1;
  box-shadow: 0 0 0 3px rgba(66, 153, 225, 0.1);
}

.time-input {
  width: 90px;
  font-family: 'Monaco', 'Consolas', monospace;
  font-weight: 700;
  text-align: center;
  padding: 0.4rem;
  background: #f7fafc;
  border-color: #cbd5e0;
}

.title-input {
  font-size: 1.25rem;
  font-weight: 700;
}

.desc-input {
  min-height: 80px;
  resize: vertical;
  line-height: 1.5;
}

.activity-card.is-editing {
  border: 2px solid #4299e1;
  box-shadow: 0 10px 25px rgba(66, 153, 225, 0.1);
  padding-left: 3rem; /* Make room for drag handle */
}

.drag-handle {
  position: absolute;
  left: 0.75rem;
  top: 50%;
  transform: translateY(-50%);
  color: #cbd5e0;
  cursor: grab;
  font-size: 1.5rem;
  padding: 0.5rem;
  transition: color 0.2s;
}

.drag-handle:hover {
  color: #4299e1;
}

.drag-handle:active {
  cursor: grabbing;
}

.remove-activity-btn {
  position: absolute;
  top: 0.75rem;
  right: 0.75rem;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  border: none;
  background: #fff5f5;
  color: #e53e3e;
  font-size: 0.75rem;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s;
  opacity: 0;
  z-index: 5;
}

.activity-card:hover .remove-activity-btn {
  opacity: 1;
}

.remove-activity-btn:hover {
  background: #e53e3e;
  color: white;
  transform: scale(1.1);
}


.activity-card.no-time {
  padding-left: 1.5rem;
}

.activity-card.no-time.is-editing {
  padding-left: 2.5rem;
}

.title-group {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.inline-edit-btn {
  background: transparent;
  border: none;
  cursor: pointer;
  font-size: 0.9rem;
  opacity: 0;
  transition: opacity 0.2s, transform 0.2s;
  padding: 4px;
  border-radius: 4px;
}

.activity-card:hover .inline-edit-btn {
  opacity: 0.6;
}

.inline-edit-btn:hover {
  opacity: 1 !important;
  background: #edf2f7;
  transform: scale(1.1);
}

.edit-actions-bottom {
  display: flex;
  justify-content: flex-end;
  margin-top: 0.5rem;
}

.save-card-btn {
  padding: 0.5rem 1.5rem;
  background: #4299e1;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: 700;
  font-size: 0.85rem;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 4px 6px rgba(66, 153, 225, 0.2);
}

.save-card-btn:hover {
  background: #3182ce;
  transform: translateY(-1px);
  box-shadow: 0 6px 12px rgba(66, 153, 225, 0.3);
}

.visited-toggle-btn {
  background: transparent;
  border: none;
  cursor: pointer;
  font-size: 1.1rem;
  padding: 0;
  margin-right: 0.2rem;
  transition: transform 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 50%;
}

.visited-toggle-btn:not(.is-visited) {
  opacity: 0.3;
}

.visited-toggle-btn:hover {
  transform: scale(1.2);
  opacity: 1 !important;
}

.visited-toggle-btn.is-visited {
  opacity: 1;
}

.activity-card.visited {
  border-color: #cbd5e0;
  background: #fcfcfc;
}

.activity-card.visited .activity-title,
.activity-card.visited .description,
.activity-card.visited .location-row {
  opacity: 0.6;
}
</style>
