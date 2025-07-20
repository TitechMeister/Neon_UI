<script setup lang="ts">
import { ref,watch, onMounted } from 'vue'
import Altimeter from './dashboard/Altimeter.vue'
import GPS from './devices/GPS.vue';
import Velosity from './dashboard/Velosity.vue';
import Angular from './devices/Angular.vue';
import AltimeterChart from './dashboard/AltimeterChart.vue';
import type { AltimeterData } from './dashboard/Altimeter.vue'

// App.vueで状態を管理
const altitudeLogs = ref<AltimeterData[]>([])

// Altimeterから新しいデータを受け取る関数
const handleAltitudeUpdate = (newData: AltimeterData) => {
  altitudeLogs.value.push(newData)
  console.log('Altitude updated:', newData)
  console.log('Total logs count:', altitudeLogs.value.length)
  console.log('Latest 5 logs:', altitudeLogs.value.slice(-5))
  
  // 最新100件のみ保持
  if (altitudeLogs.value.length > 20) {
    altitudeLogs.value.shift()
  }
}
</script>

<template>
  <header>
    <div id="header">
      <h1>Neon App</h1>
    </div>
  </header>
  <!-- <AltimeterChart /> -->
<div class ="grid">
  <div>One</div>
  <Velosity />
  <Altimeter @altitude-updated="handleAltitudeUpdate" />
  <div>Four</div>
  <div>Five</div>
  <AltimeterChart :altitudeLogs="altitudeLogs" />
</div>
</template>

<style>

#header {
  background-color: #434447;
  color: white;
  text-align: center;
}

.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-auto-rows: minmax(300px, 1fr) minmax(300px, 1fr);
  gap: 10px;
  height: calc(100vh - 80px); /* ヘッダーの高さを除いた高さ */
}

.grid-altimeter {
  grid-column: 1 / span 2;
}
.grid-velosity {
  grid-column: 3;
}


</style>
