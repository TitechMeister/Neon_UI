<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';

export interface PitotData {
  id: number;
  timestamp: number;
  temperature: number;
  velocity: number;
  pressure_v_raw: number;
  pressure_a_raw: number;
  pressure_s_raw: number;
}

// 仮の値（後から実際の値に置き換え）
const currentVelocity = ref(1)
const maxVelocity = 10 // 最大速度

// パーセンテージを計算
const percentage = computed(() => {
  return Math.min(Math.max(currentVelocity.value / maxVelocity, 0), 1)
})

// // SVG円弧の計算
// const radius = 80
// const strokeWidth = 20
// const normalizedRadius = radius - strokeWidth * 2
// const circumference = normalizedRadius * 2 * Math.PI

// // 半円（180度）の長さ
// const halfCircumference = circumference / 2

// // ストロークのオフセット計算（右から左へ進行）
// const strokeDashoffset = computed(() => {
//   return halfCircumference - (percentage.value * halfCircumference)
// })

const fetchDataInInterval = () => {
  fetchData().then(() => {
    setTimeout(() => {
      fetchDataInInterval()
    }, 500)
  })
}

onMounted(() => {
  fetchDataInInterval();
});

async function fetchData() {
  const url = './api/data/pitot';
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`レスポンスステータス: ${response.status}`);
    }
    console.log('Response received:', response);
    const data: PitotData = await response.json();
    currentVelocity.value = data.velocity;
    // 後でここで currentVelocity.value = data.velosity を設定
    console.log('Received data:', data);
  } catch (error: any) {
    console.error(error.message);
  }
}
</script>

<template>
  <div class="speedometer-container">
    <div class="speedometer">
      <!-- 背景の半円 -->
      <div class="speedometer-background"></div>
      
      <!-- プログレス表示用の半円 -->
      <div 
        class="speedometer-progress" 
        :style="{ '--progress': percentage }"
      ></div>
      
      <!-- 中央の数値表示 -->
      <div class="speedometer-value">
        <span class="value">{{ currentVelocity.toFixed(2) }}</span>
        <span class="unit">m/s</span>
      </div>
    </div>
  </div>
</template>

<style scoped>
.speedometer-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  padding: 0px;
}

.speedometer {
  position: relative;
  width: 300px;
  height: 150px;
  display: flex;
  justify-content: center;
  align-items: flex-end;
}

.speedometer-background {
  position: absolute;
  width: 300px;
  height: 300px;
  border-radius: 50%;
  background: conic-gradient(
    from 240deg,
    #e0e0e0 0deg 240deg,
    transparent 240deg 360deg
  );
  mask: radial-gradient(
    circle at center,
    transparent 105px,
    black 105px,
    black 135px,
    transparent 135px
  );
  -webkit-mask: radial-gradient(
    circle at center,
    transparent 105px,
    black 105px,
    black 135px,
    transparent 135px
  );
}

.speedometer-progress {
  position: absolute;
  width: 300px;
  height: 300px;
  border-radius: 50%;
  background: conic-gradient(
    from 240deg,
    #00ff00 0deg calc(240deg * var(--progress)),
    transparent calc(240deg * var(--progress)) 360deg
  );
  mask: radial-gradient(
    circle at center,
    transparent 105px,
    black 105px,
    black 135px,
    transparent 135px
  );
  -webkit-mask: radial-gradient(
    circle at center,
    transparent 105px,
    black 105px,
    black 135px,
    transparent 135px
  );
  transition: background 0.3s ease-in-out;
}

.speedometer-value {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  text-align: center;
  font-family: 'Arial', sans-serif;
  z-index: 10;
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
