<script setup lang = "ts">
import { Line } from 'vue-chartjs'
import { ref } from 'vue'

import { Chart as ChartJS, Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale } from 'chart.js';

ChartJS.register(Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale);


const chartData = ref({
        labels: ['1月', '2月', '3月', '4月', '5月', '6月', '７月'],
        datasets: [{
          label: 'データ',
          data: [65, 59, 80, 81, 56, 55, 40],
          fill: false,
          borderColor: 'rgb(75, 192, 192)',
          tension: 0.1
        }]
      })

const chartOptions = ref({
  responsive: true
})

</script>
<template>
  <h3>Altimeter Chart</h3>
  <div>
    <Line
    :options="chartOptions"
    :data="chartData"
  />
  </div>
</template>
