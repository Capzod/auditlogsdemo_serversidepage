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
const itemsPerPage = ref(100)

/* FILTERS */
const filters = ref({})
const filterMenu = ref({}) // Track open filter menus

/* =========================
   HEADERS
========================= */
const headers = [
  { title: 'Date & Time', key: 'createdDateTime', width: '180px' },
  { title: 'ID', key: 'id', width: '120px' },
  { title: 'User Principal Name', key: 'userPrincipalName', width: '250px' },
  { title: 'Application', key: 'appDisplayName', width: '200px' },
  { title: 'IP Address', key: 'ipaddress', width: '150px' },
  { title: 'Client', key: 'clientAppUsed', width: '180px' },
  { title: 'Resource', key: 'resourceDisplayName', width: '200px' },
  { title: 'Resource ID', key: 'resourceId', width: '180px' },
  { title: 'User', key: 'userDisplayName', width: '180px' }
]

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

/* =========================
   HELPERS
========================= */
const formatDateTime = (value) => {
  if (!value) return '—'
  const d = new Date(value)
  return d.toLocaleString()
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

const truncateText = (text, length = 20) => {
  if (!text) return '—'
  return text.length > length ? text.substring(0, length) + '...' : text
}

const getInitials = (name) => {
  if (!name) return '??'
  return name
    .split(' ')
    .map(word => word[0])
    .join('')
    .toUpperCase()
    .substring(0, 2)
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

/* =========================
   FETCH DATA
========================= */
const fetchLogs = async () => {
  loading.value = true

  try {
    let response = null

    console.log("PROD FIRST ROW:", items.value[0])

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

    // 🔒 SAFE CHECK
    if (!response || !response.data || !Array.isArray(response.data.data)) {
      console.warn("Invalid API structure:", response)
      items.value = []
      totalItems.value = 0
      return
    }

    items.value = response.data.data
    totalItems.value = response.data.totalCount ?? response.data.data.length

    console.log("FIRST ROW:", response.data.data[0])

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
      <v-card-title class="pa-5 bg-grey-lighten-5">
        <div class="d-flex align-center justify-space-between w-100">
          <div class="d-flex align-center">
            <v-icon icon="mdi-shield-account" color="primary" class="mr-3" />
            <div>
              <h3 class="text-h6 font-weight-medium mb-1">Audit Sign-In Logs</h3>
              <div class="text-caption text-grey-darken-1">
                Server-side pagination · Azure API integration
              </div>
            </div>
          </div>
          <div class="d-flex align-center">
            <v-chip
              color="primary"
              variant="tonal"
              size="small"
              prepend-icon="mdi-server"
              class="mr-3"
            >
              {{ totalItems.toLocaleString() }} records
            </v-chip>
            <v-btn
              variant="outlined"
              density="comfortable"
              prepend-icon="mdi-refresh"
              @click="fetchLogs"
              :loading="loading"
            >
              Refresh
            </v-btn>
          </div>
        </div>
      </v-card-title>

      <!-- ACTIVE FILTERS STATUS BAR -->
      <v-card-text class="pa-4" v-if="hasFilters">
        <div class="d-flex align-center">
          <v-icon icon="mdi-filter-variant" size="18" color="primary" class="mr-2" />
          <span class="text-caption text-grey-darken-1 mr-3">Active filters:</span>
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
                headers.find(h => h.key === filter.key)?.title 
              }}:</span>
              {{ filter.value }}
            </v-chip>
            <v-btn
              size="x-small"
              variant="text"
              @click="clearAllFilters"
              class="ml-2"
            >
              Clear all
            </v-btn>
          </div>
        </div>
      </v-card-text>

      <v-divider />

      <!-- TABLE -->
      <v-data-table-server
        :headers="headers"
        :items="items"
        :items-length="totalItems"
        :loading="loading"
        :page="page"
        :items-per-page="itemsPerPage"
        @update:page="page = $event"
        @update:items-per-page="itemsPerPage = $event"
        class="elevation-0"
        :loading-text="'Loading audit logs...'"
        :no-data-text="'No sign-in logs found'"
      >

        <!-- CUSTOM HEADERS WITH FILTER ICONS -->
        <template #headers="{ columns }">
          <tr class="bg-grey-lighten-4">
            <th v-for="col in columns" :key="col.key" class="pa-3">
              <div class="d-flex align-center justify-space-between">
                <span class="text-uppercase text-caption font-weight-medium text-grey-darken-2">
                  {{ col.title }}
                </span>
                
                <!-- FILTER ICON WITH MENU -->
                <v-menu
                  :close-on-content-click="false"
                  v-model="filterMenu[col.key]"
                  location="bottom"
                >
                  <template #activator="{ props }">
                    <v-btn
                      icon
                      size="x-small"
                      variant="text"
                      v-bind="props"
                      :color="filters[col.key] ? 'primary' : 'grey-darken-1'"
                      density="compact"
                      class="ml-1"
                    >
                      <v-icon size="18">
                        {{ filters[col.key] ? 'mdi-filter' : 'mdi-filter-outline' }}
                      </v-icon>
                    </v-btn>
                  </template>

                  <v-card class="pa-3" width="260">
                    <div class="text-caption font-weight-medium mb-2">
                      Filter by {{ col.title }}
                    </div>
                    <v-text-field
                      v-model="filters[col.key]"
                      density="compact"
                      variant="outlined"
                      hide-details
                      :placeholder="`Search ${col.title.toLowerCase()}...`"
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

        <!-- CUSTOM ROW TEMPLATE -->
        <template #item="{ item }">
          <tr class="hover-row">
            <!-- Date & Time -->
            <td class="pa-3">
              <div class="text-caption font-weight-medium text-grey-darken-3">
                {{ formatDate(item.createdDateTime) }}
              </div>
              <div class="text-caption text-grey-darken-1">
                {{ formatTime(item.createdDateTime) }}
              </div>
            </td>

            <!-- ID -->
            <td class="pa-3">
              <div class="text-caption font-family-monospace text-grey-darken-3">
                {{ truncateText(item.id, 12) }}
              </div>
            </td>

            <!-- User Principal Name -->
            <td class="pa-3">
              <div class="d-flex align-center">
                <v-avatar size="28" color="blue-lighten-5" class="mr-3">
                  <v-icon size="16" color="blue-darken-2">mdi-account</v-icon>
                </v-avatar>
                <div class="text-body-2 text-truncate text-grey-darken-3" style="max-width: 220px">
                  {{ item.userPrincipalName || '—' }}
                </div>
              </div>
            </td>

            <!-- Application -->
            <td class="pa-3">
              <v-chip
                size="small"
                density="comfortable"
                variant="tonal"
                color="indigo"
                label
                class="text-grey-darken-3"
              >
                {{ item.appDisplayName || '—' }}
              </v-chip>
            </td>

            <!-- IP Address -->
            <td class="pa-3">
              <div class="text-caption font-family-monospace text-grey-darken-3">
                {{ item.ipaddress || '—' }}
              </div>
            </td>

            <!-- Client -->
            <td class="pa-3">
              <div class="text-body-2 text-grey-darken-3">
                {{ item.clientAppUsed || '—' }}
              </div>
            </td>

            <!-- Resource -->
            <td class="pa-3">
              <div class="text-body-2 text-grey-darken-3">
                {{ item.resourceDisplayName || '—' }}
              </div>
            </td>

            <!-- Resource ID -->
            <td class="pa-3">
              <div class="text-caption font-family-monospace text-truncate text-grey-darken-3" style="max-width: 160px">
                {{ item.resourceId || '—' }}
              </div>
            </td>

            <!-- User -->
            <td class="pa-3">
              <div class="text-body-2 font-weight-medium text-grey-darken-3">
                {{ item.userDisplayName || '—' }}
              </div>
            </td>
          </tr>
        </template>

        <!-- LOADING STATE -->
        <template #loading>
          <tr>
            <td colspan="9" class="pa-10 text-center">
              <v-progress-circular
                indeterminate
                color="primary"
                size="32"
                width="2"
              />
              <div class="text-caption text-grey-darken-1 mt-4">
                Loading audit logs from Azure API...
                <span v-if="hasFilters" class="d-block">Applying server-side filters</span>
              </div>
            </td>
          </tr>
        </template>

        <!-- NO DATA STATE -->
        <template #no-data>
          <tr>
            <td colspan="9" class="pa-10 text-center">
              <v-icon 
                :icon="hasFilters ? 'mdi-filter-remove' : 'mdi-file-search-outline'" 
                size="48" 
                class="text-grey-lighten-1 mb-4" 
              />
              <h4 class="text-h6 font-weight-regular mb-2 text-grey-darken-2">
                {{ hasFilters ? 'No matching records found' : 'No sign-in logs available' }}
              </h4>
              <p class="text-body-2 text-grey-darken-1 mb-4">
                {{ hasFilters ? 'Try adjusting your filters' : 'Audit data will appear here once available' }}
              </p>
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
            </td>
          </tr>
        </template>

        <!-- TABLE FOOTER -->
        <template #bottom>
          <div class="text-caption text-grey-darken-1 px-4 py-3 border-t d-flex justify-space-between">
            <div>
              <span v-if="hasFilters" class="mr-3">
                <v-icon icon="mdi-filter-check" size="16" class="mr-1" />
                Server-side filtered
              </span>
              Showing {{ ((page - 1) * itemsPerPage) + 1 }} to {{ Math.min(page * itemsPerPage, totalItems) }} 
              of {{ totalItems.toLocaleString() }} records
            </div>
            <div>
              Page {{ page }} of {{ Math.ceil(totalItems / itemsPerPage) }}
            </div>
          </div>
        </template>

      </v-data-table-server>

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
              :total-visible="7"
              density="comfortable"
              color="primary"
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
.hover-row:hover {
  background-color: #f8f9fa;
}

.font-family-monospace {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
}

:deep(.v-data-table-header) {
  background-color: #f5f7fa;
}

:deep(.v-data-table__th) {
  border-bottom: 2px solid #e0e0e0;
  font-weight: 500;
}

.d-flex.flex-wrap {
  display: flex;
  flex-wrap: wrap;
}

.gap-2 {
  gap: 8px;
}

.border-t {
  border-top: 1px solid #e0e0e0;
}

/* Ensure text is visible in chips */
:deep(.v-chip) {
  color: inherit !important;
}
</style>
