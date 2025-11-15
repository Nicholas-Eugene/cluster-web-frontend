<template>
  <div class="silhouette-plot-container">
    <div class="chart-header">
      <div class="chart-title-group">
        <h3>{{ title }}</h3>
        <div class="chart-hint">
          <span class="hint-icon">💡</span>
          <div class="hint-content">
            <strong>Tentang Silhouette Plot:</strong>
            <p>Silhouette plot mengevaluasi kualitas clustering:</p>
            <ul>
              <li><strong>Batang Horizontal:</strong> Setiap batang mewakili satu daerah</li>
              <li><strong>Panjang Batang:</strong> Menunjukkan nilai silhouette (-1 hingga 1)</li>
              <li><strong>Batang Panjang ke Kanan:</strong> Daerah cocok dengan clusternya</li>
              <li><strong>Batang Pendek/Negatif:</strong> Daerah mungkin salah cluster</li>
            </ul>
            <p class="hint-tip">💡 <strong>Cara Membaca:</strong> Semua batang panjang ke kanan = clustering bagus. Banyak batang pendek/negatif = clustering perlu diperbaiki. Garis merah = rata-rata silhouette score.</p>
          </div>
        </div>
      </div>
      <div class="silhouette-info">
        <div class="info-badge">
          <span class="info-icon">ℹ️</span>
          <span class="info-text">Silhouette plot menunjukkan kualitas clustering untuk setiap data point</span>
        </div>
      </div>
    </div>
    
    <div v-if="loading" class="loading-message">
      <div class="spinner"></div>
      <p>Memuat silhouette plot...</p>
    </div>
    
    <div v-else-if="error" class="error-message">
      <p>⚠️ {{ error }}</p>
    </div>
    
    <div v-else-if="!hasValidData" class="no-data-message">
      <p>⚠️ Tidak ada data untuk ditampilkan. Cluster atau members tidak tersedia.</p>
    </div>
    
    <div v-else class="image-wrapper">
      <img 
        :src="imageUrl" 
        :alt="title"
        class="silhouette-plot-image"
        @load="onImageLoad"
        @error="onImageError"
      />
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { API_BASE_URL } from '@/utils/constants'

export default {
  name: 'SilhouettePlot',
  props: {
    clusters: {
      type: Array,
      required: true
    },
    title: {
      type: String,
      default: 'Silhouette Plot - Analisis Kualitas Clustering'
    },
    silhouetteScore: {
      type: Number,
      default: null
    },
    sessionId: {
      type: String,
      required: true
    },
    year: {
      type: [String, Number],
      default: null
    }
  },
  setup(props) {
    const imageUrl = ref('')
    const loading = ref(true)
    const error = ref(null)

    // Check if we have valid data
    const hasValidData = computed(() => {
      if (!props.clusters || props.clusters.length === 0) {
        return false
      }
      
      const hasMembers = props.clusters.some(c => c.members && c.members.length > 0)
      return hasMembers
    })

    const averageSilhouetteScore = computed(() => {
      return props.silhouetteScore
    })

    // Fetch silhouette plot image from backend
    const fetchSilhouettePlot = async () => {
      try {
        loading.value = true
        error.value = null

        if (!props.sessionId) {
          error.value = 'Session ID tidak tersedia'
          loading.value = false
          return
        }

        // Build URL
        let url = `${API_BASE_URL}clustering/silhouette-plot/${props.sessionId}/`
        if (props.year) {
          url += `${props.year}/`
        }

        console.log('🖼️ Fetching silhouette plot from:', url)

        const response = await fetch(url)
        
        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`)
        }

        // Get image blob
        const blob = await response.blob()
        imageUrl.value = URL.createObjectURL(blob)

        console.log('✅ Silhouette plot loaded successfully')
        loading.value = false
      } catch (err) {
        console.error('❌ Error fetching silhouette plot:', err)
        error.value = `Gagal memuat silhouette plot: ${err.message}`
        loading.value = false
      }
    }

    const onImageLoad = () => {
      console.log('✅ Image loaded successfully')
    }

    const onImageError = () => {
      console.error('❌ Image failed to load')
      error.value = 'Gagal memuat gambar silhouette plot'
    }

    // Fetch on mount and when props change
    onMounted(() => {
      if (hasValidData.value) {
        fetchSilhouettePlot()
      } else {
        loading.value = false
      }
    })

    // Watch for changes in sessionId or year
    watch([() => props.sessionId, () => props.year], () => {
      if (hasValidData.value) {
        fetchSilhouettePlot()
      }
    })

    // Clean up object URL on unmount
    watch(() => imageUrl.value, (newUrl, oldUrl) => {
      if (oldUrl && oldUrl.startsWith('blob:')) {
        URL.revokeObjectURL(oldUrl)
      }
    })

    return {
      imageUrl,
      loading,
      error,
      hasValidData,
      averageSilhouetteScore,
      onImageLoad,
      onImageError
    }
  }
}
</script>

<style scoped>
.silhouette-plot-container {
  background: white;
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  margin-bottom: 1.5rem;
}

.chart-header {
  margin-bottom: 1.5rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 1rem;
}

.chart-title-group {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  flex: 1;
}

.chart-header h3 {
  color: #2d3748;
  margin: 0;
  font-size: 1.25rem;
  font-weight: 600;
}

/* Chart Hint Styling */
.chart-hint {
  position: relative;
  display: inline-block;
}

.hint-icon {
  cursor: help;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  font-size: 1rem;
  transition: all 0.3s ease;
  box-shadow: 0 2px 4px rgba(102, 126, 234, 0.3);
}

.hint-icon:hover {
  transform: scale(1.1);
  box-shadow: 0 4px 8px rgba(102, 126, 234, 0.4);
}

.hint-content {
  visibility: hidden;
  opacity: 0;
  position: absolute;
  z-index: 1000;
  bottom: 125%;
  left: 50%;
  transform: translateX(-50%);
  min-width: 400px;
  max-width: 500px;
  background: white;
  color: #2d3748;
  padding: 1.5rem;
  border-radius: 12px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.2);
  transition: opacity 0.3s ease, visibility 0.3s ease;
  border: 2px solid #667eea;
  text-align: left;
}

.hint-content::after {
  content: "";
  position: absolute;
  top: 100%;
  left: 50%;
  margin-left: -10px;
  border-width: 10px;
  border-style: solid;
  border-color: #667eea transparent transparent transparent;
}

.chart-hint:hover .hint-content {
  visibility: visible;
  opacity: 1;
}

.hint-content strong {
  display: block;
  color: #667eea;
  font-size: 1.1rem;
  margin-bottom: 0.75rem;
  font-weight: 700;
}

.hint-content p {
  margin: 0.75rem 0;
  line-height: 1.6;
  color: #4a5568;
}

.hint-content ul {
  margin: 0.75rem 0;
  padding-left: 1.5rem;
  color: #4a5568;
}

.hint-content li {
  margin: 0.5rem 0;
  line-height: 1.6;
}

.hint-content li strong {
  color: #2d3748;
  display: inline;
  font-size: 0.95rem;
}

.hint-tip {
  background: linear-gradient(135deg, #e6f7ff 0%, #f0f9ff 100%);
  border-left: 4px solid #4299e1;
  padding: 0.75rem;
  border-radius: 6px;
  margin-top: 0.75rem;
  font-size: 0.9rem;
}

.hint-tip strong {
  color: #2c5282;
  display: inline;
  font-size: 0.9rem;
}

.silhouette-info {
  display: flex;
  justify-content: flex-start;
  margin-top: 0.5rem;
}

.info-badge {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  background: linear-gradient(135deg, #f0f4ff 0%, #e9f0ff 100%);
  padding: 0.5rem 1rem;
  border-radius: 8px;
  border: 1px solid #cbd5e0;
}

.info-icon {
  font-size: 1.2rem;
}

.info-text {
  color: #4a5568;
  font-size: 0.875rem;
  font-weight: 500;
}

.loading-message,
.error-message,
.no-data-message {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 3rem 1.5rem;
  text-align: center;
  background: #f7fafc;
  border-radius: 8px;
  border: 2px dashed #cbd5e0;
}

.loading-message {
  background: linear-gradient(135deg, #f0f4ff 0%, #e9f0ff 100%);
  border-color: #90cdf4;
}

.error-message {
  background: linear-gradient(135deg, #fff5f5 0%, #fed7d7 100%);
  border-color: #fc8181;
}

.spinner {
  border: 4px solid #e2e8f0;
  border-top: 4px solid #667eea;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
  margin-bottom: 1rem;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.loading-message p,
.error-message p,
.no-data-message p {
  color: #718096;
  font-size: 1rem;
  margin: 0;
}

.error-message p {
  color: #c53030;
  font-weight: 500;
}

.image-wrapper {
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background: #f7fafc;
  border-radius: 8px;
  padding: 1rem;
  min-height: 400px;
}

.silhouette-plot-image {
  max-width: 100%;
  height: auto;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

@media (max-width: 768px) {
  .silhouette-plot-container {
    padding: 1rem;
  }

  .chart-header {
    flex-direction: column;
    align-items: flex-start;
  }

  .image-wrapper {
    min-height: 300px;
    padding: 0.5rem;
  }
  
  .hint-content {
    min-width: 280px;
    max-width: 90vw;
    left: auto;
    right: 0;
    transform: none;
  }
  
  .hint-content::after {
    left: auto;
    right: 20px;
    margin-left: 0;
  }
}
</style>