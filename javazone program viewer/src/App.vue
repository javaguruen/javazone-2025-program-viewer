<script setup lang="ts">
import { ref, onMounted, computed, nextTick } from 'vue'

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
  endTimeZulu: string
  speakers: Speaker[]
}

interface DayData {
  date: string
  displayDate: string
  rooms: string[]
  timeSlots: string[]
  sessionMap: Map<string, Map<string, Session>>
}

const sessionsData = ref<Session[]>([])
const loading = ref(true)
const error = ref<string | null>(null)
const activeTab = ref<string>('')
const favorites = ref<Set<string>>(new Set())
const showFavoritesOnly = ref(false)

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

// Favorites management
const loadFavoritesFromStorage = () => {
  try {
    const stored = localStorage.getItem('javazone-favorites')
    if (stored) {
      const favArray = JSON.parse(stored)
      favorites.value = new Set(favArray)
    }
  } catch (error) {
    console.error('Error loading favorites from localStorage:', error)
  }
}

const saveFavoritesToStorage = () => {
  try {
    const favArray = Array.from(favorites.value)
    localStorage.setItem('javazone-favorites', JSON.stringify(favArray))
  } catch (error) {
    console.error('Error saving favorites to localStorage:', error)
  }
}

const toggleFavorite = (sessionId: string) => {
  if (favorites.value.has(sessionId)) {
    favorites.value.delete(sessionId)
  } else {
    favorites.value.add(sessionId)
  }
  saveFavoritesToStorage()
}

const isFavorite = (sessionId: string) => {
  return favorites.value.has(sessionId)
}

const favoriteCount = computed(() => {
  return favorites.value.size
})

// Filter sessions based on favorites setting
const filterSessions = (sessions: Session[]) => {
  if (!showFavoritesOnly.value) {
    return sessions
  }
  return sessions.filter(session => favorites.value.has(session.sessionId))
}

// Group sessions by day
const sessionsByDay = computed(() => {
  if (!sessionsData.value || !Array.isArray(sessionsData.value)) {
    return new Map<string, DayData>()
  }

  // Filter out workshops
  const nonWorkshopSessions = sessionsData.value.filter(
    (session: Session) => session.format !== 'workshop'
  )

  // Group by date
  const dayMap = new Map<string, Session[]>()
  
  nonWorkshopSessions.forEach((session: Session) => {
    if (!session.startTimeZulu) return
    
    const date = new Date(session.startTimeZulu)
    const dateKey = date.toISOString().split('T')[0] // YYYY-MM-DD format
    
    if (!dayMap.has(dateKey)) {
      dayMap.set(dateKey, [])
    }
    dayMap.get(dateKey)!.push(session)
  })

  // Create DayData for each day
  const result = new Map<string, DayData>()
  
  for (const [dateKey, sessions] of dayMap.entries()) {
    // Get unique rooms for this day
    const rooms = [...new Set(sessions.map(session => session.room))]
      .filter(room => room && room.trim() !== '')
      .sort()

    // Get unique time slots for this day and sort them
    const timeSlots = [...new Set(sessions.map(session => session.startTimeZulu))]
      .filter(time => time)
      .sort()

    // Create session map for this day
    const sessionMap = new Map<string, Map<string, Session>>()
    
    sessions.forEach((session: Session) => {
      if (!session.startTimeZulu || !session.room) return
      
      if (!sessionMap.has(session.startTimeZulu)) {
        sessionMap.set(session.startTimeZulu, new Map())
      }
      sessionMap.get(session.startTimeZulu)!.set(session.room, session)
    })

    // Format display date
    const date = new Date(dateKey)
    const displayDate = date.toLocaleDateString('en-US', {
      weekday: 'long',
      year: 'numeric',
      month: 'long',
      day: 'numeric'
    })

    result.set(dateKey, {
      date: dateKey,
      displayDate,
      rooms,
      timeSlots,
      sessionMap
    })
  }

  return result
})

// Get current day sessions for auto-scrolling
const getCurrentDayCurrentSession = (dayData: DayData) => {
  const now = new Date()
  const currentTime = now.toISOString()
  
  // Find the session that's currently running or about to start
  for (const timeSlot of dayData.timeSlots) {
    const sessionStart = new Date(timeSlot)
    const sessionEnd = new Date(sessionStart.getTime() + 60 * 60 * 1000) // Assume 1 hour if no end time
    
    // Check if current time is within this time slot (with some buffer)
    if (currentTime >= timeSlot && currentTime <= sessionEnd.toISOString()) {
      return timeSlot
    }
    
    // If we haven't reached this time slot yet, it's the next one
    if (currentTime < timeSlot) {
      return timeSlot
    }
  }
  
  return null
}

// Initialize active tab and auto-scroll
const initializeView = async () => {
  if (sessionsByDay.value.size === 0) return
  
  const today = new Date().toISOString().split('T')[0]
  const availableDays = Array.from(sessionsByDay.value.keys()).sort()
  
  // Set active tab to today if available, otherwise first day
  if (sessionsByDay.value.has(today)) {
    activeTab.value = today
  } else {
    activeTab.value = availableDays[0] || ''
  }
  
  // Auto-scroll to current session if we're on a conference day
  await nextTick()
  
  if (activeTab.value === today && sessionsByDay.value.has(today)) {
    const dayData = sessionsByDay.value.get(today)!
    const currentSession = getCurrentDayCurrentSession(dayData)
    
    if (currentSession) {
      // Scroll to current session after a short delay
      setTimeout(() => {
        const element = document.querySelector(`[data-time-slot="${currentSession}"]`)
        if (element) {
          element.scrollIntoView({ behavior: 'smooth', block: 'center' })
        }
      }, 500)
    }
  }
}

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

// Check if a time slot is currently active
const isCurrentTimeSlot = (timeSlot: string) => {
  const now = new Date()
  const sessionStart = new Date(timeSlot)
  const sessionEnd = new Date(sessionStart.getTime() + 60 * 60 * 1000) // Assume 1 hour duration
  
  return now >= sessionStart && now <= sessionEnd
}

onMounted(async () => {
  loadFavoritesFromStorage()
  await fetchSessions()
  await initializeView()
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
    
    <div v-else class="sessions-container">
      <div class="header-controls">
        <h2>Program Schedule (Workshops excluded)</h2>
        
        <!-- Filter Controls -->
        <div class="filter-controls">
          <div class="favorites-info">
            <span class="favorites-count">❤️ {{ favoriteCount }} favorites</span>
          </div>
          <div class="filter-buttons">
            <button 
              :class="['filter-btn', { active: !showFavoritesOnly }]" 
              @click="showFavoritesOnly = false"
            >
              All Sessions
            </button>
            <button 
              :class="['filter-btn', { active: showFavoritesOnly }]" 
              @click="showFavoritesOnly = true"
              :disabled="favoriteCount === 0"
            >
              Favorites Only
            </button>
          </div>
        </div>
      </div>
      
      <!-- Day Tabs -->
      <div class="tabs-container">
        <div class="tabs">
          <button 
            v-for="[dateKey, dayData] in sessionsByDay.entries()" 
            :key="dateKey"
            :class="['tab', { active: activeTab === dateKey }]"
            @click="activeTab = dateKey"
          >
            {{ dayData.displayDate }}
          </button>
        </div>
      </div>
      
      <!-- Day Content -->
      <div v-for="[dateKey, dayData] in sessionsByDay.entries()" :key="dateKey" v-show="activeTab === dateKey" class="day-content">
        <div class="table-wrapper">
          <table class="sessions-table">
            <thead>
              <tr>
                <th class="time-header">Time</th>
                <th v-for="room in dayData.rooms" :key="room" class="room-header">
                  {{ room }}
                </th>
              </tr>
            </thead>
            <tbody>
              <tr 
                v-for="timeSlot in dayData.timeSlots" 
                :key="timeSlot" 
                :class="['time-row', { 'current-time': isCurrentTimeSlot(timeSlot) }]"
                :data-time-slot="timeSlot"
              >
                <td class="time-cell">
                  <div class="time-display">
                    {{ formatTime(timeSlot) }}
                    <span v-if="isCurrentTimeSlot(timeSlot)" class="current-indicator">LIVE</span>
                  </div>
                </td>
                <td v-for="room in dayData.rooms" :key="room" class="session-cell">
                  <div 
                    v-if="dayData.sessionMap.get(timeSlot)?.get(room) && (!showFavoritesOnly || isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId))"
                    :class="['session-content', { 'favorite': isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId) }]"
                  >
                    <div class="session-header">
                      <div class="session-title">
                        {{ dayData.sessionMap.get(timeSlot).get(room).title }}
                      </div>
                      <button 
                        :class="['favorite-btn', { 'favorited': isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId) }]"
                        @click="toggleFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId)"
                        :title="isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId) ? 'Remove from favorites' : 'Add to favorites'"
                      >
                        {{ isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId) ? '❤️' : '🤍' }}
                      </button>
                    </div>
                    <div class="session-info">
                      <div class="session-duration">
                        {{ dayData.sessionMap.get(timeSlot).get(room).length }} min
                      </div>
                      <div class="session-speakers">
                        {{ getSpeakerNames(dayData.sessionMap.get(timeSlot).get(room).speakers) }}
                      </div>
                    </div>
                  </div>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
        
        <div class="stats">
          <p>{{ dayData.displayDate }} - Sessions: {{ dayData.timeSlots.length }}, Rooms: {{ dayData.rooms.length }}</p>
        </div>
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

.sessions-container {
  margin-top: 20px;
}

/* Header Controls */
.header-controls {
  margin-bottom: 20px;
}

.filter-controls {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 15px;
  padding: 15px;
  background-color: #f8f9fa;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.favorites-info {
  display: flex;
  align-items: center;
}

.favorites-count {
  font-size: 14px;
  font-weight: 600;
  color: #2c3e50;
  padding: 8px 12px;
  background-color: white;
  border-radius: 20px;
  border: 2px solid #ecf0f1;
}

.filter-buttons {
  display: flex;
  gap: 8px;
}

.filter-btn {
  padding: 8px 16px;
  border: 2px solid #3498db;
  background-color: white;
  color: #3498db;
  border-radius: 20px;
  cursor: pointer;
  font-weight: 500;
  font-size: 14px;
  transition: all 0.3s ease;
}

.filter-btn:hover:not(:disabled) {
  background-color: #3498db;
  color: white;
  transform: translateY(-1px);
}

.filter-btn.active {
  background-color: #3498db;
  color: white;
}

.filter-btn:disabled {
  background-color: #f8f9fa;
  color: #6c757d;
  border-color: #dee2e6;
  cursor: not-allowed;
  opacity: 0.6;
}

/* Tab Styles */
.tabs-container {
  margin-bottom: 20px;
  border-bottom: 2px solid #ecf0f1;
}

.tabs {
  display: flex;
  gap: 0;
  background-color: #f8f9fa;
  border-radius: 8px 8px 0 0;
  overflow: hidden;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.tab {
  padding: 12px 24px;
  background-color: #f8f9fa;
  color: #6c757d;
  border: none;
  cursor: pointer;
  font-weight: 500;
  font-size: 14px;
  border-bottom: 3px solid transparent;
  transition: all 0.3s ease;
  flex: 1;
  text-align: center;
}

.tab:hover {
  background-color: #e9ecef;
  color: #495057;
}

.tab.active {
  background-color: #fff;
  color: #2c3e50;
  border-bottom-color: #3498db;
  font-weight: 600;
}

.day-content {
  animation: fadeIn 0.3s ease-in-out;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
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
  position: relative;
  transition: all 0.3s ease;
}

.session-content.favorite {
  background-color: #fff8f8;
  border: 2px solid #ffecec;
  border-radius: 6px;
  padding: 6px;
  margin: -2px;
}

.session-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 8px;
  margin-bottom: 8px;
}

.session-title {
  font-weight: bold;
  line-height: 1.3;
  color: #2c3e50;
  font-size: 13px;
  overflow: hidden;
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  flex: 1;
}

.favorite-btn {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 16px;
  padding: 2px;
  border-radius: 50%;
  transition: all 0.2s ease;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
}

.favorite-btn:hover {
  transform: scale(1.1);
  background-color: rgba(231, 76, 60, 0.1);
}

.favorite-btn.favorited {
  animation: heartBeat 0.3s ease-in-out;
}

@keyframes heartBeat {
  0% { transform: scale(1); }
  50% { transform: scale(1.2); }
  100% { transform: scale(1); }
}

.session-info {
  display: flex;
  flex-direction: column;
  gap: 6px;
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

/* Current Time Styles */
.time-row.current-time .time-cell {
  background-color: #e74c3c;
  color: white;
  animation: pulse 2s infinite;
}

.time-row.current-time .session-cell {
  border-left: 4px solid #e74c3c;
  background-color: #fff5f5;
}

.time-display {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}

.current-indicator {
  background-color: #e74c3c;
  color: white;
  padding: 1px 6px;
  border-radius: 8px;
  font-size: 9px;
  font-weight: bold;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  animation: blink 1.5s infinite;
}

@keyframes pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.02); }
  100% { transform: scale(1); }
}

@keyframes blink {
  0%, 50% { opacity: 1; }
  51%, 100% { opacity: 0.6; }
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
