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
  abstract?: string
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
const selectedSession = ref<Session | null>(null)
const showModal = ref(false)

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

    // Get all unique time slots for this day
    const allTimeSlots = [...new Set(sessions.map(session => session.startTimeZulu))]
      .filter(time => time)
      .sort()

    // Filter time slots based on favorites filter
    const timeSlots = showFavoritesOnly.value 
      ? allTimeSlots.filter(timeSlot => {
          // Check if this time slot has any favorite sessions
          return sessions.some(session => 
            session.startTimeZulu === timeSlot && favorites.value.has(session.sessionId)
          )
        })
      : allTimeSlots

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

// Modal functions
const openSessionModal = (session: Session) => {
  selectedSession.value = session
  showModal.value = true
  // Prevent body scroll when modal is open
  document.body.style.overflow = 'hidden'
}

const closeSessionModal = () => {
  selectedSession.value = null
  showModal.value = false
  // Restore body scroll
  document.body.style.overflow = 'auto'
}

// Close modal on escape key
const handleKeyDown = (event: KeyboardEvent) => {
  if (event.key === 'Escape' && showModal.value) {
    closeSessionModal()
  }
}

onMounted(async () => {
  loadFavoritesFromStorage()
  await fetchSessions()
  await initializeView()
  
  // Add escape key listener
  window.addEventListener('keydown', handleKeyDown)
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
        <!-- Desktop Grid View -->
        <div class="grid-wrapper desktop-view">
          <!-- Responsive Grid Layout -->
          <div class="responsive-schedule">
            <template v-for="timeSlot in dayData.timeSlots" :key="timeSlot">
              <div 
                :class="['time-slot-container', { 'current-time-slot': isCurrentTimeSlot(timeSlot) }]"
                :data-time-slot="timeSlot"
              >
                <!-- Time Header -->
                <div class="time-slot-header">
                  <div class="time-display">
                    {{ formatTime(timeSlot) }}
                    <span v-if="isCurrentTimeSlot(timeSlot)" class="current-indicator">LIVE</span>
                  </div>
                </div>
                
                <!-- Sessions Grid -->
                <div class="sessions-row">
                  <div 
                    v-for="room in dayData.rooms" 
                    :key="room"
                    class="session-column"
                    :class="{ 'has-session': dayData.sessionMap.get(timeSlot)?.get(room) && (!showFavoritesOnly || isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId)) }"
                  >
                    <!-- Room Header -->
                    <div class="room-label">
                      {{ room }}
                    </div>
                    
                    <!-- Session Content -->
                    <div class="session-wrapper">
                      <div 
                        v-if="dayData.sessionMap.get(timeSlot)?.get(room) && (!showFavoritesOnly || isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId))"
                        :class="['session-content', { 'favorite': isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId) }]"
                      >
                        <div class="session-header">
                          <a 
                            href="#"
                            class="session-title-link"
                            @click.prevent="openSessionModal(dayData.sessionMap.get(timeSlot).get(room))"
                            :title="'Click to view details for: ' + dayData.sessionMap.get(timeSlot).get(room).title"
                          >
                            <div class="session-title">
                              {{ dayData.sessionMap.get(timeSlot).get(room).title }}
                            </div>
                          </a>
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
                    </div>
                  </div>
                </div>
              </div>
            </template>
          </div>
        </div>
        
        <!-- Mobile Card View -->
        <div class="mobile-view">
          <template v-for="timeSlot in dayData.timeSlots" :key="timeSlot">
            <!-- Only show time slot header if there are sessions for this time slot -->
            <div 
              v-if="dayData.rooms.some(room => dayData.sessionMap.get(timeSlot)?.get(room) && (!showFavoritesOnly || isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId)))"
              class="time-slot-mobile"
            >
              <div :class="['time-header-mobile', { 'current-time': isCurrentTimeSlot(timeSlot) }]" :data-time-slot="timeSlot">
                <div class="time-display-mobile">
                  {{ formatTime(timeSlot) }}
                  <span v-if="isCurrentTimeSlot(timeSlot)" class="current-indicator-mobile">LIVE</span>
                </div>
              </div>
              
              <div class="sessions-grid-mobile">
                <template v-for="room in dayData.rooms" :key="room">
                  <div 
                    v-if="dayData.sessionMap.get(timeSlot)?.get(room) && (!showFavoritesOnly || isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId))"
                    class="session-card-mobile"
                  >
                    <div 
                      :class="['session-content-mobile', { 'favorite': isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId) }]"
                    >
                      <div class="session-room-mobile">
                        {{ room }}
                      </div>
                      
                      <div class="session-header-mobile">
                        <a 
                          href="#"
                          class="session-title-link-mobile"
                          @click.prevent="openSessionModal(dayData.sessionMap.get(timeSlot).get(room))"
                          :title="'Click to view details for: ' + dayData.sessionMap.get(timeSlot).get(room).title"
                        >
                          <div class="session-title-mobile">
                            {{ dayData.sessionMap.get(timeSlot).get(room).title }}
                          </div>
                        </a>
                        <button 
                          :class="['favorite-btn-mobile', { 'favorited': isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId) }]"
                          @click="toggleFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId)"
                          :title="isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId) ? 'Remove from favorites' : 'Add to favorites'"
                        >
                          {{ isFavorite(dayData.sessionMap.get(timeSlot).get(room).sessionId) ? '❤️' : '🤍' }}
                        </button>
                      </div>
                      
                      <div class="session-info-mobile">
                        <div class="session-duration-mobile">
                          {{ dayData.sessionMap.get(timeSlot).get(room).length }} min
                        </div>
                        <div class="session-speakers-mobile">
                          {{ getSpeakerNames(dayData.sessionMap.get(timeSlot).get(room).speakers) }}
                        </div>
                      </div>
                    </div>
                  </div>
                </template>
              </div>
            </div>
          </template>
        </div>
        
        <div class="stats">
          <p>{{ dayData.displayDate }} - Sessions: {{ dayData.timeSlots.length }}, Rooms: {{ dayData.rooms.length }}</p>
        </div>
      </div>
    </div>
    
    <!-- Session Modal -->
    <div v-if="showModal && selectedSession" class="modal-overlay" @click="closeSessionModal">
      <div class="modal-content" @click.stop>
        <div class="modal-header">
          <h2 class="modal-title">{{ selectedSession.title }}</h2>
          <button class="modal-close" @click="closeSessionModal" title="Close modal (ESC)">×</button>
        </div>
        <div class="modal-body">
          <div class="session-meta">
            <div class="meta-item">
              <strong>Room:</strong> {{ selectedSession.room }}
            </div>
            <div class="meta-item">
              <strong>Duration:</strong> {{ selectedSession.length }} minutes
            </div>
            <div class="meta-item">
              <strong>Time:</strong> {{ formatTime(selectedSession.startTimeZulu) }}
            </div>
            <div class="meta-item">
              <strong>Format:</strong> {{ selectedSession.format }}
            </div>
            <div v-if="selectedSession.speakers && selectedSession.speakers.length > 0" class="meta-item">
              <strong>Speaker(s):</strong> {{ getSpeakerNames(selectedSession.speakers) }}
            </div>
          </div>
          
          <div class="session-abstract">
            <h3>Abstract</h3>
            <div v-if="selectedSession.abstract" class="abstract-content">
              {{ selectedSession.abstract }}
            </div>
            <div v-else class="no-abstract">
              <em>No abstract available for this session.</em>
            </div>
          </div>
          
          <div class="modal-actions">
            <button 
              :class="['modal-favorite-btn', { 'favorited': isFavorite(selectedSession.sessionId) }]"
              @click="toggleFavorite(selectedSession.sessionId)"
            >
              {{ isFavorite(selectedSession.sessionId) ? '❤️ Remove from favorites' : '🤍 Add to favorites' }}
            </button>
          </div>
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
  overflow-x: hidden;
}

@media (max-width: 768px) {
  .container {
    padding: 10px;
  }
}

h1 {
  color: #2c3e50;
  margin-bottom: 20px;
  font-size: 24px;
}

@media (max-width: 768px) {
  h1 {
    font-size: 20px;
    margin-bottom: 15px;
    text-align: center;
  }
}

h2 {
  color: #34495e;
  margin-bottom: 15px;
  font-size: 18px;
}

@media (max-width: 768px) {
  h2 {
    font-size: 16px;
    margin-bottom: 10px;
    text-align: center;
  }
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

@media (max-width: 768px) {
  .filter-controls {
    flex-direction: column;
    gap: 12px;
    padding: 12px;
  }
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
  overflow-x: auto;
  -webkit-overflow-scrolling: touch;
}

@media (max-width: 768px) {
  .tabs {
    border-radius: 4px;
    margin-bottom: 10px;
  }
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
  white-space: nowrap;
  min-width: fit-content;
}

@media (max-width: 768px) {
  .tab {
    padding: 10px 16px;
    font-size: 12px;
    flex: none;
    min-width: 120px;
  }
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

/* CSS Grid Layout */
.grid-wrapper {
  margin-bottom: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  background-color: white;
  overflow: hidden;
}

/* Responsive Schedule Layout */
.responsive-schedule {
  display: flex;
  flex-direction: column;
  gap: 0;
}

.time-slot-container {
  border-bottom: 2px solid #dee2e6;
  background-color: white;
}

.time-slot-container.current-time-slot {
  background-color: #fff5f5;
  border-left: 4px solid #e74c3c;
}

.time-slot-header {
  background-color: #2c3e50;
  color: white;
  padding: 12px 16px;
  text-align: center;
  font-weight: bold;
  position: sticky;
  top: 0;
  z-index: 15;
}

.time-slot-container.current-time-slot .time-slot-header {
  background-color: #e74c3c;
  animation: pulse 2s infinite;
}

.sessions-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0;
  min-height: 120px;
}

.session-column {
  flex: 1 1 200px;
  min-width: 200px;
  border-right: 1px solid #dee2e6;
  display: flex;
  flex-direction: column;
}

.session-column:last-child {
  border-right: none;
}

.room-label {
  background-color: #6c757d;
  color: white;
  padding: 8px 6px;
  text-align: center;
  font-size: 12px;
  font-weight: 600;
  border-bottom: 1px solid #dee2e6;
}

.session-wrapper {
  flex: 1;
  padding: 8px;
  display: flex;
  align-items: stretch;
}

.session-wrapper .session-content {
  width: 100%;
}

/* Responsive breakpoints for wrapping */
/* Allow natural wrapping by setting flex-basis to min-width and allowing growth */
@media (min-width: 1201px) {
  .session-column {
    flex: 1 1 300px;
    min-width: 300px;
  }
}

@media (max-width: 1200px) {
  .session-column {
    flex: 1 1 250px;
    min-width: 250px;
  }
}

@media (max-width: 900px) {
  .session-column {
    flex: 1 1 45%;
    min-width: 45%;
    max-width: 48%;
  }
}

@media (max-width: 700px) {
  .session-column {
    flex: 1 1 100%;
    min-width: 100%;
    max-width: 100%;
    border-right: none;
    border-bottom: 1px solid #dee2e6;
  }
  
  .session-column:last-child {
    border-bottom: none;
  }
}

@media (max-width: 800px) {
  .room-label {
    font-size: 11px;
    padding: 6px 4px;
  }
  
  .session-wrapper {
    padding: 6px;
  }
}

@media (max-width: 480px) {
  .time-slot-header {
    padding: 10px 12px;
  }
  
  .room-label {
    font-size: 10px;
    padding: 5px 3px;
  }
  
  .session-wrapper {
    padding: 4px;
  }
}

.sessions-grid {
  display: grid;
  gap: 0;
  font-size: 14px;
}

/* Grid Headers */
.grid-header {
  padding: 12px 8px;
  text-align: center;
  font-weight: bold;
  border-bottom: 2px solid #bdc3c7;
}

.grid-header.time-header {
  background-color: #2c3e50;
  color: white;
  position: sticky;
  left: 0;
  z-index: 10;
}

.grid-header.room-header {
  background-color: #34495e;
  color: white;
}

/* Grid Cells */
.grid-time-cell {
  background-color: #ecf0f1;
  padding: 12px 8px;
  text-align: center;
  font-weight: bold;
  border-right: 2px solid #bdc3c7;
  border-bottom: 1px solid #dee2e6;
  position: sticky;
  left: 0;
  z-index: 5;
  display: flex;
  align-items: center;
  justify-content: center;
}

.grid-time-cell.current-time {
  background-color: #e74c3c;
  color: white;
  animation: pulse 2s infinite;
}

.grid-session-cell {
  padding: 8px;
  border: 1px solid #dee2e6;
  border-top: none;
  min-height: 120px;
  display: flex;
  align-items: stretch;
}

.grid-session-cell.current-time-session {
  border-left: 4px solid #e74c3c;
  background-color: #fff5f5;
}

/* Responsive Grid Columns */
@media (min-width: 1400px) {
  .sessions-grid {
    font-size: 15px;
  }
  .grid-session-cell {
    min-height: 130px;
  }
}

@media (max-width: 1200px) {
  .sessions-grid {
    font-size: 13px;
  }
  .grid-session-cell {
    min-height: 110px;
  }
  .grid-header, .grid-time-cell {
    padding: 10px 6px;
  }
}

@media (max-width: 992px) {
  .sessions-grid {
    font-size: 12px;
  }
  .grid-session-cell {
    min-height: 100px;
    padding: 6px;
  }
  .grid-header, .grid-time-cell {
    padding: 8px 4px;
  }
}

@media (max-width: 768px) {
  .sessions-grid {
    font-size: 11px;
  }
  .grid-session-cell {
    min-height: 90px;
    padding: 4px;
  }
  .grid-header, .grid-time-cell {
    padding: 6px 2px;
  }
}

/* Legacy table styles for compatibility */
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

.session-title-link {
  text-decoration: none;
  color: inherit;
  flex: 1;
  cursor: pointer;
  border-radius: 4px;
  transition: all 0.2s ease;
  display: block;
}

.session-title-link:hover {
  background-color: rgba(52, 152, 219, 0.1);
  transform: translateY(-1px);
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
  padding: 2px 4px;
  border-radius: 2px;
}

.session-title-link:hover .session-title {
  color: #3498db;
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

/* Modal Styles */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  backdrop-filter: blur(2px);
  animation: fadeIn 0.3s ease-out;
}

.modal-content {
  background: white;
  border-radius: 12px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.2);
  max-width: 700px;
  max-height: 80vh;
  width: 90vw;
  overflow-y: auto;
  animation: slideIn 0.3s ease-out;
  position: relative;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(-20px) scale(0.95);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  padding: 24px 24px 16px 24px;
  border-bottom: 2px solid #ecf0f1;
  background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
  border-radius: 12px 12px 0 0;
}

.modal-title {
  margin: 0;
  color: #2c3e50;
  font-size: 22px;
  font-weight: 600;
  line-height: 1.3;
  flex: 1;
  padding-right: 16px;
}

.modal-close {
  background: none;
  border: none;
  font-size: 28px;
  font-weight: bold;
  color: #6c757d;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 50%;
  transition: all 0.2s ease;
  line-height: 1;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-close:hover {
  background-color: #e74c3c;
  color: white;
  transform: scale(1.1);
}

.modal-body {
  padding: 24px;
}

.session-meta {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 12px;
  margin-bottom: 24px;
  padding: 16px;
  background-color: #f8f9fa;
  border-radius: 8px;
  border-left: 4px solid #3498db;
}

.meta-item {
  font-size: 14px;
  color: #495057;
  line-height: 1.4;
}

.meta-item strong {
  color: #2c3e50;
  font-weight: 600;
}

.session-abstract {
  margin-bottom: 24px;
}

.session-abstract h3 {
  color: #2c3e50;
  font-size: 18px;
  font-weight: 600;
  margin: 0 0 16px 0;
  border-bottom: 2px solid #3498db;
  padding-bottom: 8px;
}

.abstract-content {
  color: #495057;
  font-size: 15px;
  line-height: 1.6;
  text-align: justify;
  padding: 16px;
  background-color: #fff;
  border: 1px solid #dee2e6;
  border-radius: 8px;
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.1);
}

.no-abstract {
  color: #6c757d;
  font-style: italic;
  text-align: center;
  padding: 32px 16px;
  background-color: #f8f9fa;
  border-radius: 8px;
  border: 2px dashed #dee2e6;
}

.modal-actions {
  display: flex;
  justify-content: center;
  padding-top: 16px;
  border-top: 1px solid #dee2e6;
}

.modal-favorite-btn {
  padding: 12px 24px;
  border: 2px solid #e74c3c;
  background-color: white;
  color: #e74c3c;
  border-radius: 25px;
  cursor: pointer;
  font-weight: 600;
  font-size: 16px;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 8px;
}

.modal-favorite-btn:hover {
  background-color: #e74c3c;
  color: white;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(231, 76, 60, 0.3);
}

.modal-favorite-btn.favorited {
  background-color: #e74c3c;
  color: white;
}

.modal-favorite-btn.favorited:hover {
  background-color: #c0392b;
  border-color: #c0392b;
}

/* Enhanced Mobile Responsiveness */
@media (max-width: 768px) {
  /* Table improvements */
  .sessions-table {
    font-size: 12px;
  }
  
  .time-header, .room-header {
    padding: 8px 4px;
    font-size: 11px;
  }
  
  .time-cell, .session-cell {
    padding: 6px 4px;
    min-width: 150px;
    height: 100px;
  }
  
  .session-title {
    font-size: 11px;
    -webkit-line-clamp: 2;
  }
  
  .session-speakers {
    font-size: 10px;
    -webkit-line-clamp: 1;
  }
  
  .session-duration {
    font-size: 10px;
    padding: 1px 4px;
  }
  
  /* Improve touch targets */
  .favorite-btn {
    width: 32px;
    height: 32px;
    font-size: 18px;
  }
  
  .session-title-link {
    padding: 4px;
    min-height: 32px;
  }
  
  /* Filter buttons */
  .filter-btn {
    padding: 12px 20px;
    font-size: 16px;
    min-height: 44px;
    border-radius: 25px;
  }
  
  .filter-buttons {
    flex-direction: column;
    width: 100%;
    gap: 8px;
  }
  
  .filter-btn {
    width: 100%;
  }
  
  /* Modal improvements */
  .modal-content {
    width: 95vw;
    max-height: 90vh;
    border-radius: 8px;
  }
  
  .modal-header {
    padding: 16px;
  }
  
  .modal-title {
    font-size: 18px;
  }
  
  .modal-close {
    width: 44px;
    height: 44px;
    font-size: 24px;
  }
  
  .modal-body {
    padding: 16px;
  }
  
  .session-meta {
    grid-template-columns: 1fr;
    gap: 8px;
    padding: 12px;
  }
  
  .modal-favorite-btn {
    padding: 16px 24px;
    font-size: 18px;
    width: 100%;
    justify-content: center;
    min-height: 44px;
  }
  
  /* Stats */
  .stats {
    padding: 10px;
    font-size: 12px;
  }
  
  /* Time display improvements */
  .time-display {
    font-size: 11px;
  }
  
  .current-indicator {
    font-size: 8px;
    padding: 1px 4px;
  }
}

/* Extra small screens */
@media (max-width: 480px) {
  .container {
    padding: 5px;
  }
  
  .time-cell, .session-cell {
    min-width: 120px;
    height: 90px;
    padding: 4px 2px;
  }
  
  .session-title {
    font-size: 10px;
  }
  
  .session-speakers {
    font-size: 9px;
  }
  
  .session-duration {
    font-size: 9px;
  }
  
  .tab {
    min-width: 100px;
    padding: 8px 12px;
    font-size: 11px;
  }
  
  .modal-content {
    width: 98vw;
    max-height: 95vh;
  }
  
  .modal-header {
    padding: 12px;
  }
  
  .modal-title {
    font-size: 16px;
  }
  
  .modal-body {
    padding: 12px;
  }
}

/* Desktop/Tablet View - Show grid only on larger screens */
.desktop-view {
  display: block;
}

.mobile-view {
  display: none;
}

/* Add horizontal scrolling for medium screens that still use grid */
@media (max-width: 1024px) {
  .grid-wrapper {
    overflow-x: auto;
    -webkit-overflow-scrolling: touch;
  }
  
  .sessions-grid {
    min-width: 800px; /* Ensure minimum width so columns don't get too cramped */
  }
}

/* Switch to mobile card view on smaller screens */
@media (max-width: 768px) {
  .desktop-view {
    display: none !important;
  }
  
  .mobile-view {
    display: block !important;
  }
}

/* Mobile Time Slot Styles */
.time-slot-mobile {
  margin-bottom: 28px;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
}

.time-header-mobile {
  background-color: #2c3e50;
  color: white;
  padding: 14px 16px;
  font-weight: bold;
  text-align: center;
  position: sticky;
  top: 0;
  z-index: 20;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.time-header-mobile.current-time {
  background-color: #e74c3c;
  animation: pulse 2s infinite;
}

.time-display-mobile {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  font-size: 16px;
}

.current-indicator-mobile {
  background-color: rgba(255, 255, 255, 0.2);
  color: white;
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 10px;
  font-weight: bold;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  animation: blink 1.5s infinite;
}

/* Sessions Grid for Mobile */
.sessions-grid-mobile {
  display: grid;
  grid-template-columns: 1fr;
  gap: 12px;
  padding: 16px;
  background-color: #f8f9fa;
  border-radius: 0 0 8px 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

@media (max-width: 480px) {
  .sessions-grid-mobile {
    padding: 12px;
    gap: 10px;
  }
}

/* Session Card Mobile */
.session-card-mobile {
  background: white;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  transition: all 0.3s ease;
  border: 1px solid #e9ecef;
}

.session-card-mobile:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);
}

.session-content-mobile {
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
  height: 100%;
}

.session-content-mobile.favorite {
  background-color: #fff8f8;
  border-color: #ffecec;
}

@media (max-width: 480px) {
  .session-content-mobile {
    padding: 12px;
    gap: 10px;
  }
}

/* Room Label Mobile */
.session-room-mobile {
  background-color: #6c757d;
  color: white;
  padding: 6px 12px;
  border-radius: 16px;
  font-size: 12px;
  font-weight: 600;
  text-align: center;
  margin: -16px -16px 0 -16px;
  position: sticky;
  top: 0;
  z-index: 10;
}

@media (max-width: 480px) {
  .session-room-mobile {
    margin: -12px -12px 0 -12px;
    font-size: 11px;
    padding: 5px 10px;
  }
}

/* Session Header Mobile */
.session-header-mobile {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 12px;
}

.session-title-link-mobile {
  text-decoration: none;
  color: inherit;
  flex: 1;
  cursor: pointer;
  border-radius: 6px;
  transition: all 0.2s ease;
  display: block;
  padding: 8px;
  margin: -8px;
}

.session-title-link-mobile:hover {
  background-color: rgba(52, 152, 219, 0.08);
  transform: translateY(-1px);
}

.session-title-mobile {
  font-weight: 600;
  line-height: 1.4;
  color: #2c3e50;
  font-size: 16px;
  margin-bottom: 4px;
}

.session-title-link-mobile:hover .session-title-mobile {
  color: #3498db;
}

@media (max-width: 480px) {
  .session-title-mobile {
    font-size: 14px;
  }
}

.favorite-btn-mobile {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 24px;
  padding: 8px;
  border-radius: 50%;
  transition: all 0.2s ease;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 44px;
  height: 44px;
  margin: -8px;
}

.favorite-btn-mobile:hover {
  transform: scale(1.1);
  background-color: rgba(231, 76, 60, 0.1);
}

.favorite-btn-mobile.favorited {
  animation: heartBeat 0.3s ease-in-out;
}

/* Session Info Mobile */
.session-info-mobile {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding-top: 8px;
  border-top: 1px solid #e9ecef;
}

.session-duration-mobile {
  background-color: #3498db;
  color: white;
  padding: 4px 10px;
  border-radius: 16px;
  font-size: 12px;
  font-weight: 600;
  text-align: center;
  align-self: flex-start;
}

@media (max-width: 480px) {
  .session-duration-mobile {
    font-size: 11px;
    padding: 3px 8px;
  }
}

.session-speakers-mobile {
  font-size: 14px;
  color: #7f8c8d;
  font-style: italic;
  line-height: 1.3;
}

@media (max-width: 480px) {
  .session-speakers-mobile {
    font-size: 12px;
  }
}

/* Mobile-specific responsive adjustments */
@media (max-width: 768px) {
  .time-slot-mobile {
    margin-bottom: 20px;
  }
  
  .time-header-mobile {
    padding: 10px 14px;
    font-size: 14px;
  }
  
  .time-display-mobile {
    font-size: 14px;
  }
}

@media (max-width: 480px) {
  .time-slot-mobile {
    margin-bottom: 16px;
  }
  
  .time-header-mobile {
    padding: 8px 12px;
    font-size: 13px;
  }
  
  .time-display-mobile {
    font-size: 13px;
  }
  
  .session-card-mobile:hover {
    transform: none;
  }
}
</style>
