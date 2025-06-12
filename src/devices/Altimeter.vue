<script setup lang="ts">
import { ref,watch, onMounted } from 'vue'
import { Line } from 'vue-chartjs'
import { Chart as ChartJS, Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale } from 'chart.js';

ChartJS.register(Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale);

interface AltimeterData {
  altitude: number;
  timestamp: string;
}

const altitude = ref<AltimeterData>();
const altitudeLogs = ref<AltimeterData[]>([]);
const isUpdateConstant = ref<boolean>(false);

// component logic
async function fetchData() {
  // Simulate fetching data
  const url = './api/altimeter';
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`レスポンスステータス: ${response.status}`);
    }
    console.log('Response received:', response);
    altitude.value = await response.json();
    // altitude.valueがundefinedでないことを確認
    if (!altitude.value) {
      throw new Error('Invalid altitude data received');
    }
    altitudeLogs.value.push({
      altitude: altitude.value.altitude,
      timestamp: altitude.value.timestamp
    });
    if (altitudeLogs.value.length > 100) {
      altitudeLogs.value.shift(); // Keep only the last 10 logs
    }
  } catch (error: any) {
    console.error(error.message);
  }
}

const fetchDataInInterval = () => {
  fetchData().then(() => {
    if (!isUpdateConstant.value) {
      return; // Exit if constant updates are not enabled
    }
    setTimeout(() => {
      fetchDataInInterval()
    }, 100)
  })
}

onMounted(() => {
  fetchDataInInterval();
});

watch(isUpdateConstant, (newValue) => {
  if (newValue) {
    fetchDataInInterval();
  }
});

// Chart.js setup

const chartData = ref({
        labels: altitudeLogs.value.map(log => log.timestamp),
        datasets: [{
          label: 'データ',
          data: altitudeLogs.value.map(log => log.altitude),
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
watch(altitudeLogs, (logs) => {
  chartData.value = {
    labels: logs.map(log => log.timestamp),
    datasets: [{
      label: 'データ',
      data: logs.map(log => log.altitude),
      fill: false,
      borderColor: 'rgb(75, 192, 192)',
      tension: 0.1
    }]
  }
}, { deep: true })

// declare some reactive state here.
</script>

<template>
  <h2>Altimeter</h2>
    <p>Altitude:{{ altitude?.altitude }}</p>
    <p>TimeStamp:{{ altitude?.timestamp }}</p>
  <button @click="fetchData" :disabled="isUpdateConstant">Fetch Altimeter Data</button>
  <!-- checkbox to toggle constant updates -->
  <label>
    <input type="checkbox" v-model="isUpdateConstant" />
    Constant Updates
  </label>
  <p v-if="isUpdateConstant">Constant updates are enabled.</p>
  <p v-else>Constant updates are disabled.</p>
  <h3>Log</h3>
  <ul>
    <li v-for="(log, index) in altitudeLogs.slice(-10,-1)" :key="index">
      Altitude: {{ log.altitude }}, Timestamp: {{ log.timestamp }}
    </li>
  </ul>
  <h3>Altimeter Chart</h3>
  <div>
    <Line
    :options="chartOptions"
    :data="chartData"
  />
  </div>
</template>
