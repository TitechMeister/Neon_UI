<script setup lang="ts">
import { ref,watch, onMounted } from 'vue'

interface AltimeterData {
  altitude: number;
  timestamp: string;
}

const altitude = ref<AltimeterData>();
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
  <h1>Neon</h1>
    <p>Welcome to the Neon app!</p>
    <p>Explore the features and enjoy your stay.</p>
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
</template>
