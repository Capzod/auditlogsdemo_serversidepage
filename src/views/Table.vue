<script setup>
import { ref, onMounted, watch, computed } from 'vue'
import axios from 'axios'

/* =========================
   CONFIG
========================= */
const API_BASE = import.meta.env.VITE_API_BASE_URL

/* =========================
   STATE
========================= */
const items = ref([])
const totalItems = ref(0)
const loading = ref(false)

const page = ref(1)
const itemsPerPage = ref(10)

/* FILTERS */
const filters = ref({})
const filterMenu = ref({})

/* COLUMN VISIBILITY */
const visibleColumns = ref([])
const columnPresets = ref({
  default: [],
  essential: [],
  all: []
})

/* =========================
   HEADERS
========================= */
const headers = ref([])

const formatHeader = (key) => {
  return key
    .replace(/([A-Z])/g, ' $1')
    .replace(/^./, str => str.toUpperCase())
    .replace('Id', 'ID')
    .replace('Url', 'URL')
    .replace('Ip', 'IP')
    .replace('Api', 'API')
    .replace('Sso', 'SSO')
    .replace('Mfa', 'MFA')
}

const generateHeaders = (data) => {
  if (!data || !data.length) return

  const allHeaders = Object.keys(data[0]).map(key => ({
    title: formatHeader(key),
    key,
    width: '160px',
    sortable: true
  }))

  headers.value = allHeaders
  
  // Initializing visible columns with first 15 columns by default
  if (visibleColumns.value.length === 0) {
    visibleColumns.value = allHeaders.slice(0, 15).map(h => h.key)
  }
}

/* =========================
   COMPUTED PROPERTIES
========================= */
const hasFilters = computed(() =>
  Object.values(filters.value).some(v => v && v.trim() !== '')
)

const activeFilters = computed(() => {
  return Object.entries(filters.value)
    .filter(([_, value]) => value && value.trim() !== '')
    .map(([key, value]) => ({ key, value }))
})

const visibleHeaders = computed(() => {
  return headers.value.filter(h => visibleColumns.value.includes(h.key))
})

const columnOptions = computed(() => {
  return headers.value.map(h => ({
    title: h.title,
    value: h.key,
    selected: visibleColumns.value.includes(h.key)
  }))
})

/* =========================
   HELPERS
========================= */
const formatDateTime = (value) => {
  if (!value) return '—'
  try {
    const d = new Date(value)
    return d.toLocaleString()
  } catch {
    return value
  }
}

const formatDate = (value) => {
  if (!value) return '—'
  try {
    return new Date(value).toLocaleDateString()
  } catch {
    return value
  }
}

const formatTime = (value) => {
  if (!value) return ''
  try {
    return new Date(value).toLocaleTimeString([], { 
      hour: '2-digit', 
      minute: '2-digit' 
    })
  } catch {
    return ''
  }
}

const truncateText = (text, length = 25) => {
  if (!text) return '—'
  if (typeof text !== 'string') return String(text)
  return text.length > length ? text.substring(0, length) + '…' : text
}

const isDateField = (key) => {
  const datePatterns = ['date', 'time', 'created', 'modified', 'updated', 'timestamp']
  return datePatterns.some(pattern => key.toLowerCase().includes(pattern))
}

const isIdField = (key) => {
  return key.toLowerCase().includes('id') || key.toLowerCase().includes('guid')
}

const clearFilter = (key) => {
  filters.value[key] = ''
  fetchLogs()
}

const clearAllFilters = () => {
  Object.keys(filters.value).forEach(key => {
    filters.value[key] = ''
  })
}

const toggleColumn = (key) => {
  if (visibleColumns.value.includes(key)) {
    visibleColumns.value = visibleColumns.value.filter(k => k !== key)
  } else {
    visibleColumns.value = [...visibleColumns.value, key]
  }
}

const showAllColumns = () => {
  visibleColumns.value = headers.value.map(h => h.key)
}

const showDefaultColumns = () => {
  visibleColumns.value = headers.value.slice(0, 15).map(h => h.key)
}

const showEssentialColumns = () => {
  const essential = ['id', 'createdDateTime', 'userPrincipalName', 'appDisplayName', 'status', 'ipaddress']
  visibleColumns.value = headers.value
    .filter(h => essential.includes(h.key))
    .map(h => h.key)
}

const getRiskColor = (risk) => {
  if (!risk) return 'grey-lighten-3'
  risk = risk.toLowerCase()
  if (risk.includes('high')) return 'red-lighten-4'
  if (risk.includes('medium')) return 'orange-lighten-4'
  if (risk.includes('low')) return 'green-lighten-4'
  if (risk.includes('none')) return 'blue-lighten-4'
  return 'grey-lighten-3'
}

const getRiskTextColor = (risk) => {
  if (!risk) return 'text-grey-darken-3'
  risk = risk.toLowerCase()
  if (risk.includes('high')) return 'text-red-darken-3'
  if (risk.includes('medium')) return 'text-orange-darken-3'
  if (risk.includes('low')) return 'text-green-darken-3'
  if (risk.includes('none')) return 'text-blue-darken-3'
  return 'text-grey-darken-3'
}

/* =========================
   FETCH DATA
========================= */
const fetchLogs = async () => {
  loading.value = true

  try {
    let response = null

    if (hasFilters.value) {
      const payload = Object.entries(filters.value)
        .filter(([_, v]) => v && v.trim() !== '')
        .map(([key, value]) => ({
          columnName: key,
          operator: 'contains',
          value
        }))

      response = await axios.post(
        `${API_BASE}/api/signin-logs/filter`,
        payload,
        {
          params: {
            page: page.value,
            pageSize: itemsPerPage.value
          }
        }
      )
    } else {
      response = await axios.get(
        `${API_BASE}/api/signin-logs`,
        {
          params: {
            page: page.value,
            pageSize: itemsPerPage.value
          }
        }
      )
    }

    if (!response || !response.data || !Array.isArray(response.data.data)) {
      console.warn("Invalid API structure:", response)
      items.value = []
      totalItems.value = 0
      return
    }

    items.value = response.data.data
    generateHeaders(items.value)
    totalItems.value = response.data.totalCount ?? response.data.data.length

  } catch (err) {
    console.error("API ERROR:", err)
    items.value = []
    totalItems.value = 0
  } finally {
    loading.value = false
  }
}

/* =========================
   WATCHERS
========================= */
watch([page, itemsPerPage], fetchLogs)

onMounted(fetchLogs)
</script>

<template>
  <v-container fluid class="pa-4">
    <v-card class="elevation-1" rounded="lg">
      <!-- HEADER -->
      <v-card-title class="pa-5">
        <div class="d-flex align-center">
          <v-avatar color="primary" size="40" class="mr-4" variant="tonal">
            <v-icon icon="mdi-shield-lock" color="primary" />
          </v-avatar>
          <div>
            <div class="d-flex align-center">
              <h2 class="text-h5 font-weight-medium text-primary">Sign-In Logs</h2>
              <v-chip
                color="primary"
                variant="tonal"
                size="small"
                class="ml-4"
              >
                {{ headers.length }} columns
              </v-chip>
            </div>
            <div class="d-flex align-center mt-1">
              <span class="text-caption text-grey-darken-1 mr-3">
                Server-side pagination 
              </span>
              <v-chip
                size="x-small"
                color="transparent"
                variant="tonal"
                prepend-icon="mdi-database"
                class="mr-2"
              >
                {{ totalItems.toLocaleString() }} records
              </v-chip>
            </div>
          </div>
        </div>
      </v-card-title>

      <!-- ACTION BAR - COLUMNS & REFRESH ON LEFT -->
      <v-card-text class="pa-4 pt-0">
        <v-row align="center" justify="space-between">
          <v-col cols="12" md="6" class="d-flex align-center">
            <!-- COLUMN VISIBILITY MENU -->
            <v-menu
              :close-on-content-click="false"
              location="bottom-start"
              width="320"
              offset="5"
            >
              <template #activator="{ props }">
                <v-btn
                  variant="outlined"
                  density="comfortable"
                  prepend-icon="mdi-table-column"
                  v-bind="props"
                  class="mr-3"
                  color="primary"
                >
                  Columns
                  <v-chip size="x-small" color="primary" class="ml-2">{{ visibleColumns.length }}/{{ headers.length }}</v-chip>
                </v-btn>
              </template>

              <v-card class="pa-4" elevation="4" rounded="lg">
                <div class="d-flex align-center justify-space-between mb-3">
                  <span class="text-subtitle-2 font-weight-medium">Column Visibility</span>
                  <div>
                    <v-btn size="x-small" variant="text" @click="showDefaultColumns" class="mr-1">Default</v-btn>
                    <v-btn size="x-small" variant="text" @click="showEssentialColumns" class="mr-1">Essential</v-btn>
                    <v-btn size="x-small" variant="text" @click="showAllColumns">All</v-btn>
                  </div>
                </div>
                <v-divider class="mb-3" />
                <div style="max-height: 400px; overflow-y: auto;">
                  <v-checkbox
                    v-for="col in columnOptions"
                    :key="col.value"
                    :label="col.title"
                    :model-value="col.selected"
                    @update:model-value="toggleColumn(col.value)"
                    density="compact"
                    hide-details
                    class="my-1"
                    color="primary"
                  />
                </div>
              </v-card>
            </v-menu>

            <v-btn
              variant="outlined"
              density="comfortable"
              prepend-icon="mdi-refresh"
              @click="fetchLogs"
              :loading="loading"
              color="primary"
            >
              Refresh
            </v-btn>
          </v-col>
          
          <v-col cols="12" md="6" class="text-right">
            <div v-if="hasFilters" class="d-inline-flex align-center">
              <v-icon icon="mdi-filter-variant" size="18" color="primary" class="mr-1" />
              <span class="text-caption text-grey-darken-1 mr-2">Filters active</span>
              <v-btn size="x-small" variant="text" @click="clearAllFilters" color="primary">
                Clear all
              </v-btn>
            </div>
          </v-col>
        </v-row>
      </v-card-text>

      <!-- ACTIVE FILTERS CHIPS -->
      <v-card-text class="pa-4 pt-0" v-if="hasFilters">
        <div class="d-flex flex-wrap gap-2">
          <v-chip
            v-for="filter in activeFilters"
            :key="filter.key"
            size="small"
            density="comfortable"
            color="blue-lighten-4"
            closable
            @click:close="clearFilter(filter.key)"
            class="text-grey-darken-3"
          >
            <span class="font-weight-medium mr-1">{{ 
              headers.find(h => h.key === filter.key)?.title || filter.key
            }}:</span>
            {{ truncateText(filter.value, 20) }}
          </v-chip>
        </div>
      </v-card-text>

      <v-divider />

      <!-- TABLE CONTAINER WITH HORIZONTAL SCROLL -->
      <div style="overflow-x: auto; overflow-y: hidden; position: relative;">
        <v-data-table-server
          :headers="visibleHeaders"
          :items="items"
          :items-length="totalItems"
          :loading="loading"
          :page="page"
          :items-per-page="itemsPerPage"
          @update:page="page = $event"
          @update:items-per-page="itemsPerPage = $event"
          class="elevation-0 elegant-table"
          :loading-text="'Loading Sign-In logs...'"
          :no-data-text="'No sign-in logs found'"
          fixed-header
          height="65vh"
          density="compact"
        >
          <!-- ELEGANT HEADERS -->
          <template #headers="{ columns }">
            <tr>
              <th v-for="col in columns" :key="col.key" class="elegant-header-cell">
                <div class="d-flex align-center">
                  <span class="elegant-header-text">{{ col.title }}</span>
                  
                  <!-- FILTER ICON -->
                  <v-menu
                    :close-on-content-click="false"
                    v-model="filterMenu[col.key]"
                    location="bottom"
                    offset="4"
                  >
                    <template #activator="{ props }">
                      <v-btn
                        icon
                        size="x-small"
                        variant="text"
                        v-bind="props"
                        :color="filters[col.key] ? 'primary' : 'grey-lighten-1'"
                        density="compact"
                        class="filter-btn"
                      >
                        <v-icon :size="filters[col.key] ? 16 : 14">
                          {{ filters[col.key] ? 'mdi-filter' : 'mdi-filter-outline' }}
                        </v-icon>
                      </v-btn>
                    </template>

                    <v-card class="pa-3" width="260" elevation="8" rounded="lg">
                      <div class="text-body-2 font-weight-medium mb-2">
                        Filter by {{ col.title }}
                      </div>
                      <v-text-field
                        v-model="filters[col.key]"
                        density="compact"
                        variant="outlined"
                        hide-details
                        :placeholder="`Search...`"
                        :prepend-inner-icon="filters[col.key] ? 'mdi-filter-check' : 'mdi-filter'"
                        clearable
                        @keyup.enter="fetchLogs"
                        @click:clear="clearFilter(col.key)"
                      />
                      <div class="d-flex justify-end mt-3">
                        <v-btn
                          size="small"
                          variant="text"
                          @click="clearFilter(col.key)"
                          class="mr-2"
                        >
                          Clear
                        </v-btn>
                        <v-btn
                          size="small"
                          color="primary"
                          @click="fetchLogs"
                        >
                          Apply
                        </v-btn>
                      </div>
                    </v-card>
                  </v-menu>
                </div>
              </th>
            </tr>
          </template>

          <!-- DYNAMIC CELL RENDERING WITH TOOLTIPS -->
          <template #item="{ item }">
            <tr class="hover-row">
              <td v-for="col in visibleHeaders" :key="col.key" class="elegant-cell">
                <template v-if="item[col.key] !== undefined && item[col.key] !== null && item[col.key] !== ''">
                  <!-- Date fields with tooltip -->
                  <v-tooltip v-if="isDateField(col.key)" location="top" :text="formatDateTime(item[col.key])">
                    <template #activator="{ props }">
                      <div v-bind="props" class="d-inline-block">
                        <div class="text-caption font-weight-medium text-grey-darken-3">
                          {{ formatDate(item[col.key]) }}
                        </div>
                        <div class="text-caption text-grey-darken-1">
                          {{ formatTime(item[col.key]) }}
                        </div>
                      </div>
                    </template>
                  </v-tooltip>
                  
                  <!-- ID fields with tooltip -->
                  <v-tooltip v-else-if="isIdField(col.key)" location="top" :text="item[col.key]">
                    <template #activator="{ props }">
                      <span v-bind="props" class="text-caption font-family-monospace text-grey-darken-3">
                        {{ truncateText(item[col.key], 12) }}
                      </span>
                    </template>
                  </v-tooltip>
                  
                  <!-- Email/UPN fields with tooltip -->
                  <v-tooltip v-else-if="col.key.includes('mail') || col.key.includes('Principal') || col.key.includes('Email')" 
                             location="top" :text="item[col.key]">
                    <template #activator="{ props }">
                      <span v-bind="props" class="text-body-2 text-truncate text-grey-darken-3 d-inline-block" 
                            style="max-width: 200px;">
                        {{ item[col.key] }}
                      </span>
                    </template>
                  </v-tooltip>
                  
                  <!-- Boolean fields -->
                  <v-chip
                    v-else-if="typeof item[col.key] === 'boolean'"
                    size="x-small"
                    density="compact"
                    :color="item[col.key] ? 'green-lighten-4' : 'grey-lighten-3'"
                    :class="item[col.key] ? 'text-green-darken-3' : 'text-grey-darken-2'"
                    label
                  >
                    {{ item[col.key] ? 'Yes' : 'No' }}
                  </v-chip>
                  
                  <!-- Status/Risk fields with tooltip -->
                  <v-tooltip v-else-if="col.key.toLowerCase().includes('status') || col.key.toLowerCase().includes('risk') || col.key.toLowerCase().includes('level')" 
                             location="top" :text="item[col.key]">
                    <template #activator="{ props }">
                      <v-chip
                        v-bind="props"
                        size="x-small"
                        density="compact"
                        :color="col.key.toLowerCase().includes('risk') ? getRiskColor(item[col.key]) : 'grey-lighten-3'"
                        :class="col.key.toLowerCase().includes('risk') ? getRiskTextColor(item[col.key]) : 'text-grey-darken-3'"
                        label
                      >
                        {{ item[col.key] }}
                      </v-chip>
                    </template>
                  </v-tooltip>
                  
                  <!-- Number fields with tooltip -->
                  <v-tooltip v-else-if="typeof item[col.key] === 'number'" location="top" :text="item[col.key].toLocaleString()">
                    <template #activator="{ props }">
                      <span v-bind="props" class="text-caption text-grey-darken-3 font-family-monospace">
                        {{ item[col.key].toLocaleString() }}
                      </span>
                    </template>
                  </v-tooltip>
                  
                  <!-- Default text fields with tooltip -->
                  <v-tooltip v-else location="top" :text="item[col.key]">
                    <template #activator="{ props }">
                      <span v-bind="props" class="text-body-2 text-truncate text-grey-darken-3 d-inline-block" 
                            style="max-width: 180px;">
                        {{ truncateText(item[col.key], 25) }}
                      </span>
                    </template>
                  </v-tooltip>
                </template>
                <span v-else class="text-caption text-grey-lighten-1">—</span>
              </td>
            </tr>
          </template>

          <!-- LOADING STATE -->
          <template #loading>
            <tr>
              <td :colspan="visibleHeaders.length" class="pa-10 text-center">
                <v-progress-circular
                  indeterminate
                  color="primary"
                  size="40"
                  width="2"
                />
                <div class="text-body-2 text-grey-darken-1 mt-4">
                  Loading {{ headers.length }} columns...
                  <span v-if="hasFilters" class="d-block mt-1">Applying server-side filters</span>
                </div>
              </td>
            </tr>
          </template>

          <!-- NO DATA STATE -->
          <template #no-data>
            <tr>
              <td :colspan="visibleHeaders.length" class="pa-10 text-center">
                <v-icon 
                  :icon="hasFilters ? 'mdi-filter-off' : 'mdi-file-search-outline'" 
                  size="56" 
                  class="text-grey-lighten-2 mb-4" 
                />
                <h4 class="text-h6 font-weight-regular mb-2 text-grey-darken-2">
                  {{ hasFilters ? 'No matching records' : 'No sign-in logs' }}
                </h4>
                <p class="text-body-2 text-grey-darken-1 mb-4">
                  {{ hasFilters ? 'Try adjusting your filters' : 'Data will appear here once available' }}
                </p>
                <div>
                  <v-btn
                    v-if="hasFilters"
                    variant="outlined"
                    @click="clearAllFilters"
                    prepend-icon="mdi-filter-remove"
                    class="mr-2"
                  >
                    Clear Filters
                  </v-btn>
                  <v-btn
                    variant="tonal"
                    @click="fetchLogs"
                    prepend-icon="mdi-refresh"
                  >
                    Refresh
                  </v-btn>
                </div>
              </td>
            </tr>
          </template>

          <!-- TABLE FOOTER -->
          <template #bottom>
            <div class="text-caption text-grey-darken-1 px-4 py-3 border-t d-flex justify-space-between">
              <div>
                <span v-if="hasFilters" class="mr-3">
                  <v-icon icon="mdi-filter-check" size="16" class="mr-1" color="primary" />
                  <span class="text-primary">Filtered</span>
                </span>
                Showing {{ ((page - 1) * itemsPerPage) + 1 }}-{{ Math.min(page * itemsPerPage, totalItems) }} 
                of {{ totalItems.toLocaleString() }}
              </div>
              <div class="d-flex align-center">
                <span class="mr-2">Page {{ page }} of {{ Math.ceil(totalItems / itemsPerPage) }}</span>
                <span class="text-grey-lighten-1 mx-2">|</span>
                <span>{{ visibleColumns.length }} columns</span>
              </div>
            </div>
          </template>
        </v-data-table-server>
      </div>

      <!-- PAGINATION CONTROLS -->
      <v-divider />
      <v-card-actions class="pa-4">
        <v-row align="center" justify="space-between" dense>
          <v-col cols="12" md="3">
            <div class="d-flex align-center">
              <span class="text-caption text-grey-darken-1 mr-3">
                Rows per page:
              </span>
              <v-select
                v-model="itemsPerPage"
                :items="[10, 25, 50, 100, 250, 500]"
                density="compact"
                variant="outlined"
                hide-details
                style="width: 110px"
              />
            </div>
          </v-col>
          
          <v-col cols="12" md="6" class="text-center">
            <v-pagination
              v-model="page"
              :length="Math.ceil(totalItems / itemsPerPage)"
              :total-visible="5"
              density="comfortable"
              color="primary"
              rounded
            />
          </v-col>
          
          <v-col cols="12" md="3" class="text-right">
            <div class="text-caption text-grey-darken-1">
              {{ items.length }} rows loaded
            </div>
          </v-col>
        </v-row>
      </v-card-actions>
    </v-card>
  </v-container>
</template>

<style scoped>
/* ELEGANT HEADER STYLES */
.elegant-header-cell {
  padding: 14px 12px !important;
  background-color: #ffffff !important;
  border-bottom: 1px solid #eaeef2 !important;
  font-weight: 500 !important;
  transition: background-color 0.2s ease;
}

.elegant-header-cell:hover {
  background-color: #f8fafc !important;
}

.elegant-header-text {
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.3px;
  text-transform: uppercase;
  color: #5f6b7a;
  margin-right: 6px;
}

.filter-btn {
  opacity: 0.6;
  transition: opacity 0.2s ease, color 0.2s ease;
}

.filter-btn:hover {
  opacity: 1;
  background-color: #f0f3f7 !important;
}

/* ELEGANT CELL STYLES */
.elegant-cell {
  padding: 8px 12px !important;
  border-bottom: 1px solid #f0f3f7 !important;
  transition: background-color 0.2s ease;
}

.hover-row:hover .elegant-cell {
  background-color: #f8fafc !important;
}

/* CUSTOM SCROLLBAR */
:deep(::-webkit-scrollbar) {
  width: 8px;
  height: 8px;
}

:deep(::-webkit-scrollbar-track) {
  background: #f5f7fa;
  border-radius: 4px;
}

:deep(::-webkit-scrollbar-thumb) {
  background: #cfd8e6;
  border-radius: 4px;
  transition: background 0.2s ease;
}

:deep(::-webkit-scrollbar-thumb:hover) {
  background: #a0b3cc;
}

/* TABLE BASE STYLES */
:deep(.elegant-table .v-data-table-header) {
  background-color: #ffffff !important;
}

:deep(.elegant-table .v-data-table__th) {
  border-bottom: 1px solid #eaeef2 !important;
}

:deep(.elegant-table .v-data-table__td) {
  border-bottom: 1px solid #f0f3f7 !important;
}

.font-family-monospace {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
}

.d-flex.flex-wrap {
  display: flex;
  flex-wrap: wrap;
}

.gap-2 {
  gap: 8px;
}

.border-t {
  border-top: 1px solid #eaeef2;
}

:deep(.v-chip) {
  color: inherit !important;
}

:deep(.v-data-table-server__wrapper) {
  overflow-x: auto;
}

/* ELEGANT TOOLTIPS */
:deep(.v-tooltip .v-overlay__content) {
  background-color: #1e293b !important;
  color: white !important;
  font-size: 11px !important;
  padding: 6px 12px !important;
  border-radius: 6px !important;
  font-weight: 400 !important;
  letter-spacing: 0.2px !important;
}
</style>
