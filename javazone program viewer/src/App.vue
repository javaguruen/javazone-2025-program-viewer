<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'

interface Speaker {
  name: string
}

interface Session {
  sessionId: string
  title: string
  format: string
  length: string
  room: string
  startTimeZulu: string
  speakers: Speaker[]
}

const sessionsData = ref<Session[]>([])
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
    sessionsData.value = data.sessions || data || []
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'An error occurred'
    console.error('Error fetching sessions:', err)
  } finally {
    loading.value = false
  }
}

// Filter out workshops and organize by room
const filteredAndGroupedSessions = computed(() => {
  if (!sessionsData.value || !Array.isArray(sessionsData.value)) {
    return { rooms: [], timeSlots: [] }
  }

  // Filter out workshops
  const nonWorkshopSessions = sessionsData.value.filter(
    (session: Session) => session.format !== 'workshop'
  )

  // Get unique rooms
  const rooms = [...new Set(nonWorkshopSessions.map((session: Session) => session.room))]
    .filter(room => room && room.trim() !== '')
    .sort()

  // Get unique time slots and sort them
  const timeSlots = [...new Set(nonWorkshopSessions.map((session: Session) => session.startTimeZulu))]
    .filter(time => time)
    .sort()

  // Create a map for easy lookup: timeSlot -> room -> session
  const sessionMap = new Map()
  
  nonWorkshopSessions.forEach((session: Session) => {
    if (!session.startTimeZulu || !session.room) return
    
    if (!sessionMap.has(session.startTimeZulu)) {
      sessionMap.set(session.startTimeZulu, new Map())
    }
    sessionMap.get(session.startTimeZulu).set(session.room, session)
  })

  return { rooms, timeSlots, sessionMap }
})

// Helper function to format time
const formatTime = (timeString: string) => {
  if (!timeString) return ''
  try {
    const date = new Date(timeString)
    return date.toLocaleTimeString('en-US', { 
      hour: '2-digit', 
      minute: '2-digit',
      hour12: false
    })
  } catch {
    return timeString
  }
}

// Helper function to get speaker names
const getSpeakerNames = (speakers: Speaker[]) => {
  if (!speakers || !Array.isArray(speakers)) return ''
  return speakers.map(speaker => speaker.name).join(', ')
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
    
    <div v-else class="sessions-table-container">
      <h2>Program Schedule (Workshops excluded)</h2>
      
      <div class="table-wrapper">
        <table class="sessions-table">
          <thead>
            <tr>
              <th class="time-header">Time</th>
              <th v-for="room in filteredAndGroupedSessions.rooms" :key="room" class="room-header">
                {{ room }}
              </th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="timeSlot in filteredAndGroupedSessions.timeSlots" :key="timeSlot" class="time-row">
              <td class="time-cell">
                {{ formatTime(timeSlot) }}
              </td>
              <td v-for="room in filteredAndGroupedSessions.rooms" :key="room" class="session-cell">
                <div v-if="filteredAndGroupedSessions.sessionMap.get(timeSlot)?.get(room)" class="session-content">
                  <div class="session-title">
                    {{ filteredAndGroupedSessions.sessionMap.get(timeSlot).get(room).title }}
                  </div>
                  <div class="session-duration">
                    {{ filteredAndGroupedSessions.sessionMap.get(timeSlot).get(room).length }} min
                  </div>
                  <div class="session-speakers">
                    {{ getSpeakerNames(filteredAndGroupedSessions.sessionMap.get(timeSlot).get(room).speakers) }}
                  </div>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
      
      <div class="stats">
        <p>Total sessions displayed: {{ filteredAndGroupedSessions.timeSlots.length * filteredAndGroupedSessions.rooms.length - 
           (filteredAndGroupedSessions.timeSlots.reduce((count, timeSlot) => 
             count + filteredAndGroupedSessions.rooms.filter(room => 
               !filteredAndGroupedSessions.sessionMap.get(timeSlot)?.get(room)
             ).length, 0)) }}</p>
      </div>
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

.sessions-table-container {
  margin-top: 20px;
}

.table-wrapper {
  overflow-x: auto;
  margin-bottom: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.sessions-table {
  width: 100%;
  border-collapse: collapse;
  background-color: white;
  font-size: 14px;
}

.time-header {
  background-color: #2c3e50;
  color: white;
  padding: 12px 8px;
  text-align: center;
  font-weight: bold;
  min-width: 80px;
  position: sticky;
  left: 0;
  z-index: 10;
}

.room-header {
  background-color: #34495e;
  color: white;
  padding: 12px 8px;
  text-align: center;
  font-weight: bold;
  min-width: 200px;
  max-width: 250px;
}

.time-cell {
  background-color: #ecf0f1;
  padding: 12px 8px;
  text-align: center;
  font-weight: bold;
  border-right: 2px solid #bdc3c7;
  min-width: 80px;
  position: sticky;
  left: 0;
  z-index: 5;
}

.session-cell {
  padding: 8px;
  border: 1px solid #dee2e6;
  vertical-align: top;
  min-width: 200px;
  max-width: 250px;
  height: 120px;
}

.session-content {
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.session-title {
  font-weight: bold;
  margin-bottom: 8px;
  line-height: 1.3;
  color: #2c3e50;
  font-size: 13px;
  overflow: hidden;
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
}

.session-duration {
  background-color: #3498db;
  color: white;
  padding: 2px 6px;
  border-radius: 12px;
  font-size: 11px;
  font-weight: bold;
  text-align: center;
  margin-bottom: 6px;
  align-self: flex-start;
}

.session-speakers {
  font-size: 12px;
  color: #7f8c8d;
  font-style: italic;
  line-height: 1.2;
  overflow: hidden;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
}

.time-row:nth-child(even) .time-cell {
  background-color: #f8f9fa;
}

.time-row:nth-child(even) .session-cell {
  background-color: #fefefe;
}

.time-row:hover .time-cell {
  background-color: #d5dbdb;
}

.time-row:hover .session-cell {
  background-color: #f1f2f6;
}

.stats {
  margin-top: 20px;
  padding: 15px;
  background-color: #ecf0f1;
  border-radius: 8px;
  text-align: center;
  color: #2c3e50;
  font-weight: bold;
}

@media (max-width: 768px) {
  .sessions-table {
    font-size: 12px;
  }
  
  .time-header, .room-header {
    padding: 8px 4px;
  }
  
  .time-cell, .session-cell {
    padding: 6px 4px;
  }
  
  .session-title {
    font-size: 12px;
  }
  
  .session-speakers {
    font-size: 11px;
  }
}
</style>
