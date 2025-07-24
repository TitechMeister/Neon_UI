<script setup lang="ts">

import { ref, onMounted, onUnmounted, watch, computed } from 'vue'

// 左上緯度経度:35.253495 136.090031
// 右下緯度経度:35.384179 136.277834

// propsでisUpdateConstantを受け取る
const props = defineProps<{
  isUpdateConstant: boolean
}>()

export interface GPSUIData {
  unixtime: number;
  lon: number;
  lat: number;
  received_time: number;
}

export interface TargetData {
  id: number;
  timestamp: number;
  target_lon: number;
  target_lat: number;
  data: number[];
}

export interface GPSLog {
  download_link: string;
  timestamp: string;
}

const gpsData = ref<GPSUIData>({
  unixtime: 0,
  lon: 1362345000,  // 136.2345度 * 10^7 (地図中央付近のテスト座標)
  lat: 353190000,   // 35.319度 * 10^7 (地図中央付近のテスト座標)
  received_time: 0
})

// 地図の境界座標（実際の緯度経度）
const MAP_BOUNDS = {
  north: 35.384179,  // 上端の緯度
  south: 35.253495,  // 下端の緯度
  east: 136.277834,  // 右端の経度
  west: 136.090031   // 左端の経度
}

// 画像要素への参照
const mapImageRef = ref<HTMLImageElement>()

// 現在位置のピクセル座標
const currentPositionPixel = ref<{x: number, y: number} | null>(null)

// クリック位置のピクセル座標
const clickedPositionPixel = ref<{x: number, y: number} | null>(null)

// クリック位置の緯度経度（リサイズ時の再計算用）
const clickedLatLon = ref<{lat: number, lon: number} | null>(null)

// GPSログのダウンロードリンク
const gpsLogDLlink = ref<GPSLog>();

// GPS軌跡データを保存する配列
const gpsTrail = ref<{lat: number, lon: number, timestamp: number}[]>([])

// 軌跡の最大保持数
const MAX_TRAIL_POINTS = 10000

// 軌跡の更新頻度
const trailInterval = 5000

// GPSデータ（10^7倍された整数値）を実際の緯度経度に変換
const convertGPSToDecimal = (gpsValue: number): number => {
  return gpsValue / 10000000
}

// object-fit: contain時の実際の画像表示サイズと位置を計算
const getActualImageBounds = (imgElement: HTMLImageElement) => {
  const rect = imgElement.getBoundingClientRect()
  const naturalWidth = imgElement.naturalWidth
  const naturalHeight = imgElement.naturalHeight
  
  // 画像の自然な縦横比
  const imageAspectRatio = naturalWidth / naturalHeight
  // 表示領域の縦横比
  const containerAspectRatio = rect.width / rect.height
  
  let actualWidth: number
  let actualHeight: number
  let offsetX: number
  let offsetY: number
  
  if (imageAspectRatio > containerAspectRatio) {
    // 画像が横長：幅に合わせてスケール、上下に余白
    actualWidth = rect.width
    actualHeight = rect.width / imageAspectRatio
    offsetX = 0
    offsetY = (rect.height - actualHeight) / 2
  } else {
    // 画像が縦長：高さに合わせてスケール、左右に余白
    actualWidth = rect.height * imageAspectRatio
    actualHeight = rect.height
    offsetX = (rect.width - actualWidth) / 2
    offsetY = 0
  }
  
  return {
    actualWidth,
    actualHeight,
    offsetX,
    offsetY,
    containerRect: rect
  }
}

// 緯度経度を画像上のピクセル座標に変換（object-fit: contain対応）
const latLonToPixel = (lat: number, lon: number, imgElement: HTMLImageElement) => {
  const bounds = getActualImageBounds(imgElement)
  
  // 緯度経度を0-1の範囲に正規化
  const normalizedX = (lon - MAP_BOUNDS.west) / (MAP_BOUNDS.east - MAP_BOUNDS.west)
  const normalizedY = (MAP_BOUNDS.north - lat) / (MAP_BOUNDS.north - MAP_BOUNDS.south)
  
  // 実際の画像表示領域内でのピクセル座標に変換
  const pixelX = bounds.offsetX + normalizedX * bounds.actualWidth
  const pixelY = bounds.offsetY + normalizedY * bounds.actualHeight
  
  return { x: pixelX, y: pixelY }
}

// GPSデータが更新されたときに現在位置を計算
watch(gpsData, (newData) => {
  if (newData.lat && newData.lon && mapImageRef.value) {
    const actualLat = convertGPSToDecimal(newData.lat)
    const actualLon = convertGPSToDecimal(newData.lon)
    
    // 軌跡データに新しい位置を追加
    gpsTrail.value.push({
      lat: actualLat,
      lon: actualLon,
      timestamp: Date.now()
    })
    
    // 最大保持数を超えた場合は古いデータを削除
    if (gpsTrail.value.length > MAX_TRAIL_POINTS) {
      gpsTrail.value.shift()
    }
    
    currentPositionPixel.value = latLonToPixel(actualLat, actualLon, mapImageRef.value)
    console.log(`現在位置: 緯度 ${actualLat.toFixed(6)}, 経度 ${actualLon.toFixed(6)}`)
    console.log(`ピクセル座標: x=${currentPositionPixel.value.x.toFixed(1)}, y=${currentPositionPixel.value.y.toFixed(1)}`)
  }
}, { deep: true })

// 画像上のピクセル座標を緯度経度に変換（object-fit: contain対応）
const pixelToLatLon = (pixelX: number, pixelY: number, imgElement: HTMLImageElement) => {
  const bounds = getActualImageBounds(imgElement)
  
  // 実際の画像表示領域内での相対位置を計算
  const relativeX = (pixelX - bounds.offsetX) / bounds.actualWidth
  const relativeY = (pixelY - bounds.offsetY) / bounds.actualHeight
  
  // 緯度経度に変換
  const lon = MAP_BOUNDS.west + relativeX * (MAP_BOUNDS.east - MAP_BOUNDS.west)
  const lat = MAP_BOUNDS.north - relativeY * (MAP_BOUNDS.north - MAP_BOUNDS.south)
  
  return { lat, lon }
}

// 現在のGPSデータを画像上のピクセル座標に変換（ヘルパー関数）
const getCurrentPositionPixel = () => {
  if (!mapImageRef.value || !gpsData.value.lat || !gpsData.value.lon) {
    return null
  }
  
  const actualLat = convertGPSToDecimal(gpsData.value.lat)
  const actualLon = convertGPSToDecimal(gpsData.value.lon)
  
  return latLonToPixel(actualLat, actualLon, mapImageRef.value)
}

// 目標座標をAPIに送信する関数
const sendTargetData = async (lat: number, lon: number) => {
  try {
    // 緯度経度を10^7倍して整数に変換
    const targetLat = Math.round(lat * 10000000)
    const targetLon = Math.round(lon * 10000000)
    
    // 現在時刻をUnixタイムスタンプとして取得
    const timestamp = Math.floor(Date.now() / 1000)
    
    // Dataフィールドを0xFFで埋めた32バイト配列
    const data = new Array(32).fill(0xFF)
    
    const targetData: TargetData = {
      id: 0xE0,
      timestamp: timestamp,
      target_lon: targetLon,
      target_lat: targetLat,
      data: data
    }
    
    console.log('送信する目標座標データ:', targetData)
    
    const response = await fetch('/api/data/gps/target', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(targetData)
    })
    
    if (!response.ok) {
      throw new Error(`送信失敗: ${response.status}`)
    }
    
    console.log('目標座標の送信が完了しました')
    
  } catch (error) {
    console.error('目標座標の送信中にエラーが発生しました:', error)
  }
}
const handleMapClick = async (event: MouseEvent) => {
  if (!mapImageRef.value) return
  
  const imageRect = mapImageRef.value.getBoundingClientRect()
  const pixelX = event.clientX - imageRect.left
  const pixelY = event.clientY - imageRect.top
  
  // クリック位置のピクセル座標を保存
  clickedPositionPixel.value = { x: pixelX, y: pixelY }
  
  const coordinates = pixelToLatLon(pixelX, pixelY, mapImageRef.value)
  
  // クリック位置の緯度経度も保存（リサイズ時の再計算用）
  clickedLatLon.value = coordinates
  
  console.log(`クリック位置: 緯度 ${coordinates.lat.toFixed(6)}, 経度 ${coordinates.lon.toFixed(6)}`)
  
  // 目標座標をAPIに送信
  await sendTargetData(coordinates.lat, coordinates.lon)
  
  return coordinates
}

// マーカー位置を再計算する関数
const updateMarkerPosition = () => {
  if (gpsData.value.lat && gpsData.value.lon && mapImageRef.value) {
    const actualLat = convertGPSToDecimal(gpsData.value.lat)
    const actualLon = convertGPSToDecimal(gpsData.value.lon)
    currentPositionPixel.value = latLonToPixel(actualLat, actualLon, mapImageRef.value)
  }
}

// 軌跡の座標を再計算する関数
const updateTrailPositions = () => {
  // リサイズやロード時に軌跡の座標も再計算
  if (mapImageRef.value && gpsTrail.value.length > 0) {
    // 軌跡は自動的に再描画されるため、特別な処理は不要
  }
}

// 軌跡のSVGパスを生成する計算プロパティ
const trailPath = computed(() => {
  if (!mapImageRef.value || gpsTrail.value.length < 2) {
    return ''
  }
  
  let pathData = ''
  
  gpsTrail.value.forEach((point, index) => {
    const pixelPos = latLonToPixel(point.lat, point.lon, mapImageRef.value!)
    
    if (index === 0) {
      pathData += `M ${pixelPos.x} ${pixelPos.y}`
    } else {
      pathData += ` L ${pixelPos.x} ${pixelPos.y}`
    }
  })
  
  return pathData
})

// クリック位置マーカーを再計算する関数
const updateClickedMarkerPosition = () => {
  if (clickedLatLon.value && mapImageRef.value) {
    // 保存された緯度経度から新しいピクセル座標を計算
    clickedPositionPixel.value = latLonToPixel(
      clickedLatLon.value.lat, 
      clickedLatLon.value.lon, 
      mapImageRef.value
    )
  }
}

// ウィンドウリサイズ時の処理
const handleResize = () => {
  updateMarkerPosition()
  updateClickedMarkerPosition()
  updateTrailPositions()
}

// 画像ロード時の処理
const handleImageLoad = () => {
  updateMarkerPosition()
  updateClickedMarkerPosition()
  updateTrailPositions()
}

const fetchDataInInterval = () => {
  fetchData().then(() => {
    if (!props.isUpdateConstant) {
      return; // Exit if constant updates are not enabled
    }
    setTimeout(() => {
      fetchDataInInterval()
    }, trailInterval)
  })
}

onMounted(() => {
  fetchDataInInterval();
  // ウィンドウリサイズイベントを監視
  window.addEventListener('resize', handleResize)
});

onUnmounted(() => {
  // イベントリスナーをクリーンアップ
  window.removeEventListener('resize', handleResize)
})

watch(props, (newValue) => {
  if (newValue) {
    fetchDataInInterval();
  }
});

async function fetchData() {
  try {
    const response = await fetch('/api/data/gps');
    if (!response.ok) {
      throw new Error(`レスポンスステータス: ${response.status}`);
    }
    const data: GPSUIData = await response.json();
    gpsData.value = data;
  } catch (error) {
    console.error('Error fetching GPS data:', error);
  }
}

// GPSデータのログをPOSTする関数
async function postData() { 
  const url = './api/data/gps/log';
  // 前のログが存在する場合は削除
  gpsLogDLlink.value = undefined;
  try {
    const response = await fetch(url, {
      method: 'POST'
    });
    if (!response.ok) {
      throw new Error(`レスポンスステータス: ${response.status}`);
    }
    console.log('GPS log response received:', response);
    gpsLogDLlink.value = await response.json();
    if (!gpsLogDLlink.value) {
      throw new Error('Invalid GPS log data received');
    }
    console.log('GPS log download link:', gpsLogDLlink.value.download_link);
  } catch (error: any) {
    console.error('GPS log POST error:', error.message);
  }
}

// 外部からアクセス可能にする
defineExpose({
  postData
})


</script>

<template>
  <!-- <img src="./biwafp.png" alt="Vue logo" class="logo" /> -->
  <div class ="map-container">
    <div class="map-wrapper">
      <img 
        id="biwamap" 
        ref="mapImageRef"
        src="./mapbiwa2.png" 
        alt="Map Image" 
        class="map-image"
        @click="handleMapClick"
        @load="handleImageLoad"
      />
      
      <!-- GPS軌跡を表示するSVG -->
      <svg 
        v-if="gpsTrail.length > 1"
        class="trail-svg"
        :width="mapImageRef?.getBoundingClientRect().width || 0"
        :height="mapImageRef?.getBoundingClientRect().height || 0"
      >
        <path
          :d="trailPath"
          class="trail-path"
        />
      </svg>
      
      <!-- 現在位置マーカー（赤色） -->
      <div 
        v-if="currentPositionPixel"
        class="position-marker"
        :style="{
          left: `${currentPositionPixel.x}px`,
          top: `${currentPositionPixel.y}px`
        }"
      >
        <div class="marker-dot"></div>
      </div>
      
      <!-- クリック位置マーカー（青色） -->
      <div 
        v-if="clickedPositionPixel"
        class="clicked-marker"
        :style="{
          left: `${clickedPositionPixel.x}px`,
          top: `${clickedPositionPixel.y}px`
        }"
      >
        <div class="clicked-dot"></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.map-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  width: 100%;
  padding: 0px;
  box-sizing: border-box;
}

.map-wrapper {
  position: relative;
  width: 100%;
  height: 100%;
  background-color: #f0f0f0; /* 余白部分の背景色 */
}

.map-image {
  width: 100%;
  height: 100%;
  object-fit: contain;
  cursor: crosshair;
}

.trail-svg {
  position: absolute;
  top: 0;
  left: 0;
  pointer-events: none;
  z-index: 5;
}

.trail-path {
  fill: none;
  stroke: #ff6b35;
  stroke-width: 3;
  stroke-linecap: round;
  stroke-linejoin: round;
  opacity: 0.8;
}

.position-marker {
  position: absolute;
  pointer-events: none;
  transform: translate(-50%, -50%);
  z-index: 10;
}

.clicked-marker {
  position: absolute;
  pointer-events: none;
  transform: translate(-50%, -50%);
  z-index: 11;
}

.marker-dot {
  width: 12px;
  height: 12px;
  background-color: #ff0000;
  border: 2px solid #ffffff;
  border-radius: 50%;
  box-shadow: 0 0 0 2px rgba(255, 0, 0, 0.3);
  animation: pulse 2s infinite;
}

.clicked-dot {
  width: 10px;
  height: 10px;
  background-color: #0066ff;
  border: 2px solid #ffffff;
  border-radius: 50%;
  box-shadow: 0 0 0 2px rgba(0, 102, 255, 0.3);
}

@keyframes pulse {
  0% {
    box-shadow: 0 0 0 2px rgba(255, 0, 0, 0.3);
  }
  50% {
    box-shadow: 0 0 0 8px rgba(255, 0, 0, 0.1);
  }
  100% {
    box-shadow: 0 0 0 2px rgba(255, 0, 0, 0.3);
  }
}
</style>
