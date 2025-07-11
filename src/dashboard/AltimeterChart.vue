<script setup lang = "ts">
import { Line } from 'vue-chartjs'
import { ref, type Ref ,watch } from 'vue'
import { Chart as ChartJS, Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale, Filler } from 'chart.js';

import type {AltimeterData} from './Altimeter.vue'
ChartJS.register(Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale, Filler);
const props = defineProps<{ altitudeLogs: AltimeterData[] }>()

const chartData = ref({
  labels: [] as string[],
  datasets: [{
    label: 'データ',
    data: [] as number[],
    fill: true,
    backgroundColor: 'rgba(75, 192, 192, 0.8)',
    borderColor: 'rgb(75, 192, 192)',
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

// altitudeLogsが変化したらchartDataを更新
watch(() => props.altitudeLogs, (logs) => {
  console.log('Chart updating with logs:', logs.length, 'items')
  console.log('Latest logs:', logs.slice(-3))
  
  chartData.value = {
    labels: logs.map(log => log.received_time),
    datasets: [{
      label: 'データ',
      data: logs.map(log => log.altitude),
      fill: true, // 塗りつぶしを有効化
      backgroundColor: 'rgba(75, 192, 192, 0.8)', // 半透明の背景色
      borderColor: 'rgb(75, 192, 192)',
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
