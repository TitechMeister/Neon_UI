<script setup lang="ts">
import { ref,reactive, onMounted } from 'vue'

interface AltimeterData {
  altitude: number;
  timestamp: string;
}

const altitude = ref<AltimeterData>();

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

fetchData();
onMounted(fetchData)
// declare some reactive state here.
</script>

<template>
  <h1>Neon</h1>
    <p>Welcome to the Neon app!</p>
    <p>Explore the features and enjoy your stay.</p>
  <h2>Altimeter</h2>
    <p>Altitude:{{ altitude?.altitude }}</p>
    <p>TimeStamp:{{ altitude?.timestamp }}</p>
</template>
