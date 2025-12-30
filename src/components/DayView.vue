<script setup>
import ActivityCard from './ActivityCard.vue'
import { computed } from 'vue'

const props = defineProps({
  day: {
    type: Object,
    required: true
  }
})

const formattedDateWithDay = computed(() => {
  if (!props.day.date) return ''
  const date = new Date(props.day.date)
  // Use 'en-US' for "Saturday" or 'zh-TW' for "週六" depending on preference.
  // Given the UI is English ("Day 1"), English weekday might fit better, 
  // BUT user asked for "Chinese display" in previous prompts for content.
  // "日期後方應自動計算星期幾" -> Let's use English to match "Day 1" header, 
  // or maybe use a format like "2025-01-25 (Sat)" which is neutral.
  const weekday = date.toLocaleDateString('en-US', { weekday: 'long' })
  return `${props.day.date} · ${weekday}`
})
</script>

<template>
  <div class="day-view">
    <header class="day-header">
      <div class="day-badge">Day {{ day.day }}</div>
      <h2 class="date">{{ formattedDateWithDay }}</h2>
      <h1 class="day-title">{{ day.title }}</h1>
    </header>

    <div class="activities-list">
      <ActivityCard 
        v-for="(activity, index) in day.activities" 
        :key="index" 
        :activity="activity" 
      />
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
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.activities-list {
  padding: 0 1rem;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}
</style>
