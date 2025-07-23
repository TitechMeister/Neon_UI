<script setup lang="ts">
import { ref, onMounted, computed, watch } from 'vue';

// emitを定義
const emit = defineEmits<{
  'velocity-updated': [data: PitotData]
}>()

// propsでisUpdateConstantを受け取る
const props = defineProps<{
  isUpdateConstant: boolean
}>()

export interface PitotData {
  id: number;
  timestamp: number;
  temperature: number;
  velocity: number;
  pressure_v_raw: number;
  pressure_a_raw: number;
  pressure_s_raw: number;
  attemptNumber: number; // n回目の取得試行
}

export interface PitotLog {
  download_link: string;
  timestamp: string;
}

// 仮の値（後から実際の値に置き換え）
const currentVelocity = ref<PitotData>()
const velocityLogDLlink = ref<PitotLog>();
let attemptCounter = 0; // データ取得の試行回数カウンター

const maxVelocity = 10 // 最大速度

// 外側メーターの仮の値
const outerValue = ref(200)
const maxOuterValue = 300 // 外側メーターの最大値

// パーセンテージを計算
const percentage = computed(() => {
  if (!currentVelocity.value) {
    return 0; // データがない場合は0を返す
  }
  return Math.min(Math.max(currentVelocity.value?.velocity / maxVelocity, 0), 1)
})

// 外側メーターのパーセンテージ
const outerPercentage = computed(() => {
  return Math.min(Math.max(outerValue.value / maxOuterValue, 0), 1)
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
  const url = './api/data/pitot';
  attemptCounter++; // 試行回数をインクリメント
  
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`レスポンスステータス: ${response.status}`);
    }
    console.log('Response received:', response);
    const data: PitotData = await response.json();
    if (!currentVelocity.value) {
      currentVelocity.value = { id: data.id, timestamp: 0, temperature: 0, velocity: 0, pressure_v_raw: 0, pressure_a_raw: 0, pressure_s_raw: 0, attemptNumber: attemptCounter }; // 初期化
    }
    currentVelocity.value.velocity = data.velocity;
    currentVelocity.value.attemptNumber = attemptCounter; // 試行回数を設定
    
    // 親コンポーネントに新しいオブジェクトとしてコピーして送信
    emit('velocity-updated', {
      id: data.id,
      timestamp: data.timestamp,
      temperature: data.temperature,
      velocity: data.velocity,
      pressure_v_raw: data.pressure_v_raw,
      pressure_a_raw: data.pressure_a_raw,
      pressure_s_raw: data.pressure_s_raw,
      attemptNumber: attemptCounter
    })
    
    console.log('Received data:', data);
  } catch (error: any) {
    console.error(`データ取得失敗 (試行${attemptCounter}回目):`, error.message);
    // エラーの場合はemitしない（データをスキップ）
  }
}

async function postData() { 
  const url = './api/data/pitot/log';
  // Delete the previous log if it exists
  velocityLogDLlink.value = undefined;
  try {
    const response = await fetch(url, {
      method: 'POST'
    });
    if (!response.ok) {
      throw new Error(`レスポンスステータス: ${response.status}`);
    }
    console.log('Response received:', response);
    velocityLogDLlink.value = await response.json();
    if (!velocityLogDLlink.value) {
      throw new Error('Invalid velocity log data received');
    }
    console.log('Velocity log download link:', velocityLogDLlink.value.download_link);
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
  <div class="speed-container">
    <div class="speed">
      
      <!-- 外側メーターのプログレス -->
      <div 
        class="outer-speed-progress" 
        :style="{ '--outer-progress': outerPercentage }"
      ></div>
      
      <!-- 内側メーターの背景 -->
      <div class="speed-background"></div>
      
      <!-- 内側メーターのプログレス -->
      <div 
        class="speed-progress" 
        :style="{ '--progress': percentage }"
      ></div>
      
      <!-- 中央の数値表示 -->
      <div class="speed-value">
        <span class="outer-value">{{ outerValue }}</span>
        <span class="value">{{ currentVelocity?.velocity.toFixed(2) }}</span>
        <span class="unit">m/s</span>
      </div>
    </div>
  </div>
</template>

<style scoped>
.speed-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  width: 100%;
  padding: 10px;
  box-sizing: border-box;
}

.speed {
  position: relative;
  width: 500px;  /* 外側メーターに合わせて拡大 */
  height: 350px; /* メーターの上部が見えるように高さを増加 */
  display: flex;
  justify-content: center;
  align-items: flex-start; /* centerからflex-startに変更してメーターを上寄せ */
  /* overflow: hidden を削除して上部を表示 */
}

.speed-background {
  position: absolute;
  width: 400px;          /* 内側円の直径 */
  height: 400px;         /* 内側円の直径 */
  border-radius: 50%;
  top: 20px;             /* さらに下にずらす */
  left: 50%;
  transform: translateX(-50%);
  background: conic-gradient(
    from 240deg,
    #e0e0e0 0deg 240deg,
    transparent 240deg 360deg
  );
  mask: radial-gradient(
    circle at center,
    transparent 140px,   /* 内径 = 400px × 0.35 */
    black 140px,
    black 180px,         /* 外径 = 400px × 0.45 */
    transparent 180px
  );
  -webkit-mask: radial-gradient(
    circle at center,
    transparent 140px,
    black 140px,
    black 180px,
    transparent 180px
  );
}

.speed-progress {
  position: absolute;
  width: 400px;          /* 内側円の直径 */
  height: 400px;         /* 内側円の直径 */
  border-radius: 50%;
  top: 20px;             /* さらに下にずらす */
  left: 50%;
  transform: translateX(-50%);
  background: conic-gradient(
    from 240deg,
    #00ff00 0deg calc(240deg * var(--progress)),
    transparent calc(240deg * var(--progress)) 360deg
  );
  mask: radial-gradient(
    circle at center,
    transparent 140px,   /* 内径 = 400px × 0.35 */
    black 140px,
    black 180px,         /* 外径 = 400px × 0.45 */
    transparent 180px
  );
  -webkit-mask: radial-gradient(
    circle at center,
    transparent 140px,
    black 140px,
    black 180px,
    transparent 180px
  );
  transition: background 0.3s ease-in-out;
}


/* 外側メーターのプログレス */
.outer-speed-progress {
  position: absolute;
  width: 500px;          /* 外側円の直径 */
  height: 500px;         /* 外側円の直径 */
  border-radius: 50%;
  top: -30px;            /* さらに下にずらす */
  left: 50%;
  transform: translateX(-50%);
  background: conic-gradient(
    from 240deg,
    #ff4444 0deg calc(240deg * var(--outer-progress)),
    transparent calc(240deg * var(--outer-progress)) 360deg
  );
  mask: radial-gradient(
    circle at center,
    transparent 180px,   /* 外側内径 */
    black 180px,
    black 200px,         /* 外側外径 */
    transparent 200px
  );
  -webkit-mask: radial-gradient(
    circle at center,
    transparent 180px,
    black 180px,
    black 200px,
    transparent 200px
  );
  transition: background 0.3s ease-in-out;
}

.speed-value {
  position: absolute;
  bottom: 10px;          /* 数値表示もさらに下に移動 */
  left: 50%;
  transform: translateX(-50%);
  text-align: center;
  font-family: 'Arial', sans-serif;
  z-index: 10;
}

.outer-value {
  display: block;
  font-size: 5rem;
  color: #777;
  line-height: 1;
  margin-bottom: 10px;
}

.value {
  display: block;
  font-size: 8.5rem;
  /* font-weight: bold; */
  color: #333;
  line-height: 1;
}

.unit {
  display: block;
  font-size: 1.2rem;
  color: #666;
  margin-top: 4px;
}
</style>
