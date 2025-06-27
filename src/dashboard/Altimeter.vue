<script setup lang="ts">
import { ref, watch, onMounted, computed } from 'vue'

const altitudeValue = ref<AltimeterData>() // リアクティブな数値（0-100の想定）
const altitudeLogDLlink = ref<AltimeterLog>();
const isUpdateConstant = ref<boolean>(false);
const altitudeMax = 8
const maxHeight = 200 // 最大高さ（px）

export interface AltimeterData {
  id : number;
  altitude: number;
  received_time: string;
}

export interface AltimeterLog {
  download_link: string;
  timestamp: string;
}

// 高さを計算
const barHeight = computed(() => {
  if (!altitudeValue.value || altitudeValue.value.altitude === undefined) {
    return '0px' // データがない場合は高さを0に設定
  }
  return `${Math.max(0, Math.min(1, altitudeValue.value.altitude/altitudeMax)) * maxHeight}px`
})

const fetchDataInInterval = () => {
  fetchData().then(() => {
    if (!isUpdateConstant.value) {
      return; // Exit if constant updates are not enabled
    }
    setTimeout(() => {
      fetchDataInInterval()
    }, 300)
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

async function fetchData() {
  const url = './api/altimeter';
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`レスポンスステータス: ${response.status}`);
    }
    const data: AltimeterData = await response.json();
    if (!altitudeValue.value) {
      altitudeValue.value = { id: data.id, altitude: 0, received_time: '' }; // 初期化
    }
    altitudeValue.value.altitude = Number(data.altitude.toFixed(2)); // データからaltitudeを取得
    altitudeValue.value.received_time = data.received_time; // データから受信時間を取得
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
</script>

<template>

  <div class="bar-container">
    <div class="bar" :style="{ height: barHeight }"></div>
    <span class="bar-text">{{ altitudeValue?.altitude }}</span>
  </div>
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

</template>

<style>

.bar-container {
  height: 200px; /* 高さを400pxに設定 */
  display: flex;
  align-items: flex-end;
  background-color: rgb(190, 190, 190);
  border-radius: 3px;
  position: relative; /* 追加：または、この要素を基準にしてもOK */
}

.bar {
  width: 100%;
  background: linear-gradient(to top, #0fc5d9, #4dfff0);
  border-radius: 3px 3px 0 0;
  transition: height 0.1s ease;
}

.bar-text {
  position: absolute;
  bottom: 10%; /* 棒グラフの上に配置 */
  left: 50%; /* 棒グラフの中央に配置 */
  transform: translateX(-50%); /* 中央揃え */
  color: #333;
  font-weight: bold;
  font-size: 100px;
  white-space: nowrap;
}
</style>
