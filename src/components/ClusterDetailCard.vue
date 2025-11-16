<template>
  <div class="cluster-detail-card card">
    <h3>🔍 Detail Cluster</h3>
    <div class="cluster-tabs">
      <button 
        v-for="(cluster, index) in clusters" 
        :key="`cluster-${index}`"
        @click="selectCluster(cluster.id)"
        :class="['cluster-tab', { active: isClusterActive(cluster.id) }]"
        :style="{ borderColor: getClusterColor(index) }"
      >
        <div class="tab-color" :style="{ backgroundColor: getClusterColor(index) }"></div>
        Cluster {{ cluster.id }} ({{ cluster.size }})
      </button>
    </div>
    
    <div v-if="activeCluster" class="cluster-detail">
      <div class="cluster-info">
        <h4>{{ getClusterLabel(activeCluster) }}</h4>
        <div class="cluster-stats">
          <div class="stat-item">
            <span class="stat-label">Jumlah Daerah:</span>
            <span class="stat-value">{{ activeCluster.size }}</span>
          </div>
          <div v-if="activeCluster.centroid" class="centroid-info">
            <h5>Centroid (Rata-rata Cluster):</h5>
            <div class="centroid-values">
              <div class="centroid-item">
                <span>IPM:</span>
                <span>{{ activeCluster.centroid.ipm?.toFixed(2) || 'N/A' }}</span>
              </div>
              <div class="centroid-item">
                <span>Garis Kemiskinan:</span>
                <span>{{ formatCurrency(activeCluster.centroid.garis_kemiskinan) }} / bulan</span>
              </div>
              <div class="centroid-item">
                <span>Pengeluaran Per Kapita:</span>
                <span>{{ formatCurrency(activeCluster.centroid.pengeluaran_per_kapita * 1000) }} /tahun</span>
              </div>
            </div>
          </div>
        </div>
      </div>
      
      <div class="cluster-members">
        <div class="members-header">
          <h5>Daftar Daerah ({{ filteredMembers.length }} dari {{ activeCluster.members.length }}):</h5>
          <div class="members-controls">
            <div class="search-box">
              <span class="search-icon">🔍</span>
              <input 
                v-model="searchQuery" 
                type="text" 
                placeholder="Cari daerah..." 
                class="search-input"
              />
              <button 
                v-if="searchQuery" 
                @click="searchQuery = ''" 
                class="clear-search"
                title="Hapus pencarian"
              >
                ✕
              </button>
            </div>
            
            <!-- NEW: Province Filter -->
            <div class="province-filter-box">
              <label for="province-filter">Filter Provinsi:</label>
              <select v-model="selectedProvince" id="province-filter" class="province-select">
                <option value="all">Semua Provinsi</option>
                <option v-for="province in availableProvinces" :key="province" :value="province">
                  {{ province }}
                </option>
              </select>
            </div>
            
            <div class="sort-box">
              <label for="sort-select">Urutkan:</label>
              <select v-model="sortBy" id="sort-select" class="sort-select">
                <option value="name-asc">Nama (A-Z)</option>
                <option value="name-desc">Nama (Z-A)</option>
                <option value="province-asc">Provinsi (A-Z)</option>
                <option value="province-desc">Provinsi (Z-A)</option>
                <option value="ipm-asc">IPM (Terendah)</option>
                <option value="ipm-desc">IPM (Tertinggi)</option>
                <option value="poverty-asc">Kemiskinan (Terendah)</option>
                <option value="poverty-desc">Kemiskinan (Tertinggi)</option>
                <option value="expenditure-asc">Pengeluaran (Terendah)</option>
                <option value="expenditure-desc">Pengeluaran (Tertinggi)</option>
                <option v-if="showMembership" value="membership-desc">Membership (Tertinggi)</option>
                <option v-if="showMembership" value="membership-asc">Membership (Terendah)</option>
              </select>
            </div>
            <div class="items-per-page-box">
              <label for="items-per-page">Tampilkan:</label>
              <select v-model.number="itemsPerPage" id="items-per-page" class="items-per-page-select">
                <option :value="10">10</option>
                <option :value="25">25</option>
                <option :value="50">50</option>
                <option :value="100">100</option>
                <option :value="-1">Semua</option>
              </select>
            </div>
          </div>
        </div>
        
        <div v-if="filteredMembers.length === 0" class="no-results">
          <p>🔍 Tidak ada daerah yang cocok dengan filter</p>
          <button @click="resetFilters" class="btn-reset-filter">Reset Filter</button>
        </div>
        
        <div v-else class="table-container">
          <table class="members-table">
            <thead>
              <tr>
                <th class="col-no">No</th>
                <th class="col-name">Kabupaten/Kota</th>
                <th class="col-province">Provinsi</th>
                <th class="col-ipm">IPM</th>
                <th class="col-poverty">Garis Kemiskinan</th>
                <th class="col-expenditure">Pengeluaran/Kapita</th>
                <th v-if="showMembership" class="col-membership">Membership</th>
              </tr>
            </thead>
            <tbody>
              <tr 
                v-for="(member, index) in paginatedMembers" 
                :key="`${member.kabupaten_kota}-${member.tahun || 'all'}`"
              >
                <td class="col-no">{{ (currentPage - 1) * itemsPerPage + index + 1 }}</td>
                <td class="col-name">
                  <strong>{{ member.kabupaten_kota }}</strong>
                </td>
                <td class="col-province">
                  <span class="province-badge">{{ member.provinsi || 'N/A' }}</span>
                </td>
                <td class="col-ipm">{{ member.ipm?.toFixed(2) || 'N/A' }}</td>
                <td class="col-poverty">{{ formatCurrency(member.garis_kemiskinan) }}</td>
                <td class="col-expenditure">{{ formatCurrency(member.pengeluaran_per_kapita * 1000) }}</td>
                <td v-if="showMembership" class="col-membership">
                  <div v-if="member.membership != null" class="membership-cell">
                    <div class="membership-bar">
                      <div 
                        class="membership-fill" 
                        :style="{ 
                          width: `${member.membership * 100}%`,
                          backgroundColor: getClusterColor(clusters.findIndex(c => normalizeId(c.id) === normalizeId(activeCluster.id)))
                        }"
                      ></div>
                    </div>
                    <span class="membership-value">{{ (member.membership * 100).toFixed(1) }}%</span>
                  </div>
                  <span v-else>-</span>
                </td>
              </tr>
            </tbody>
          </table>

          <!-- Pagination Controls -->
          <div class="pagination-container">
            <div class="pagination-info">
              {{ showingRange }}
            </div>
            <div class="pagination-controls">
              <button 
                @click="goToPage(1)" 
                :disabled="currentPage === 1"
                class="pagination-btn"
                title="Halaman pertama"
              >
                ⟪
              </button>
              <button 
                @click="prevPage" 
                :disabled="currentPage === 1"
                class="pagination-btn"
                title="Halaman sebelumnya"
              >
                ‹
              </button>
              
              <div class="page-numbers">
                <template v-for="page in getPageNumbers()" :key="page">
                  <button 
                    v-if="page !== '...'"
                    @click="goToPage(page)" 
                    :class="['page-btn', { active: currentPage === page }]"
                  >
                    {{ page }}
                  </button>
                  <span v-else class="page-ellipsis">...</span>
                </template>
              </div>

              <button 
                @click="nextPage" 
                :disabled="currentPage === totalPages"
                class="pagination-btn"
                title="Halaman berikutnya"
              >
                ›
              </button>
              <button 
                @click="goToPage(totalPages)" 
                :disabled="currentPage === totalPages"
                class="pagination-btn"
                title="Halaman terakhir"
              >
                ⟫
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch } from 'vue'

export default {
  name: 'ClusterDetailCard',
  props: {
    clusters: {
      type: Array,
      required: true
    },
    showMembership: {
      type: Boolean,
      default: true
    }
  },
  setup(props) {
    const selectedClusterId = ref(null)
    const searchQuery = ref('')
    const selectedProvince = ref('all')
    const sortBy = ref('name-asc')
    const currentPage = ref(1)
    const itemsPerPage = ref(25)

    // Consistent cluster colors matching the design system
    const colors = [
      '#667eea', '#48bb78', '#ed8936', '#4299e1', '#f56565',
      '#38b2ac', '#9f7aea', '#ecc94b', '#f687b3', '#4fd1c5',
    ]

    const getClusterColor = (index) => {
      return colors[index % colors.length]
    }

    // Helper to normalize cluster ID
    const normalizeId = (id) => {
      if (id === null || id === undefined) return null
      const num = Number(id)
      return isNaN(num) ? id : num
    }
    
    // Get display label for cluster
    const getClusterLabel = (cluster) => {
      if (!cluster) return 'Unknown'
      if (cluster.id === -1 || cluster.id === '-1') {
        return '🔸 Noise (Outliers)'
      }
      return `Cluster ${cluster.id}`
    }

    // Check if cluster is active
    const isClusterActive = (clusterId) => {
      const normalizedCluster = normalizeId(clusterId)
      const normalizedSelected = normalizeId(selectedClusterId.value)
      return normalizedCluster === normalizedSelected
    }

    // Select cluster method
    const selectCluster = (clusterId) => {
      selectedClusterId.value = normalizeId(clusterId)
      // Reset filters when changing cluster
      selectedProvince.value = 'all'
      searchQuery.value = ''
      currentPage.value = 1
    }

    const activeCluster = computed(() => {
      if (selectedClusterId.value === null || selectedClusterId.value === undefined) {
        return null
      }
      
      if (!props.clusters || props.clusters.length === 0) {
        return null
      }
      
      const normalizedSelected = normalizeId(selectedClusterId.value)
      const found = props.clusters.find(cluster => {
        return normalizeId(cluster.id) === normalizedSelected
      })
      
      return found || null
    })

    // Get available provinces from active cluster
    const availableProvinces = computed(() => {
      if (!activeCluster.value || !activeCluster.value.members) {
        return []
      }

      const provinces = new Set()
      activeCluster.value.members.forEach(member => {
        if (member.provinsi && member.provinsi.trim()) {
          provinces.add(member.provinsi.trim())
        }
      })

      return Array.from(provinces).sort()
    })

    // Filtered and sorted members
    const filteredMembers = computed(() => {
      if (!activeCluster.value || !activeCluster.value.members) {
        return []
      }

      let members = [...activeCluster.value.members]

      // Apply search filter
      if (searchQuery.value.trim()) {
        const query = searchQuery.value.toLowerCase().trim()
        members = members.filter(member => {
          const cityName = (member.kabupaten_kota || '').toLowerCase()
          const provinceName = (member.provinsi || '').toLowerCase()
          return cityName.includes(query) || provinceName.includes(query)
        })
      }

      // Apply province filter
      if (selectedProvince.value !== 'all') {
        members = members.filter(member => {
          return member.provinsi && member.provinsi.trim() === selectedProvince.value
        })
      }

      // Apply sorting
      members.sort((a, b) => {
        switch (sortBy.value) {
          case 'name-asc':
            return (a.kabupaten_kota || '').localeCompare(b.kabupaten_kota || '')
          case 'name-desc':
            return (b.kabupaten_kota || '').localeCompare(a.kabupaten_kota || '')
          case 'province-asc':
            return (a.provinsi || '').localeCompare(b.provinsi || '')
          case 'province-desc':
            return (b.provinsi || '').localeCompare(a.provinsi || '')
          case 'ipm-asc':
            return (a.ipm || 0) - (b.ipm || 0)
          case 'ipm-desc':
            return (b.ipm || 0) - (a.ipm || 0)
          case 'poverty-asc':
            return (a.garis_kemiskinan || 0) - (b.garis_kemiskinan || 0)
          case 'poverty-desc':
            return (b.garis_kemiskinan || 0) - (a.garis_kemiskinan || 0)
          case 'expenditure-asc':
            return (a.pengeluaran_per_kapita || 0) - (b.pengeluaran_per_kapita || 0)
          case 'expenditure-desc':
            return (b.pengeluaran_per_kapita || 0) - (a.pengeluaran_per_kapita || 0)
          case 'membership-asc':
            return (a.membership || 0) - (b.membership || 0)
          case 'membership-desc':
            return (b.membership || 0) - (a.membership || 0)
          default:
            return 0
        }
      })

      return members
    })

    // Paginated members
    const paginatedMembers = computed(() => {
      const filtered = filteredMembers.value
      
      // Reset to page 1 if current page is out of bounds
      if (currentPage.value > totalPages.value && totalPages.value > 0) {
        currentPage.value = 1
      }

      // If "All" is selected, return all filtered members
      if (itemsPerPage.value === -1) {
        return filtered
      }

      const start = (currentPage.value - 1) * itemsPerPage.value
      const end = start + itemsPerPage.value
      return filtered.slice(start, end)
    })

    // Total pages
    const totalPages = computed(() => {
      if (itemsPerPage.value === -1) return 1
      return Math.ceil(filteredMembers.value.length / itemsPerPage.value)
    })

    // Pagination helpers
    const goToPage = (page) => {
      if (page >= 1 && page <= totalPages.value) {
        currentPage.value = page
      }
    }

    const nextPage = () => {
      if (currentPage.value < totalPages.value) {
        currentPage.value++
      }
    }

    const prevPage = () => {
      if (currentPage.value > 1) {
        currentPage.value--
      }
    }

    // Reset page when search, province filter, or sort changes
    watch([searchQuery, selectedProvince, sortBy], () => {
      currentPage.value = 1
    })

    // Reset filters
    const resetFilters = () => {
      searchQuery.value = ''
      selectedProvince.value = 'all'
      sortBy.value = 'name-asc'
      currentPage.value = 1
    }

    // Showing range text
    const showingRange = computed(() => {
      if (filteredMembers.value.length === 0) return 'Tidak ada data'
      if (itemsPerPage.value === -1) {
        return `Menampilkan semua ${filteredMembers.value.length} daerah`
      }
      const start = (currentPage.value - 1) * itemsPerPage.value + 1
      const end = Math.min(start + itemsPerPage.value - 1, filteredMembers.value.length)
      return `Menampilkan ${start}-${end} dari ${filteredMembers.value.length} daerah`
    })

    // Get page numbers for pagination with ellipsis
    const getPageNumbers = () => {
      const pages = []
      const total = totalPages.value
      const current = currentPage.value

      if (total <= 7) {
        for (let i = 1; i <= total; i++) {
          pages.push(i)
        }
      } else {
        pages.push(1)

        if (current > 3) {
          pages.push('...')
        }

        const start = Math.max(2, current - 1)
        const end = Math.min(total - 1, current + 1)

        for (let i = start; i <= end; i++) {
          pages.push(i)
        }

        if (current < total - 2) {
          pages.push('...')
        }

        pages.push(total)
      }

      return pages
    }

    const formatCurrency = (value) => {
      if (!value) return 'N/A'
      return new Intl.NumberFormat('id-ID', {
        style: 'currency',
        currency: 'IDR',
        minimumFractionDigits: 0
      }).format(value)
    }

    // Initialize with first cluster
    watch(() => props.clusters, (newClusters) => {
      if (newClusters && newClusters.length > 0) {
        selectedClusterId.value = normalizeId(newClusters[0].id)
      }
    }, { immediate: true })

    return {
      selectedClusterId,
      activeCluster,
      searchQuery,
      selectedProvince,
      availableProvinces,
      sortBy,
      filteredMembers,
      paginatedMembers,
      currentPage,
      itemsPerPage,
      totalPages,
      showingRange,
      goToPage,
      nextPage,
      prevPage,
      getPageNumbers,
      getClusterColor,
      formatCurrency,
      selectCluster,
      isClusterActive,
      normalizeId,
      getClusterLabel,
      resetFilters
    }
  }
}
</script>

<style scoped>
.cluster-detail-card {
  margin-bottom: 2rem;
}

.cluster-tabs {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
  margin-bottom: 2rem;
}

.cluster-tab {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 1.5rem;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  background: white;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
}

.cluster-tab:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.cluster-tab.active {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-color: #667eea;
}

.tab-color {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  border: 2px solid white;
}

.cluster-detail {
  display: grid;
  gap: 2rem;
}

.cluster-info h4 {
  font-size: 1.5rem;
  color: #2d3748;
  margin-bottom: 1.5rem;
}

.cluster-stats {
  background: #f7fafc;
  padding: 1.5rem;
  border-radius: 8px;
  border-left: 4px solid #667eea;
}

.stat-item {
  display: flex;
  justify-content: space-between;
  padding: 0.5rem 0;
  margin-bottom: 1rem;
}

.stat-label {
  font-weight: 600;
  color: #4a5568;
}

.stat-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #667eea;
}

.centroid-info {
  margin-top: 1.5rem;
}

.centroid-info h5 {
  font-size: 1.1rem;
  color: #2d3748;
  margin-bottom: 1rem;
}

.centroid-values {
  display: grid;
  gap: 0.75rem;
}

.centroid-item {
  display: flex;
  justify-content: space-between;
  padding: 0.75rem 1rem;
  background: white;
  border-radius: 6px;
  border-left: 3px solid #667eea;
}

.centroid-item span:first-child {
  font-weight: 600;
  color: #4a5568;
}

.centroid-item span:last-child {
  font-weight: 700;
  color: #2d3748;
}

.members-header {
  margin-bottom: 1.5rem;
}

.members-header h5 {
  font-size: 1.2rem;
  color: #2d3748;
  margin-bottom: 1rem;
}

.members-controls {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
  align-items: center;
}

.search-box {
  position: relative;
  flex: 1;
  min-width: 250px;
}

.search-icon {
  position: absolute;
  left: 1rem;
  top: 50%;
  transform: translateY(-50%);
  font-size: 1rem;
  color: #718096;
  pointer-events: none;
}

.search-input {
  width: 100%;
  padding: 0.75rem 2.5rem 0.75rem 2.75rem;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.95rem;
  transition: all 0.3s ease;
  background: white;
}

.search-input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.search-input::placeholder {
  color: #a0aec0;
}

.clear-search {
  position: absolute;
  right: 0.75rem;
  top: 50%;
  transform: translateY(-50%);
  background: transparent;
  border: none;
  color: #718096;
  font-size: 1.25rem;
  cursor: pointer;
  padding: 0.25rem;
  border-radius: 4px;
  transition: all 0.2s ease;
  line-height: 1;
}

.clear-search:hover {
  background: #f7fafc;
  color: #e53e3e;
}

/* NEW: Province Filter Styles */
.province-filter-box {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  min-width: 200px;
}

.province-filter-box label {
  font-weight: 600;
  color: #4a5568;
  font-size: 0.95rem;
  white-space: nowrap;
}

.province-select {
  padding: 0.75rem 1rem;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.95rem;
  background: white;
  cursor: pointer;
  transition: all 0.3s ease;
  min-width: 180px;
  font-weight: 500;
}

.province-select:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.province-select:hover {
  border-color: #cbd5e0;
}

.sort-box {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.sort-box label {
  font-weight: 600;
  color: #4a5568;
  font-size: 0.95rem;
  white-space: nowrap;
}

.sort-select {
  padding: 0.75rem 1rem;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.95rem;
  background: white;
  cursor: pointer;
  transition: all 0.3s ease;
  min-width: 200px;
}

.sort-select:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.sort-select:hover {
  border-color: #cbd5e0;
}

.items-per-page-box {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.items-per-page-box label {
  font-weight: 600;
  color: #4a5568;
  font-size: 0.95rem;
  white-space: nowrap;
}

.items-per-page-select {
  padding: 0.75rem 1rem;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.95rem;
  background: white;
  cursor: pointer;
  transition: all 0.3s ease;
  min-width: 100px;
}

.items-per-page-select:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.items-per-page-select:hover {
  border-color: #cbd5e0;
}

.no-results {
  text-align: center;
  padding: 3rem 1.5rem;
  background: #f7fafc;
  border-radius: 8px;
  border: 2px dashed #cbd5e0;
}

.no-results p {
  color: #718096;
  font-size: 1rem;
  margin: 0 0 1rem 0;
}

.btn-reset-filter {
  padding: 0.5rem 1.5rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
}

.btn-reset-filter:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(102, 126, 234, 0.3);
}

/* Table Styles */
.table-container {
  background: white;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.members-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.95rem;
}

.members-table thead {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.members-table th {
  padding: 1rem;
  text-align: left;
  font-weight: 600;
  font-size: 0.9rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.members-table tbody tr {
  border-bottom: 1px solid #e2e8f0;
  transition: background-color 0.2s ease;
}

.members-table tbody tr:hover {
  background-color: #f7fafc;
}

.members-table tbody tr:last-child {
  border-bottom: none;
}

.members-table td {
  padding: 1rem;
}

.col-no {
  width: 60px;
  text-align: center;
  color: #718096;
  font-weight: 500;
}

.col-name {
  min-width: 200px;
}

.col-name strong {
  color: #2d3748;
  font-weight: 600;
}

.col-province {
  min-width: 150px;
}

.province-badge {
  display: inline-block;
  padding: 0.25rem 0.75rem;
  background: linear-gradient(135deg, #e6f7ff 0%, #f0f9ff 100%);
  color: #2c5282;
  border-radius: 12px;
  font-size: 0.85rem;
  font-weight: 600;
  border: 1px solid #90cdf4;
}

.col-ipm {
  width: 100px;
  text-align: center;
  font-weight: 600;
}

.col-poverty,
.col-expenditure {
  min-width: 140px;
  text-align: right;
  font-weight: 500;
}

.col-membership {
  min-width: 180px;
}

.membership-cell {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.membership-bar {
  flex: 1;
  height: 20px;
  background: #e2e8f0;
  border-radius: 10px;
  overflow: hidden;
  position: relative;
}

.membership-fill {
  height: 100%;
  transition: width 0.3s ease;
  background: linear-gradient(90deg, #667eea 0%, #764ba2 100%);
}

.membership-value {
  font-weight: 600;
  color: #667eea;
  font-size: 0.875rem;
  min-width: 50px;
  text-align: right;
}

/* Pagination Styles */
.pagination-container {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1.5rem;
  background: #f7fafc;
  border-top: 1px solid #e2e8f0;
  flex-wrap: wrap;
  gap: 1rem;
}

.pagination-info {
  color: #4a5568;
  font-weight: 500;
  font-size: 0.95rem;
}

.pagination-controls {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.pagination-btn {
  padding: 0.5rem 0.75rem;
  border: 2px solid #e2e8f0;
  background: white;
  color: #4a5568;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
  font-weight: 600;
  font-size: 1.1rem;
}

.pagination-btn:hover:not(:disabled) {
  background: #667eea;
  color: white;
  border-color: #667eea;
}

.pagination-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.page-numbers {
  display: flex;
  gap: 0.25rem;
  align-items: center;
}

.page-btn {
  padding: 0.5rem 0.75rem;
  min-width: 40px;
  border: 2px solid #e2e8f0;
  background: white;
  color: #4a5568;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
  font-weight: 500;
  font-size: 0.95rem;
}

.page-btn:hover {
  background: #f7fafc;
  border-color: #667eea;
}

.page-btn.active {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-color: #667eea;
  font-weight: 600;
}

.page-ellipsis {
  padding: 0.5rem 0.25rem;
  color: #a0aec0;
  font-weight: 600;
}

@media (max-width: 768px) {
  .cluster-tabs {
    flex-direction: column;
  }
  
  .cluster-tab {
    width: 100%;
  }

  .members-controls {
    flex-direction: column;
    align-items: stretch;
  }

  .search-box,
  .province-filter-box,
  .sort-box,
  .items-per-page-box {
    min-width: 100%;
  }

  .province-filter-box,
  .sort-box,
  .items-per-page-box {
    flex-direction: column;
    align-items: stretch;
  }

  .province-filter-box label,
  .sort-box label,
  .items-per-page-box label {
    font-size: 0.875rem;
  }

  .province-select,
  .sort-select,
  .items-per-page-select {
    width: 100%;
  }

  /* Mobile table - make it scrollable */
  .table-container {
    overflow-x: auto;
  }

  .members-table {
    min-width: 900px;
  }

  .members-table th,
  .members-table td {
    padding: 0.75rem 0.5rem;
    font-size: 0.85rem;
  }

  .pagination-container {
    flex-direction: column;
    align-items: center;
    gap: 1rem;
  }

  .pagination-controls {
    flex-wrap: wrap;
    justify-content: center;
  }

  .page-numbers {
    flex-wrap: wrap;
    justify-content: center;
  }
}
</style>