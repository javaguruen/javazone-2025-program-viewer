<script setup lang="ts">
import { ref, onMounted } from 'vue'

const sessionsData = ref<any>(null)
const loading = ref(true)
const error = ref<string | null>(null)

const fetchSessions = async () => {
  try {
    loading.value = true
    error.value = null
    
    const response = await fetch('https://sleepingpill.javazone.no/public/allSessions/javazone_2025')
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }
    
    const data = await response.json()
    sessionsData.value = data
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'An error occurred'
    console.error('Error fetching sessions:', err)
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  fetchSessions()
})
</script>

<template>
  <div class="container">
    <h1>JavaZone 2025 Sessions</h1>
    
    <div v-if="loading" class="loading">
      Loading sessions data...
    </div>
    
    <div v-else-if="error" class="error">
      Error: {{ error }}
      <button @click="fetchSessions" class="retry-btn">Retry</button>
    </div>
    
    <div v-else class="json-container">
      <h2>Raw JSON Data:</h2>
      <pre class="json-output">{{ JSON.stringify(sessionsData, null, 2) }}</pre>
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 100vw;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

h1 {
  color: #2c3e50;
  margin-bottom: 20px;
}

h2 {
  color: #34495e;
  margin-bottom: 15px;
}

.loading {
  padding: 20px;
  text-align: center;
  font-size: 18px;
  color: #7f8c8d;
}

.error {
  padding: 20px;
  background-color: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
  border-radius: 4px;
  margin-bottom: 20px;
}

.retry-btn {
  margin-left: 10px;
  padding: 5px 10px;
  background-color: #dc3545;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.retry-btn:hover {
  background-color: #c82333;
}

.json-container {
  margin-top: 20px;
}

.json-output {
  background-color: #f8f9fa;
  border: 1px solid #dee2e6;
  border-radius: 4px;
  padding: 15px;
  font-family: 'Courier New', Courier, monospace;
  font-size: 12px;
  line-height: 1.4;
  overflow-x: auto;
  white-space: pre-wrap;
  word-wrap: break-word;
  max-height: 80vh;
  overflow-y: auto;
}
</style>
