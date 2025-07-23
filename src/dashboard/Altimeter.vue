<script setup lang="ts">
import { ref, watch, onMounted, computed } from 'vue'

// emitを定義
const emit = defineEmits<{
  'altitude-updated': [data: AltimeterData]
}>()

// propsでisUpdateConstantを受け取る
const props = defineProps<{
  isUpdateConstant: boolean
}>()

const altitudeValue = ref<AltimeterData>() // リアクティブな数値（0-100の想定）
const altitudeLogDLlink = ref<AltimeterLog>();
let attemptCounter = 0; // データ取得の試行回数カウンター
// const isUpdateConstant = ref<boolean>(false);
const altitudeMax = 8
const maxHeight = 400 // 最大高さ（px）

export interface AltimeterData {
  id : number;
  altitude: number;
  received_time: string;
  attemptNumber: number; // n回目の取得試行
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
    if (!props.isUpdateConstant) {
      return; // Exit if constant updates are not enabled
    }
    setTimeout(() => {
      fetchDataInInterval()
    }, 500)
  })
}

onMounted(() => {
  fetchDataInInterval();
});

watch(props, (newValue) => {
  if (newValue) {
    fetchDataInInterval();
  }
});

async function fetchData() {
  const url = './api/data/altimeter';
  attemptCounter++; // 試行回数をインクリメント
  
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`レスポンスステータス: ${response.status}`);
    }
    const data: AltimeterData = await response.json();
    if (!altitudeValue.value) {
      altitudeValue.value = { id: data.id, altitude: 0, received_time: '', attemptNumber: attemptCounter }; // 初期化
    }
    altitudeValue.value.altitude = Number(data.altitude.toFixed(2)); // データからaltitudeを取得
    altitudeValue.value.received_time = data.received_time; // データから受信時間を取得
    altitudeValue.value.attemptNumber = attemptCounter; // 試行回数を設定
    
    // 親コンポーネントに新しいオブジェクトとしてコピーして送信
    emit('altitude-updated', {
      id: altitudeValue.value.id,
      altitude: altitudeValue.value.altitude,
      received_time: altitudeValue.value.received_time,
      attemptNumber: altitudeValue.value.attemptNumber
    })
    
  } catch (error: any) {
    console.error(`データ取得失敗 (試行${attemptCounter}回目):`, error.message);
    // エラーの場合はemitしない（データをスキップ）
  }
}

async function postData() { 
  const url = './api/data/altimeter/log';
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

// 外部からアクセス可能にする
defineExpose({
  postData
})
</script>

<template>

<div>
  <div class="bar-container">
    <div class="bar" :style="{ height: barHeight }"></div>
    <span class="bar-text">{{ altitudeValue?.altitude }}</span>
  </div>
  
  <!-- <button @click="fetchData" :disabled="isUpdateConstant">Fetch Altimeter Data</button> -->
  <!-- checkbox to toggle constant updates -->
  <!-- <label>
    <input type="checkbox" v-model="isUpdateConstant" />
    Constant Updates
  </label> -->
    <!-- <button @click="postData">Post Altimeter Data</button>
    <p v-if="altitudeLogDLlink?.download_link">
      Download Altimeter Log: 
    <a :href="altitudeLogDLlink.download_link" target="_blank">Download</a></p>
    <p v-else>
      Download rink displayed here:
    </p> -->
  
</div>
</template>

<style>

.altimeter-grid{
  display: grid;
  grid-template-columns: 3fr 1fr;
}

.bar-container {
  height: 400px; /* 高さを400pxに設定 */
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
  transition: height 0.15s ease;
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
