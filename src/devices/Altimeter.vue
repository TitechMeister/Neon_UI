<script setup lang="ts">
import { ref,watch, onMounted } from 'vue'
import AltimeterChart from './AltimeterChart.vue';

export interface AltimeterData {
  id : number;
  altitude: number;
  temperature: number;
  received_time: string;
}

export interface AltimeterLog {
  download_link: string;
  timestamp: string;
}

const altitude = ref<AltimeterData>();
const altitudeLogDLlink = ref<AltimeterLog>();
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
      id: altitude.value.id,
      altitude: altitude.value.altitude,
      temperature: altitude.value.temperature,
      received_time: altitude.value.received_time
    });
    if (altitudeLogs.value.length > 100) {
      altitudeLogs.value.shift(); // Keep only the last 10 logs
    }
  } catch (error: any) {
    console.error(error.message);
  }
}

async function postData() { 
  const url = './api/altimeter/log';
  // Delete the previous log if it exists
  altitudeLogDLlink.value = undefined;
  try {
    const response = await fetch(url, {
      method: 'POST'
    });
    if (!response.ok) {
      throw new Error(`レスポンスステータス: ${response.status}`);
    }
    console.log('Response received:', response);
    altitudeLogDLlink.value = await response.json();
    if (!altitudeLogDLlink.value) {
      throw new Error('Invalid altitude log data received');
    }
    console.log('Altitude log download link:', altitudeLogDLlink.value.download_link);
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


// declare some reactive state here.
</script>

<template>
  <h2>Altimeter</h2>
    <p>Altimeter ID:{{ altitude?.id }}</p>
    <p>Temperature:{{ altitude?.temperature }}</p>
    <p>Altitude:{{ altitude?.altitude }}</p>
    <p>RecTime:{{ altitude?.received_time }}</p>
  <button @click="fetchData" :disabled="isUpdateConstant">Fetch Altimeter Data</button>
  <!-- checkbox to toggle constant updates -->
  <label>
    <input type="checkbox" v-model="isUpdateConstant" />
    Constant Updates
  </label>
  <div><button @click="postData">Post Altimeter Data</button></div>
  <div><p v-if="altitudeLogDLlink?.download_link">
    Download Altimeter Log: 
    <a :href="altitudeLogDLlink.download_link" target="_blank">Download</a>
  </p></div>
  <p v-if="isUpdateConstant">Constant updates are enabled.</p>
  <p v-else>Constant updates are disabled.</p>
  <h3>Log</h3>
  <ul>
    <li v-for="(log, index) in altitudeLogs.slice(-10,-1)" :key="index">
      Altitude: {{ log.altitude }}, RecTime: {{ log.received_time }}
    </li>
  </ul>

  <AltimeterChart :altitudeLogs="altitudeLogs" />
</template>
