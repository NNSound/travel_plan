<script setup>
import ActivityCard from './ActivityCard.vue'
import { computed } from 'vue'
import draggable from 'vuedraggable'

const props = defineProps({
  day: {
    type: Object,
    required: true
  }
})

const isDayHeaderEditing = ref(false)

const formattedDateWithDay = computed(() => {
  if (!props.day.date) return ''
  const date = new Date(props.day.date)
  const weekday = date.toLocaleDateString('en-US', { weekday: 'long' })
  return `${props.day.date} · ${weekday}`
})

const addActivity = () => {
  props.day.activities.push({
    id: Date.now(),
    time: '09:00',
    title: 'New Activity',
    description: '',
    location: '',
    mapLink: '',
    tags: [],
    isEditing: true
  })
}

const removeActivity = (index) => {
  if (confirm('Delete this activity?')) {
    props.day.activities.splice(index, 1)
  }
}

import { ref } from 'vue'
</script>

<template>
  <div class="day-view">
    <header class="day-header" :data-agent="'day-header-' + day.day">
      <div class="day-badge">Day {{ day.day }}</div>
      
      <div v-if="isDayHeaderEditing" class="edit-fields">
        <div class="input-group">
          <label>Date</label>
          <input type="date" v-model="day.date" class="edit-input" data-agent="day-date-input">
        </div>
        <div class="input-group">
          <label>Day Title</label>
          <input type="text" v-model="day.title" class="edit-input title-input" data-agent="day-title-input">
        </div>
        <div class="header-actions">
          <button class="save-btn" @click="isDayHeaderEditing = false" data-agent="save-day-header">Done</button>
        </div>
      </div>
      <template v-else>
        <div class="header-content" @click="isDayHeaderEditing = true" title="Click to edit date/title" data-agent="edit-day-header">
          <h2 class="date">{{ formattedDateWithDay }}</h2>
          <h1 class="day-title">{{ day.title }}</h1>
          <span class="edit-hint">✏️</span>
        </div>
      </template>
    </header>

    <div class="activities-list">
      <draggable 
        v-model="day.activities" 
        item-key="id"
        handle=".drag-handle"
        ghost-class="ghost-card"
        class="drag-container"
      >
        <template #item="{ element, index }">
          <ActivityCard 
            :activity="element" 
            :isEditing="!!element.isEditing"
            @remove="removeActivity(index)"
            @edit="element.isEditing = true"
            @save="element.isEditing = false"
          />
        </template>
      </draggable>

      <button class="add-activity-btn" @click="addActivity" data-agent="add-activity">
        ➕ Add New Activity
      </button>
    </div>
  </div>
</template>

<style scoped>
.day-view {
  width: 100%;
  max-width: 600px;
  margin: 0 auto;
  animation: fadeIn 0.5s ease-out;
}

.day-header {
  text-align: center;
  margin-bottom: 2rem;
  color: #2d3748;
}

.day-badge {
  display: inline-block;
  background: #e2e8f0;
  color: #4a5568;
  padding: 0.25rem 1rem;
  border-radius: 20px;
  font-size: 0.9rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  margin-bottom: 0.5rem;
  border: none;
}

.date {
  font-size: 1.1rem;
  opacity: 0.8;
  font-weight: 500;
  margin: 0.5rem 0;
  color: #718096;
}

.day-title {
  font-size: 2.5rem;
  font-weight: 800;
  margin: 0.5rem 0;
  letter-spacing: -0.03em;
  background: linear-gradient(to right, #2d3748, #4a5568);
  background-clip: text;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.edit-fields {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  max-width: 400px;
  margin: 1rem auto;
  text-align: left;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.input-group label {
  font-size: 0.8rem;
  font-weight: 600;
  color: #718096;
}

.edit-input {
  padding: 0.5rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 1rem;
  width: 100%;
}

.title-input {
  font-size: 1.5rem;
  font-weight: 700;
}

.activities-list {
  padding: 0 1rem;
}

.add-activity-btn {
  width: 100%;
  padding: 1rem;
  margin-top: 1rem;
  background: white;
  border: 2px dashed #cbd5e0;
  border-radius: 20px;
  color: #718096;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
}

.add-activity-btn:hover {
  background: #f7fafc;
  border-color: #4299e1;
  color: #4299e1;
  transform: translateY(-2px);
}

.ghost-card {
  opacity: 0.5;
  background: #edf2f7;
  border: 2px dashed #4299e1;
}

.drag-container {
  display: flex;
  flex-direction: column;
  gap: 0;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}

.header-content {
  cursor: pointer;
  position: relative;
  padding: 0.5rem;
  border-radius: 12px;
  transition: background 0.2s;
}

.header-content:hover {
  background: #f7fafc;
}

.edit-hint {
  position: absolute;
  top: 50%;
  right: -2rem;
  transform: translateY(-50%);
  opacity: 0;
  transition: opacity 0.2s;
}

.header-content:hover .edit-hint {
  opacity: 0.6;
}

.header-actions {
  display: flex;
  justify-content: center;
  margin-top: 1rem;
}

.save-btn {
  padding: 0.5rem 2rem;
  background: #4299e1;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: 700;
  cursor: pointer;
}

.save-btn:hover {
  background: #3182ce;
}
</style>
