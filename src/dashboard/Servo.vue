<script setup lang="ts">
import { ref,onMounted, watch } from 'vue'

export interface ServoUIData {
  rudder: number
  elevator: number
  trim: number
  rudder_servo_angle: number
  elevator_servo_angle: number
  rudder_temperature: number
  elevator_temperature: number
  received_time: number
  timestamp: number
}

const servoData = ref<ServoUIData | null>(null)

// サーボログのダウンロードリンク用インターface
export interface ServoLog {
  download_link: string;
  timestamp: string;
}

const servoLogDLlink = ref<ServoLog>();

// rudderの値を横軸上の位置に変換する関数
const getRudderPosition = () => {
  if (!servoData.value?.rudder) return 50 // デフォルトは中央
  
  // rudderの値を-15から15の範囲で正規化し、5%から95%の位置にマッピング
  const normalizedValue = (servoData.value.rudder + 15) / 30 // 0-1の範囲に正規化
  const position = 5 + normalizedValue * 90 // 5%から95%の範囲にマッピング
  
  // 範囲を制限
  return Math.max(5, Math.min(95, position))
}

// rudder_servo_angleの値を横軸上の位置に変換する関数
const getRudderServoPosition = () => {
  if (!servoData.value?.rudder_servo_angle) return 50 // デフォルトは中央

  // rudder_servo_angleの値を-15から15の範囲で正規化し、5%から95%の位置にマッピング
  const normalizedValue = (servoData.value.rudder_servo_angle + 15) / 30 // 0-1の範囲に正規化
  const position = 5 + normalizedValue * 90 // 5%から95%の範囲に

  // 範囲を制限
  return Math.max(5, Math.min(95, position))
}

// elevator_servo_angle+trimの値を縦軸上の位置に変換する関数
const getElevatorServoPosition = () => {
  if (!servoData.value?.elevator_servo_angle || !servoData.value?.trim) return 50 // デフォルトは中央

  const elevatorPlusTrim = servoData.value.elevator_servo_angle + servoData.value.trim
  // elevator_servo_angle+trimの値を-14から4の範囲で正規化し、90%から10%の位置にマッピング（上下反転）
  const normalizedValue = (elevatorPlusTrim + 14) / 18 // 0-1の範囲に正規化（範囲は18）
  const position = 90 - normalizedValue * 80 // 90%から10%

  // 範囲を制限
  return Math.max(10, Math.min(90, position))
}


// elevator+trimの値を縦軸上の位置に変換する関数
const getElevatorPosition = () => {
  if (!servoData.value?.elevator || !servoData.value?.trim) return 27.8 // デフォルトは中央
  
  const elevatorPlusTrim = servoData.value.elevator + servoData.value.trim
  
  // elevator+trimの値を-14から4の範囲で正規化し、90%から10%の位置にマッピング（上下反転）
  const normalizedValue = (elevatorPlusTrim + 14) / 18 // 0-1の範囲に正規化（範囲は18）
  const position = 90 - normalizedValue * 80 // 90%から10%の範囲にマッピング（上下反転）
  
  // 範囲を制限
  return Math.max(10, Math.min(90, position))
}

// isUpdateConstantをpropsで受け取る
const props = defineProps<{
  isUpdateConstant: boolean
}>()

// fetchData関数を定義
const fetchData = async () => {
  try {
    const response = await fetch('/api/data/servo')
    if (!response.ok) {
      throw new Error('Network response was not ok')
    }
    const data = await response.json() as ServoUIData
    servoData.value = data
    console.log('Servo data fetched:', data)
  } catch (error) {
    console.error('Error fetching servo data:', error)
  }
}

// 定期的にデータを取得する関数
const fetchDataInInterval = () => {
  fetchData().then(() => {
    if (!props.isUpdateConstant) {
      return // Exit if constant updates are not enabled
    }
    setTimeout(() => {
      fetchDataInInterval()
    }, 500) // 500msごとにデータを取得
  })
}

onMounted(() => {
  fetchDataInInterval() // コンポーネントがマウントされたときにデータ取得を開始
})

watch(props, (newValue) => {
  if (newValue.isUpdateConstant) {
    fetchDataInInterval() // isUpdateConstantがtrueになったらデータ取得を開始
  }
})

// サーボデータのログをPOSTする関数
async function postData() { 
  const url = './api/data/servo/log';
  // 前のログが存在する場合は削除
  servoLogDLlink.value = undefined;
  try {
    const response = await fetch(url, {
      method: 'POST'
    });
    if (!response.ok) {
      throw new Error(`レスポンスステータス: ${response.status}`);
    }
    console.log('Servo log response received:', response);
    servoLogDLlink.value = await response.json();
    if (!servoLogDLlink.value) {
      throw new Error('Invalid servo log data received');
    }
    console.log('Servo log download link:', servoLogDLlink.value.download_link);
  } catch (error: any) {
    console.error('Servo log POST error:', error.message);
  }
}

// 外部からアクセス可能にする
defineExpose({
  postData
})

</script>

<template>
  <div class="servo-container">
    <!-- サーボ状態表示 -->
    <div class="servo-display">
      <!-- 十字インジケーター -->
      <div class="crosshair-container">
        <div class="crosshair">
          
          <!-- 中央の十字線 -->
          <div class="vertical-line"></div>
          <div class="horizontal-line"></div>
          
          <!-- rudder値を表す緑色の円 -->
          <div 
            class="rudder-indicator"
            :style="{
              left: `${getRudderPosition()}%`
            }"
          ></div>
          <!-- rudder実値を表す黄色の円 -->
          <div 
            class="rudder-real-indicator"
            :style="{
              left: `${getRudderServoPosition()}%`
            }"
          ></div>
          <!-- elevator+trim値を表す黄色の円 -->
          <div 
            class="elevator-indicator"
            :style="{
              top: `${getElevatorPosition()}%`
            }"
          ></div>
          <!-- elevator+trim実値を表す黄色の円 -->
          <div 
            class="elevator-real-indicator"
            :style="{
              top: `${getElevatorServoPosition()}%`
            }"
          ></div>
        </div>
      </div>
      
      <!-- 数値表示テーブル -->
      <div class="servo-values">
        <table class="values-table">
          <thead>
            <tr>
              <th></th>
              <th>Rud</th>
              <th>Elv</th>
              <th>Trim</th>
              <th>Sum</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td class="row-label">Target</td>
              <td class="value">{{ servoData?.rudder?.toFixed(2) || '-12.34' }}</td>
              <td class="value">{{ servoData?.elevator?.toFixed(2) || '-12.34' }}</td>
              <td class="value">{{ servoData?.trim?.toFixed(2) || '-1.23' }}</td>
              <td class="value">{{ (servoData?.rudder && servoData?.trim ? (servoData.elevator + servoData.trim) : -1.23).toFixed(2) }}</td>
            </tr>
            <tr>
              <td class="row-label">Actual</td>
              <td class="value">{{ servoData?.rudder_servo_angle?.toFixed(2) || '-12.34' }}</td>
              <td class="value">{{ servoData?.elevator_servo_angle?.toFixed(2) || '-12.34' }}</td>
              <td class="value">{{ servoData?.trim?.toFixed(2) || '-1.23' }}</td>
              <td class="value">{{ (servoData?.rudder_servo_angle && servoData?.trim ? (servoData.elevator_servo_angle + servoData.trim) : -1.23).toFixed(2) }}</td>
            </tr>
          </tbody>
        </table>
      </div>
      
      <!-- 温度表示 -->
      <div class="temperature-display">
        <div class="temp-item">
          <span class="temp-label">Rud Temp:</span>
          <span class="temp-value">{{ servoData?.rudder_temperature?.toFixed(1) || '25.0' }}°C</span>
        </div>
        <div class="temp-item">
          <span class="temp-label">Elv Temp:</span>
          <span class="temp-value">{{ servoData?.elevator_temperature?.toFixed(1) || '24.5' }}°C</span>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.servo-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  padding: 20px;
  background-color: transparent;
}

.servo-display {
  background: transparent;
  padding: 20px;
  max-width: 500px;
  width: 100%;
}

/* 十字インジケーター */
.crosshair-container {
  display: flex;
  justify-content: center;
  margin-bottom: 30px;
}

.crosshair {
  position: relative;
  width: 400px;
  height: 200px;
  background-color: #f0f0f0;
  border: 1px solid #ddd;
}

.vertical-line {
  position: absolute;
  top: 10%;
  left: 50%;
  transform: translateX(-50%);
  width: 4px;
  height: 80%;
  background-color: #007bff;
}

.horizontal-line {
  position: absolute;
  top: 27.8%; /* 縦軸における0の位置に調整 */
  left: 5%;
  transform: translateY(-50%);
  width: 90%;
  height: 4px;
  background-color: #dc3545;
}

/* rudder値を表す緑色の円 */
.rudder-indicator {
  position: absolute;
  top: 27.8%; /* 横軸の位置に合わせて調整 */
  transform: translate(-50%, -50%);
  width: 16px;
  height: 16px;
  border: 3px solid #28a745;
  border-radius: 50%;
  background-color: transparent;
  z-index: 10;
}

.rudder-real-indicator {
  position: absolute;
  top: 27.8%; /* 横軸の位置に合わせて調整 */
  transform: translate(-50%, -50%);
  width: 16px;
  height: 16px;
  border: 3px solid #000000;
  border-radius: 50%;
  background-color: #28a745;
  z-index: 10;
}

/* elevator+trim値を表す黄色の円 */
.elevator-indicator {
  position: absolute;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 16px;
  height: 16px;
  border: 3px solid #bdaa19;
  border-radius: 50%;
  background-color: transparent;
  z-index: 10;
}

/* elevator+trim値を表す黄色の円 */
.elevator-real-indicator {
  position: absolute;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 16px;
  height: 16px;
  border: 3px solid #000000;
  border-radius: 50%;
  background-color: #bdaa19;
  z-index: 10;
}


/* 数値表示テーブル */
.servo-values {
  margin-bottom: 20px;
}

.values-table {
  width: 100%;
  border-collapse: collapse;
  font-family: 'Courier New', monospace;
  font-size: 16px;
}

.values-table th {
  background-color: #f8f9fa;
  padding: 8px 12px;
  text-align: center;
  font-weight: bold;
  border: 1px solid #dee2e6;
  font-size: 14px;
}

.values-table td {
  padding: 8px 12px;
  text-align: center;
  border: 1px solid #dee2e6;
}

.row-label {
  background-color: #f8f9fa;
  font-weight: bold;
  width: 60px;
}

.value {
  font-weight: bold;
  color: #495057;
  min-width: 80px;
}

/* 温度表示 */
.temperature-display {
  display: flex;
  justify-content: space-around;
  padding-top: 15px;
  border-top: 1px solid #dee2e6;
}

.temp-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
}

.temp-label {
  font-size: 12px;
  color: #6c757d;
  font-weight: 500;
}

.temp-value {
  font-size: 14px;
  font-weight: bold;
  color: #495057;
  padding: 6px 12px;
  background-color: #f8f9fa;
  border: 1px solid #dee2e6;
}
</style>
