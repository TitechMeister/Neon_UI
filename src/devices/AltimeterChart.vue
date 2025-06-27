<script setup lang = "ts">
import { Line } from 'vue-chartjs'
import { ref, type Ref ,watch } from 'vue'
import { Chart as ChartJS, Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale } from 'chart.js';

import type {AltimeterData} from './Altimeter.vue'
ChartJS.register(Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale);
const props = defineProps<{ altitudeLogs: AltimeterData[] }>()

const chartData = ref({
        labels: props.altitudeLogs.map(log => log.received_time),
        datasets: [{
          label: 'データ',
          data: props.altitudeLogs.map(log => log.altitude),
          fill: false,
          borderColor: 'rgb(75, 192, 192)',
          tension: 0.1
        }]
      })

const chartOptions = ref({
  responsive: true,
  animation: {
    duration: 0 // Disable animation for faster updates
  },
  responsiveAnimationDuration: 0
})

// altitudeLogsが変化したらchartDataを更新
watch(props.altitudeLogs, (logs) => {
  chartData.value = {
    labels: logs.map(log => log.received_time),
    datasets: [{
      label: 'データ',
      data: logs.map(log => log.altitude),
      fill: false,
      borderColor: 'rgb(75, 192, 192)',
      tension: 0.1
    }]
  }
}, { deep: true })


</script>
<template>
  <h3>Altimeter Chart</h3>
    <Line
    :options="chartOptions"
    :data="chartData"
  />
</template>
