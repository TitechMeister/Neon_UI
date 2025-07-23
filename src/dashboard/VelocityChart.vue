<script setup lang="ts">
import { Line } from 'vue-chartjs'
import { ref, watch } from 'vue'
import { Chart as ChartJS, Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale, Filler } from 'chart.js';

import type { PitotData } from './Velocity.vue'
ChartJS.register(Title, Tooltip, Legend, PointElement, LineElement, CategoryScale, LinearScale, Filler);
const props = defineProps<{ velocityLogs: PitotData[] }>()

const chartData = ref({
  labels: [] as string[],
  datasets: [{
    label: 'Velocity',
    data: [] as Array<{x: number, y: number}>,
    fill: true,
    backgroundColor: 'rgba(0, 255, 0, 0.3)',
    borderColor: 'rgb(0, 255, 0)',
    tension: 0.1
  }]
})

// チャートの再描画を強制するためのキー
const chartKey = ref(0)

const chartOptions = ref<any>({
  responsive: true,
  maintainAspectRatio: false, // アスペクト比を維持しない
  animation: {
    duration: 0 // Disable animation for faster updates
  },
  responsiveAnimationDuration: 0,
  plugins: {
    legend: {
      display: false // 凡例を非表示
    }
  },
  scales: {
    x: {
      type: 'linear' as const,
      display: false, // X軸を表示
      ticks: {
        stepSize: 1,
        callback: function(value: any) {
          return `${value}回目`
        }
      },
      min: 1,
      max: 100
    },
    y: {
      beginAtZero: true,
      title: {
        display: true,
        text: 'Velocity (m/s)'
      }
    }
  }
})

// velocityLogsが変化したらchartDataを更新
watch(() => props.velocityLogs, (logs) => {
  console.log('Velocity Chart updating with logs:', logs.length, 'items')
  console.log('Latest velocity logs:', logs.slice(-3))
  
  // attemptNumberをx軸、velocityをy軸とするデータ形式に変換
  const processedData = logs.map(log => ({
    x: log.attemptNumber, // 試行回数を横軸に使用
    y: log.velocity
  }))
  
  console.log('Processed velocity data for chart:', processedData)
  
  chartData.value = {
    labels: [], // 座標データを使用するため不要
    datasets: [{
      label: 'Velocity',
      data: processedData,
      fill: true,
      backgroundColor: 'rgba(0, 255, 0, 0.3)',
      borderColor: 'rgb(0, 255, 0)',
      tension: 0.1
    }]
  }
  
  // X軸の範囲を動的に設定（直近10回分を表示）
  if (logs.length > 0) {
    const attemptNumbers = logs.map(log => log.attemptNumber)
    const minAttempt = Math.min(...attemptNumbers)
    const maxAttempt = Math.max(...attemptNumbers)
    
    console.log(`Velocity Attempt range: ${minAttempt} - ${maxAttempt}`)
    
    // 直近10回分を表示する範囲を計算
    const displayRangeStart = Math.max(1, maxAttempt - 99) // 最新から10回分遡る
    const displayRangeEnd = maxAttempt // 少し余裕を持たせる
    
    console.log(`Velocity Display range: ${displayRangeStart} - ${displayRangeEnd}`)
    
    // X軸の範囲を設定
    chartOptions.value.scales.x.min = displayRangeStart
    chartOptions.value.scales.x.max = displayRangeEnd
  }
  
  // チャートの再描画を強制
  chartKey.value++
}, { deep: true, immediate: true })
</script>

<template>
  <div class="chart-container">
    <Line
      :key="chartKey"
      :options="chartOptions"
      :data="chartData"
    />
  </div>
</template>

<style scoped>
.chart-container {
  width: 100%;
  height: 100%;
  padding: 10px;
  box-sizing: border-box;
}
</style>
