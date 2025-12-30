<script setup>
defineProps({
  activity: {
    type: Object,
    required: true
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
</script>

<template>
  <div class="activity-card">
    <div class="time-badge">{{ activity.time }}</div>
    
    <div class="card-content">
      <div class="header-row">
        <h3 class="activity-title">{{ activity.title }}</h3>
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
</style>
