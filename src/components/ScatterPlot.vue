<template>
  <div class="scatter-plot-container">
    <div class="chart-header">
      <div class="chart-title-group">
        <h3>{{ title }}</h3>
        <div class="chart-hint">
          <span class="hint-icon">💡</span>
          <div class="hint-content">
            <strong>Tentang Scatter Plot:</strong>
            <p>Scatter plot menampilkan hubungan antara dua variabel untuk setiap daerah:</p>
            <ul>
              <li><strong>Titik-titik:</strong> Setiap titik mewakili satu daerah</li>
              <li><strong>Warna:</strong> Menunjukkan cluster/kelompok daerah tersebut</li>
              <li><strong>Posisi:</strong> Menunjukkan nilai dari dua variabel yang dipilih</li>
            </ul>
            <p class="hint-tip">💡 <strong>Cara Membaca:</strong> Daerah dengan warna sama memiliki karakteristik serupa. Titik yang berdekatan menunjukkan nilai yang mirip.</p>
          </div>
        </div>
      </div>
      <div class="chart-controls">
        <select v-model="selectedXAxis" @change="updateChart" class="axis-select">
          <option value="ipm">IPM</option>
          <option value="garis_kemiskinan">Garis Kemiskinan</option>
          <option value="pengeluaran_per_kapita">Pengeluaran Per Kapita</option>
        </select>
        <span>vs</span>
        <select v-model="selectedYAxis" @change="updateChart" class="axis-select">
          <option value="ipm">IPM</option>
          <option value="garis_kemiskinan">Garis Kemiskinan</option>
          <option value="pengeluaran_per_kapita">Pengeluaran Per Kapita</option>
        </select>
      </div>
    </div>
    <div class="chart-wrapper">
      <canvas ref="chartCanvas" class="scatter-plot-canvas"></canvas>
    </div>
    <div class="chart-legend">
      <div v-for="(cluster, index) in clusters" :key="`legend-${cluster.id}`" class="legend-item">
        <div class="legend-color" :style="{ backgroundColor: getClusterColor(index) }"></div>
        <span>{{ getClusterLabel(cluster) }} ({{ cluster.size }} daerah)</span>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted, watch, nextTick } from 'vue'
import Chart from 'chart.js/auto'

export default {
  name: 'ScatterPlot',
  props: {
    clusters: {
      type: Array,
      required: true
    },
    title: {
      type: String,
      default: 'Scatter Plot Clustering'
    }
  },
  setup(props) {
    const chartCanvas = ref(null)
    const chart = ref(null)
    const selectedXAxis = ref('ipm')
    const selectedYAxis = ref('garis_kemiskinan')

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

    const formatAxisLabel = (axis) => {
      const labels = {
        'ipm': 'IPM',
        'garis_kemiskinan': 'Garis Kemiskinan (Rp)',
        'pengeluaran_per_kapita': 'Pengeluaran Per Kapita (Rp)'
      }
      return labels[axis] || axis
    }

    const formatValue = (value, axis) => {
      if (axis === 'garis_kemiskinan' || axis === 'pengeluaran_per_kapita') {
        return new Intl.NumberFormat('id-ID', {
          style: 'currency',
          currency: 'IDR',
          minimumFractionDigits: 0
        }).format(value)
      }
      return value.toFixed(2)
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
      
      const datasets = props.clusters.map((cluster, index) => {
        const data = cluster.members.map(member => ({
          x: member[selectedXAxis.value],
          y: member[selectedYAxis.value],
          label: member.kabupaten_kota,
          membership: member.membership || 1.0
        }))

        const clusterLabel = cluster.interpretation?.label 
          ? String(cluster.interpretation.label) 
          : `Cluster ${cluster.id}`
        
        return {
          label: clusterLabel,
          data: data,
          backgroundColor: getClusterColor(index),
          borderColor: '#ffffff',
          borderWidth: 2,
          pointRadius: (ctx) => {
            const membership = ctx.parsed?.membership || 1.0
            return Math.max(4, membership * 8)
          },
          pointHoverRadius: 8,
          pointHoverBorderColor: '#ffffff',
          pointHoverBorderWidth: 3
        }
      })

      const config = {
        type: 'scatter',
        data: { datasets },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          animation: { duration: 0 },
          hover: { animationDuration: 0 },
          responsiveAnimationDuration: 0,
          plugins: {
            title: {
              display: true,
              text: `${formatAxisLabel(selectedXAxis.value)} vs ${formatAxisLabel(selectedYAxis.value)}`
            },
            legend: { display: false },
            tooltip: {
              callbacks: {
                title: (context) => context[0].raw.label,
                label: (context) => {
                  const point = context.raw
                  const lines = [
                    `${formatAxisLabel(selectedXAxis.value)}: ${formatValue(point.x, selectedXAxis.value)}`,
                    `${formatAxisLabel(selectedYAxis.value)}: ${formatValue(point.y, selectedYAxis.value)}`
                  ]
                  if (point.membership && point.membership < 1.0) {
                    lines.push(`Membership: ${(point.membership * 100).toFixed(1)}%`)
                  }
                  return lines
                }
              }
            }
          },
          scales: {
            x: {
              title: { display: true, text: formatAxisLabel(selectedXAxis.value) },
              ticks: { callback: function(value) { return formatValue(value, selectedXAxis.value) } }
            },
            y: {
              title: { display: true, text: formatAxisLabel(selectedYAxis.value) },
              ticks: { callback: function(value) { return formatValue(value, selectedYAxis.value) } }
            }
          },
          interaction: {
            intersect: false,
            mode: 'point'
          }
        }
      }

      chart.value = new Chart(ctx, config)
      
      } catch (error) {
        console.warn('Error creating scatter plot chart:', error)
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
      selectedXAxis,
      selectedYAxis,
      getClusterColor,
      getClusterLabel,
      updateChart
    }
  }
}
</script>

<style scoped>
.scatter-plot-container {
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

.chart-controls {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.axis-select {
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

.chart-legend {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.875rem;
  color: #4a5568;
}

.legend-color {
  width: 12px;
  height: 12px;
  border-radius: 50%;
}

@media (max-width: 768px) {
  .chart-header {
    flex-direction: column;
    align-items: stretch;
  }
  
  .chart-controls {
    justify-content: center;
  }
  
  .chart-wrapper {
    height: 300px;
  }
  
  .chart-legend {
    justify-content: center;
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