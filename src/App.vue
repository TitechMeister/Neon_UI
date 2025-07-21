<script setup lang="ts">
import { ref,watch, onMounted } from 'vue'
import Altimeter from './dashboard/Altimeter.vue'
import GPS from './devices/GPS.vue';
import Velocity from './dashboard/Velocity.vue';
import VelocityChart from './dashboard/VelocityChart.vue';
import Angular from './devices/Angular.vue';
import AltimeterChart from './dashboard/AltimeterChart.vue';
import type { AltimeterData } from './dashboard/Altimeter.vue'
import type { PitotData } from './dashboard/Velocity.vue'
import Flighttrace from './dashboard/flighttrace.vue';
import Servo from './dashboard/Servo.vue';

// App.vueで状態を管理
const altitudeLogs = ref<AltimeterData[]>([])
const velocityLogs = ref<PitotData[]>([])

// Altimeterから新しいデータを受け取る関数
const handleAltitudeUpdate = (newData: AltimeterData) => {
  // newDataが外れ値なら弾く
  if (newData.altitude < 0 || newData.altitude > 100) {
    console.warn('Received outlier altitude data:', newData)
    return
  }
  altitudeLogs.value.push(newData)
  console.log('Altitude updated:', newData)
  console.log('Total logs count:', altitudeLogs.value.length)
  console.log('Latest 5 logs:', altitudeLogs.value.slice(-5))
  
  // 最新100件のみ保持
  if (altitudeLogs.value.length > 100) {
    altitudeLogs.value.shift()
  }
}

// Velocityから新しいデータを受け取る関数
const handleVelocityUpdate = (newData: PitotData) => {
  velocityLogs.value.push(newData)
  console.log('Velocity updated:', newData)
  console.log('Total velocity logs count:', velocityLogs.value.length)
  
  // 最新100件のみ保持
  if (velocityLogs.value.length > 100) {
    velocityLogs.value.shift()
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
  <Flighttrace />
  <Velocity @velocity-updated="handleVelocityUpdate" />
  <Altimeter @altitude-updated="handleAltitudeUpdate" />
  <Servo />
  <VelocityChart :velocityLogs="velocityLogs" />
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
