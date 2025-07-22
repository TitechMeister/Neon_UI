<script setup lang="ts">
import { Line } from 'vue-chartjs'
import { ref, watch } from 'vue'
import { Chart as ChartJS, Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale, Filler } from 'chart.js';

import type { PitotData } from './Velocity.vue'
ChartJS.register(Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale, Filler);
const props = defineProps<{ velocityLogs: PitotData[] }>()

const chartData = ref({
  labels: [] as string[],
  datasets: [{
    label: 'Velocity',
    data: [] as number[],
    fill: true,
    backgroundColor: 'rgba(0, 255, 0, 0.3)',
    borderColor: 'rgb(0, 255, 0)',
    tension: 0.1
  }]
})

const chartOptions = ref({
  responsive: true,
  maintainAspectRatio: false, // アスペクト比を維持しない
  animation: {
    duration: 0 // Disable animation for faster updates
  },
  responsiveAnimationDuration: 0,
  plugins: {
    legend: {
      display: false // 凡例を非表示
    }
  },
  scales: {
    x: {
      display: false // X軸（横軸）を非表示
    }
  }
})

// velocityLogsが変化したらchartDataを更新
watch(() => props.velocityLogs, (logs) => {
  console.log('Velocity Chart updating with logs:', logs.length, 'items')
  console.log('Latest velocity logs:', logs.slice(-3))
  
  chartData.value = {
    labels: logs.map(log => new Date(log.timestamp * 1000).toLocaleTimeString()),
    datasets: [{
      label: 'Velocity',
      data: logs.map(log => log.velocity),
      fill: true,
      backgroundColor: 'rgba(0, 255, 0, 0.3)',
      borderColor: 'rgb(0, 255, 0)',
      tension: 0.1
    }]
  }
}, { deep: true, immediate: true })
</script>

<template>
  <div class="chart-container">
    <Line
      :options="chartOptions"
      :data="chartData"
    />
  </div>
</template>

<style scoped>
.chart-container {
  width: 100%;
  height: 100%;
  padding: 10px;
  box-sizing: border-box;
}
</style>
