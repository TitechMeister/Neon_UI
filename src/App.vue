<script setup lang="ts">
import { ref,watch, onMounted, vModelCheckbox } from 'vue'
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
const isUpdateConstant = ref<boolean>(false);

// 各コンポーネントへの参照
const altimeterRef = ref()
const velocityRef = ref()
const servoRef = ref()
const flighttraceRef = ref()

// 全てのメーターのpostDataを実行する関数
const postAllData = async () => {
  console.log('全メーターのデータをPOST中...')
  
  try {
    // 各コンポーネントのpostDataメソッドを並行実行
    const promises = []
    
    if (altimeterRef.value?.postData) {
      promises.push(altimeterRef.value.postData())
    }
    
    if (velocityRef.value?.postData) {
      promises.push(velocityRef.value.postData())
    }
    
    if (servoRef.value?.postData) {
      promises.push(servoRef.value.postData())
    }
    
    if (flighttraceRef.value?.postData) {
      promises.push(flighttraceRef.value.postData())
    }
    
    await Promise.all(promises)
    console.log('全メーターのデータPOST完了')
  } catch (error) {
    console.error('データPOST中にエラーが発生:', error)
  }
}

// Altimeterから新しいデータを受け取る関数
const handleAltitudeUpdate = (newData: AltimeterData) => {
  console.log(`データ受信 (試行${newData.attemptNumber}回目):`, newData);
  
  // newDataが外れ値なら弾く（試行回数は記録するがデータはスキップ）
  if (newData.altitude < 0 || newData.altitude > 100) {
    console.warn(`外れ値のためスキップ (試行${newData.attemptNumber}回目):`, newData)
    return
  }
  
  altitudeLogs.value.push(newData)
  console.log(`データ受信 (試行${newData.attemptNumber}回目):`, newData)
  console.log('Altitude updated:', newData)
  console.log('Total logs count:', altitudeLogs.value.length)
  console.log('Latest 5 logs:', altitudeLogs.value.slice(-5))
  
  // 最新20件のみ保持（テスト用）
  if (altitudeLogs.value.length > 100) {
    altitudeLogs.value.shift()
  }
}

// Velocityから新しいデータを受け取る関数
const handleVelocityUpdate = (newData: PitotData) => {
  console.log(`Velocity データ受信 (試行${newData.attemptNumber}回目):`, newData);
  
  velocityLogs.value.push(newData)
  console.log('Velocity updated:', newData)
  console.log('Total velocity logs count:', velocityLogs.value.length)
  
  // 最新20件のみ保持（テスト用）
  if (velocityLogs.value.length > 100) {
    velocityLogs.value.shift()
  }
}


</script>

<template>
  <header>
    <div id="header">
      <h1 style="display: inline-block; margin: 0; margin-left: 150px;">Neon App</h1>
      <div style="float: right; display: flex; align-items: center; height: 100%;">
        <button style="margin-right: 12px;" @click="postAllData">PostData</button>
        <label style="margin-right: 8px;">Constant Update</label>
        <input type="checkbox" v-model="isUpdateConstant"/>
      </div>
    </div>
  </header>
  <!-- <AltimeterChart /> -->
<div class ="grid">
  <Flighttrace ref="flighttraceRef" :is-update-constant="isUpdateConstant"/>
  <Velocity ref="velocityRef" @velocity-updated="handleVelocityUpdate" :is-update-constant="isUpdateConstant" />
  <Altimeter ref="altimeterRef" @altitude-updated="handleAltitudeUpdate" :is-update-constant="isUpdateConstant" />
  <Servo ref="servoRef" :is-update-constant="isUpdateConstant"/>
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
