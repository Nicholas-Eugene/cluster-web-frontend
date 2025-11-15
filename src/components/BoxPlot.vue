<template>
  <div class="box-plot-container">
    <div class="chart-header">
      <div class="chart-title-group">
        <h3>{{ title }}</h3>
        <div class="chart-hint">
          <span class="hint-icon">💡</span>
          <div class="hint-content">
            <strong>Tentang Box Plot:</strong>
            <p>Box plot menampilkan distribusi data untuk setiap cluster:</p>
            <ul>
              <li><strong>Box (Kotak):</strong> Menunjukkan 50% data tengah (Q1 hingga Q3)</li>
              <li><strong>Garis dalam box:</strong> Nilai median (nilai tengah)</li>
              <li><strong>Whiskers (Garis):</strong> Rentang data normal (minimum dan maksimum)</li>
              <li><strong>Titik di luar:</strong> Outlier (data yang tidak biasa)</li>
            </ul>
            <p class="hint-tip">💡 <strong>Cara Membaca:</strong> Box yang lebih tinggi = nilai rata-rata lebih tinggi. Box yang lebih panjang = variasi data lebih besar.</p>
          </div>
        </div>
      </div>
      <div class="chart-controls">
        <select v-model="selectedMetric" @change="updateChart" class="metric-select">
          <option value="ipm">IPM</option>
          <option value="garis_kemiskinan">Garis Kemiskinan</option>
          <option value="pengeluaran_per_kapita">Pengeluaran Per Kapita</option>
        </select>
      </div>
    </div>
    <div class="chart-wrapper">
      <canvas ref="chartCanvas" class="box-plot-canvas"></canvas>
    </div>
    <div class="statistics-summary">
      <div class="stats-grid">
        <div v-for="(cluster, index) in clusters" :key="`cluster-${cluster.id}`" class="stat-card">
          <div class="stat-header">
            <div class="stat-color" :style="{ backgroundColor: getClusterColor(index) }"></div>
            <h4>{{ getClusterLabel(cluster) }}</h4>
          </div>
          <div class="stat-values">
            <div class="stat-item">
              <span class="stat-label">Min:</span>
              <span class="stat-value">{{ formatValue(getStatistics(cluster)[selectedMetric].min) }}</span>
            </div>
            <div class="stat-item">
              <span class="stat-label">Q1:</span>
              <span class="stat-value">{{ formatValue(getStatistics(cluster)[selectedMetric].q1) }}</span>
            </div>
            <div class="stat-item">
              <span class="stat-label">Median:</span>
              <span class="stat-value">{{ formatValue(getStatistics(cluster)[selectedMetric].median) }}</span>
            </div>
            <div class="stat-item">
              <span class="stat-label">Q3:</span>
              <span class="stat-value">{{ formatValue(getStatistics(cluster)[selectedMetric].q3) }}</span>
            </div>
            <div class="stat-item">
              <span class="stat-label">Max:</span>
              <span class="stat-value">{{ formatValue(getStatistics(cluster)[selectedMetric].max) }}</span>
            </div>
            <div class="stat-item">
              <span class="stat-label">Mean:</span>
              <span class="stat-value">{{ formatValue(getStatistics(cluster)[selectedMetric].mean) }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted, watch, nextTick } from 'vue'
import Chart from 'chart.js/auto'
import { BoxPlotController, BoxAndWiskers } from '@sgratzl/chartjs-chart-boxplot'

// Register the box plot controller
Chart.register(BoxPlotController, BoxAndWiskers)

export default {
  name: 'BoxPlot',
  props: {
    clusters: {
      type: Array,
      required: true
    },
    title: {
      type: String,
      default: 'Box and Whisker Plot - Analisis Distribusi per Cluster'
    }
  },
  setup(props) {
    const chartCanvas = ref(null)
    const chart = ref(null)
    const selectedMetric = ref('ipm')

    // Consistent cluster colors matching the design system
    const colors = [
      '#667eea', '#48bb78', '#ed8936', '#4299e1', '#f56565',
      '#38b2ac', '#9f7aea', '#ecc94b', '#f687b3', '#4fd1c5',
    ]

    const getClusterColor = (index) => colors[index % colors.length]
    
  const getClusterLabel = (cluster) => {
    if (!cluster) return 'Unknown'
    if (cluster.id === -1 || cluster.id === '-1') return 'Noise (Outliers)'
    return `Cluster ${cluster.id}`
  }

    const formatValue = (value) => {
      if (selectedMetric.value === 'garis_kemiskinan' || selectedMetric.value === 'pengeluaran_per_kapita') {
        return new Intl.NumberFormat('id-ID', {
          style: 'currency',
          currency: 'IDR',
          minimumFractionDigits: 0
        }).format(value)
      }
      return value.toFixed(2)
    }

    const calculateStatistics = (values) => {
      const sorted = [...values].sort((a, b) => a - b)
      const n = sorted.length
      
      const min = sorted[0]
      const max = sorted[n - 1]
      const median = n % 2 === 0 
        ? (sorted[n/2 - 1] + sorted[n/2]) / 2 
        : sorted[Math.floor(n/2)]
      
      const q1Index = Math.floor(n * 0.25)
      const q3Index = Math.floor(n * 0.75)
      const q1 = sorted[q1Index]
      const q3 = sorted[q3Index]
      
      const mean = values.reduce((sum, val) => sum + val, 0) / n
      
      return { min, q1, median, q3, max, mean }
    }

    const getStatistics = (cluster) => {
      const metrics = ['ipm', 'garis_kemiskinan', 'pengeluaran_per_kapita']
      const stats = {}
      
      metrics.forEach(metric => {
        const values = cluster.members.map(member => member[metric]).filter(val => val != null)
        stats[metric] = calculateStatistics(values)
      })
      
      return stats
    }

    const createChart = async () => {
      try {
        if (!chartCanvas.value) return
        if (!props.clusters || props.clusters.length === 0) return

        await nextTick()
        if (!chartCanvas.value) return
        
        if (chart.value) {
          try {
            if (typeof chart.value.destroy === 'function') {
              chart.value.destroy()
            }
          } catch (e) {
            console.warn('Error destroying existing chart:', e)
          }
          chart.value = null
        }
        
        await new Promise(resolve => setTimeout(resolve, 50))
        if (!chartCanvas.value) return
        
        let ctx
        try {
          ctx = chartCanvas.value.getContext('2d')
        } catch (e) {
          console.error('Failed to get 2d context:', e)
          return
        }
        
        if (!ctx || !ctx.canvas) return
        
        const canvas = ctx.canvas
        if (canvas.width === 0 || canvas.height === 0) {
          setTimeout(() => createChart(), 200)
          return
        }
        
        if (!document.contains(canvas)) return
      
      const labels = props.clusters.map(cluster => `Cluster ${cluster.id}`)
      const datasets = [{
        label: 'Clusters',
        data: props.clusters.map((cluster, index) => {
          const values = cluster.members
            .map(member => member[selectedMetric.value])
            .filter(val => val != null)
          return values
        }),
        backgroundColor: props.clusters.map((cluster, index) => getClusterColor(index) + 'B3'),
        borderColor: props.clusters.map((cluster, index) => getClusterColor(index)),
        borderWidth: 2,
        outlierColor: props.clusters.map((cluster, index) => getClusterColor(index)),
        outlierBackgroundColor: props.clusters.map((cluster, index) => getClusterColor(index) + '80'),
        outlierBorderColor: props.clusters.map((cluster, index) => getClusterColor(index)),
        outlierRadius: 4,
        itemRadius: 0
      }]

      const config = {
        type: 'boxplot',
        data: { labels, datasets },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          animation: { duration: 0 },
          hover: { animationDuration: 0 },
          responsiveAnimationDuration: 0,
          plugins: {
            title: {
              display: true,
              text: `Box and Whisker Plot - ${formatAxisLabel(selectedMetric.value)} per Cluster`,
              font: { size: 14, weight: 'bold' }
            },
            legend: { display: false },
            tooltip: {
              callbacks: {
                label: (context) => {
                  const dataIndex = context.dataIndex
                  const cluster = props.clusters[dataIndex]
                  const values = cluster.members
                    .map(member => member[selectedMetric.value])
                    .filter(val => val != null)
                  const stats = calculateStatistics(values)
                  
                  return [
                    `Cluster ${cluster.id}`,
                    `Min: ${formatValue(stats.min)}`,
                    `Q1: ${formatValue(stats.q1)}`,
                    `Median: ${formatValue(stats.median)}`,
                    `Q3: ${formatValue(stats.q3)}`,
                    `Max: ${formatValue(stats.max)}`,
                    `Count: ${values.length}`
                  ]
                }
              }
            }
          },
          scales: {
            x: { title: { display: true, text: 'Cluster' } },
            y: {
              title: { display: true, text: formatAxisLabel(selectedMetric.value) },
              ticks: { callback: function(value) { return formatValue(value) } }
            }
          }
        }
      }

      chart.value = new Chart(ctx, config)
      
      } catch (error) {
        console.warn('Error creating box plot chart:', error)
        if (chart.value) {
          try {
            if (typeof chart.value.destroy === 'function') {
              chart.value.destroy()
            }
          } catch (destroyError) {
            console.warn('Error destroying chart after creation error:', destroyError)
          }
          chart.value = null
        }
      }
    }

    const formatAxisLabel = (metric) => {
      const labels = {
        'ipm': 'IPM',
        'garis_kemiskinan': 'Garis Kemiskinan (Rp)',
        'pengeluaran_per_kapita': 'Pengeluaran Per Kapita (Rp)'
      }
      return labels[metric] || metric
    }

    let updateTimeout = null
    let isUpdating = false
    
    const updateChart = () => {
      if (isUpdating) return
      if (updateTimeout) clearTimeout(updateTimeout)
      
      updateTimeout = setTimeout(async () => {
        if (!chartCanvas.value) return
        isUpdating = true
        try {
          await createChart()
        } catch (error) {
          console.error('Error during chart update:', error)
        } finally {
          isUpdating = false
        }
      }, 300)
    }

    onMounted(async () => {
      await createChart()
    })

    onUnmounted(() => {
      if (updateTimeout) {
        clearTimeout(updateTimeout)
        updateTimeout = null
      }
      isUpdating = true
      if (chart.value) {
        try {
          if (typeof chart.value.destroy === 'function') {
            chart.value.destroy()
          }
        } catch (e) {
          console.warn('Error destroying chart on unmount:', e)
        }
        chart.value = null
      }
      chartCanvas.value = null
    })

    watch(() => props.clusters, () => {
      updateChart()
    }, { deep: true })

    return {
      chartCanvas,
      selectedMetric,
      getClusterColor,
      getClusterLabel,
      getStatistics,
      formatValue,
      updateChart
    }
  }
}
</script>

<style scoped>
.box-plot-container {
  background: white;
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  margin-bottom: 2rem;
}

.chart-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
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

.metric-select {
  padding: 0.5rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  background: white;
  color: #4a5568;
  font-size: 0.875rem;
}

.chart-wrapper {
  height: 400px;
  position: relative;
}

.statistics-summary {
  margin-top: 2rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
}

.stat-card {
  background: #f7fafc;
  border-radius: 8px;
  padding: 1rem;
  border: 1px solid #e2e8f0;
}

.stat-header {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.stat-color {
  width: 16px;
  height: 16px;
  border-radius: 50%;
}

.stat-header h4 {
  color: #2d3748;
  margin: 0;
  font-size: 1rem;
}

.stat-values {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 0.5rem;
}

.stat-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.25rem 0;
}

.stat-label {
  font-weight: 600;
  color: #4a5568;
  font-size: 0.875rem;
}

.stat-value {
  color: #2d3748;
  font-size: 0.875rem;
  text-align: right;
}

@media (max-width: 768px) {
  .chart-header {
    flex-direction: column;
    align-items: stretch;
  }
  
  .chart-wrapper {
    height: 300px;
  }
  
  .stats-grid {
    grid-template-columns: 1fr;
  }
  
  .stat-values {
    grid-template-columns: 1fr;
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