*/
  Should refactor separate components, e.g. results table but internal tool, so leaving as is for now
*/
<script setup lang="ts">
defineProps<{ msg: string }>()

import { ref, onMounted } from 'vue'

// Configuration
const API_BASE = 'https://cms4.wowvegas.com/api/ams/v1'
const DEBUG = false // set true to enable console logs

// Helper: fetch with timeout
async function fetchWithTimeout(input: RequestInfo, timeout = 10000) {
  const controller = new AbortController()
  const id = setTimeout(() => controller.abort(), timeout)
  try {
    const res = await fetch(input, { signal: controller.signal })
    return res
  } finally {
    clearTimeout(id)
  }
}

// Helper: check whether partyId exists in common response shapes
function partyIdExists(data: any, partyId: string): boolean {
  if (!data) return false
  if (Array.isArray(data.entries)) {
    return data.entries.some((e: any) => e.partyId === partyId || e.playerId === partyId || e.playerName === partyId)
  }
  if (Array.isArray(data.players)) {
    return data.players.some((p: any) => p.partyId === partyId || p.playerId === partyId || p.name === partyId)
  }
  if (data.partyId === partyId || data.playerId === partyId || data.playerName === partyId) return true
  // last-resort: string search but anchored to word boundaries to reduce false positives
  try {
    const json = JSON.stringify(data)
    const re = new RegExp(`\\b${partyId.replace(/[-/\\^$*+?.()|[\]{}]/g, '\\$&')}\\b`, 'i')
    return re.test(json)
  } catch (e) {
    return false
  }
}

const searchTerm = ref('')
interface Leaderboard {
  id?: string
  _id?: string
  leaderboardId?: string
  leaderboardName?: string
  title?: string
  status?: string
}

const leaderboards = ref<Leaderboard[]>([])
const loading = ref(false)
const error = ref('')
const results = ref<{ leaderboard: Leaderboard; result: any; time: string }[]>([])

onMounted(async () => {
  try {
  const res = await fetchWithTimeout(`${API_BASE}/leaderboards`, 10000)
    if (!res.ok) throw new Error('Failed to fetch leaderboards')
    const data = await res.json()
  if (DEBUG) console.log('Raw leaderboards API response:', JSON.stringify(data))
    // If the API response is an object, try to extract the array
    let arr = []
    if (Array.isArray(data.data)) {
      arr = data.data
    }
    // Filter out CANCELLED, FINISHED, AND DRAFT LBs as data not retained on them to search anyway
    leaderboards.value = arr.filter((lb: Leaderboard) => lb.status !== 'CANCELLED' && lb.status !== 'FINISHED' && lb.status !== 'DRAFT')
    if (!Array.isArray(leaderboards.value)) {
      leaderboards.value = []
      error.value = 'No leaderboards found in response.'
    }
  } catch (e) {
    error.value = (e instanceof Error ? e.message : String(e))
  }
})

function handleInput(e: Event) {
  const val = (e.target as HTMLInputElement).value.replace(/\s/g, '')
  searchTerm.value = val
}

async function handleSearch() {
  error.value = ''
  results.value = []
  // normalize input
  searchTerm.value = searchTerm.value.trim()
  if (!searchTerm.value) {
    error.value = 'Please enter a username.'
    return
  }
  loading.value = true
  try {
    // Use America/New_York timezone for ET
    const now = new Date()
    // Get current ET time
    const etNow = new Date(now.toLocaleString('en-US', { timeZone: 'America/New_York' }))
    // Start from previous hour
    let searchHour = etNow.getHours() - 1
    const found = []
    // Log the actual array value
    console.log('Leaderboards to search:', JSON.stringify(leaderboards.value))
    // Loop from previous hour down to 0 (midnight)
    for (const lb of leaderboards.value) {
      const id = lb.id || lb._id || lb.leaderboardId
      if (!id) continue
      for (let h = searchHour; h >= 0; h--) {
        const hourStr = String(h).padStart(2, '0')
        const time = `${hourStr}:00:00`
        const url = `${API_BASE}/leaderboards/${id}?partyId=${encodeURIComponent(searchTerm.value)}&hour=${time}`
        if (DEBUG) console.log('Requesting:', url)
        const res = await fetchWithTimeout(url, 10000)
        let data = null
        if (res && res.ok) {
          try {
            data = await res.json()
            if (DEBUG) console.log('Response for', url, ':', data)
          } catch (err) {
            if (DEBUG) console.log('Error parsing response for', url, err)
          }
          // Only push if partyId is found in the response
          if (partyIdExists(data, searchTerm.value)) {
            found.push({ leaderboard: lb, result: data, time })
          }
        } else {
          if (DEBUG) console.log('Request failed:', url, res && res.status)
        }
      }
    }
    results.value = found
    if (found.length === 0) error.value = 'No results found.'
  } catch (e) {
    error.value = (e instanceof Error ? e.message : String(e))
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <h2>{{ msg }}</h2>
  <div class="card">
    <p>
      Enter a username to find out where they placed in an hourly leaderboard today
    </p>
    <input
      type="text"
      v-model="searchTerm"
      @input="handleInput"
      placeholder="Enter username"
      autocomplete="off"
    />
    <button type="button" @click="handleSearch" :disabled="loading">Search</button>
    <div v-if="loading" class="loader"></div>
    <div v-if="error" style="color:red">{{ error }}</div>
    <div v-if="results.length">
  <h3>Results: <span class="results-count">{{ results.length }}</span></h3>
      <table class="results-table">
        <thead>
          <tr>
            <th>Leaderboard</th>
            <th>Hour</th>
            <th>Result</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item in results" :key="item.leaderboard.id || item.leaderboard._id + '-' + item.time">
            <td>{{ item.leaderboard.leaderboardName || item.leaderboard.title || item.leaderboard.id }}</td>
            <td>{{ item.time }}</td>
            <td><pre>{{ item.result }}</pre></td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<style scoped>
.card {
  margin-top: 1em;
  padding: 1em;
  border: 1px solid #eee;
  border-radius: 8px;
  background: #fafafa;
  color: #2c2c2c;
}
input[type="text"] {
  margin-right: 1em;
  padding: 0.5em;
  border-radius: 4px;
  border: 1px solid #ccc;
}
button {
  padding: 0.5em 1em;
  border-radius: 4px;
  border: none;
  background: #0a84ff;
  color: #fff;
  cursor: pointer;
}
button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

/* Results table styles */
.results-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 1em;
  background: #fff;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}
.results-table th, .results-table td {
  border: 1px solid #ddd;
  padding: 0.75em 1em;
  text-align: left;
}
.results-table th {
  background: #f4f4f4;
  font-weight: bold;
}
.results-table tr:nth-child(even) {
  background: #fafafa;
}
.results-table pre {
  margin: 0;
  font-size: 0.95em;
  white-space: pre-wrap;
  word-break: break-word;
}
</style>