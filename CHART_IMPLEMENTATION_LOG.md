# Chart.js 実装 備忘録

## 概要

Vue.js + Chart.js（vue-chartjs）を使用して、リアルタイム高度データの折れ線グラフを実装。

## 実装日

2025 年 7 月 11 日

## 主要な実装内容

### 1. プロジェクト構成

```
src/
├── App.vue                    # 親コンポーネント（状態管理）
├── dashboard/
│   ├── Altimeter.vue         # 高度データ取得・表示
│   └── AltimeterChart.vue    # グラフ表示
└── devices/
    └── ...
```

### 2. データフロー設計

1. **Altimeter.vue** → API からデータ取得
2. **Altimeter.vue** → `emit('altitude-updated', data)`で親に送信
3. **App.vue** → データを受け取り、`altitudeLogs`配列に蓄積
4. **App.vue** → `:altitudeLogs="altitudeLogs"`でチャートに props 渡し
5. **AltimeterChart.vue** → watch でリアクティブ更新

### 3. 主要な技術課題と解決方法

#### 課題 1: `<script setup>`への移行

**問題**: 従来の`data()`メソッドから`<script setup>`構文への書き換え
**解決**:

```vue
// 従来
export default {
  data() {
    return { count: 0 }
  }
}

// <script setup>
const count = ref(0)
```

#### 課題 2: Chart.js の必要なコンポーネント登録

**問題**: 折れ線グラフが表示されない
**解決**: `LineElement`と`PointElement`の登録を追加

```javascript
import {
  Chart as ChartJS,
  Title,
  Tooltip,
  Legend,
  PointElement,
  LineElement,
  CategoryScale,
  LinearScale,
} from "chart.js";
ChartJS.register(
  Title,
  Tooltip,
  Legend,
  PointElement,
  LineElement,
  CategoryScale,
  LinearScale
);
```

#### 課題 3: リアクティブなグラフ更新

**問題**: データが更新されてもグラフが変化しない
**解決**: watch で`chartData`を新しいオブジェクトに置き換え

```javascript
watch(
  () => props.altitudeLogs,
  (logs) => {
    chartData.value = {
      labels: logs.map((log) => log.received_time),
      datasets: [
        {
          /* 新しいデータセット */
        },
      ],
    };
  },
  { deep: true, immediate: true }
);
```

#### 課題 4: アニメーション無効化

**問題**: グラフ更新時のアニメーションが不要
**解決**: Chart.js オプションでアニメーション無効化

```javascript
const chartOptions = ref({
  animation: { duration: 0 },
  responsiveAnimationDuration: 0,
});
```

#### 課題 5: グリッドサイズの統一

**問題**: グラフ領域が他のコンポーネントと大きさが合わない
**解決**:

- CSS Grid 設定: `grid-auto-rows: minmax(300px, 1fr)`
- Chart.js 設定: `maintainAspectRatio: false`
- コンテナー設定: `width: 100%; height: 100%`

#### 課題 6: 配列データの参照問題

**問題**: 配列内の全ての値が同時に変化してしまう
**解決**: オブジェクトのコピーを作成して送信

```javascript
// 問題のあるコード
emit("altitude-updated", altitudeValue.value);

// 修正後
emit("altitude-updated", {
  id: altitudeValue.value.id,
  altitude: altitudeValue.value.altitude,
  received_time: altitudeValue.value.received_time,
});
```

### 4. 最終的なファイル構成

#### App.vue

```vue
<script setup lang="ts">
import type { AltimeterData } from "./dashboard/Altimeter.vue";

const altitudeLogs = ref<AltimeterData[]>([]);

const handleAltitudeUpdate = (newData: AltimeterData) => {
  altitudeLogs.value.push(newData);
  if (altitudeLogs.value.length > 100) {
    altitudeLogs.value.shift(); // 最新100件のみ保持
  }
};
</script>

<template>
  <Altimeter @altitude-updated="handleAltitudeUpdate" />
  <AltimeterChart :altitudeLogs="altitudeLogs" />
</template>
```

#### Altimeter.vue（重要部分）

```vue
<script setup lang="ts">
const emit = defineEmits<{
  "altitude-updated": [data: AltimeterData];
}>();

async function fetchData() {
  // APIからデータ取得
  const data: AltimeterData = await response.json();

  // 新しいオブジェクトとして送信（重要！）
  emit("altitude-updated", {
    id: data.id,
    altitude: data.altitude,
    received_time: data.received_time,
  });
}
</script>
```

#### AltimeterChart.vue

```vue
<script setup lang="ts">
import { Line } from "vue-chartjs";
import { Chart as ChartJS /* 必要なコンポーネント */ } from "chart.js";

ChartJS.register(/* 必要なコンポーネント */);

const props = defineProps<{ altitudeLogs: AltimeterData[] }>();

const chartData = ref({
  labels: [] as string[],
  datasets: [
    {
      data: [] as number[],
      fill: true, // 塗りつぶし有効
      backgroundColor: "rgba(75, 192, 192, 0.2)", // 半透明背景
      borderColor: "rgb(75, 192, 192)",
      tension: 0.1,
    },
  ],
});

const chartOptions = ref({
  responsive: true,
  maintainAspectRatio: false,
  animation: { duration: 0 },
  plugins: { legend: { display: false } },
  scales: { x: { display: false } },
});

watch(
  () => props.altitudeLogs,
  (logs) => {
    chartData.value = {
      labels: logs.map((log) => log.received_time),
      datasets: [
        {
          /* 新しいデータセット */
        },
      ],
    };
  },
  { deep: true, immediate: true }
);
</script>
```

### 5. 学んだベストプラクティス

1. **Vue のリアクティブシステム**: オブジェクトの参照に注意し、必要に応じてコピーを作成
2. **Chart.js の設定**: レスポンシブ対応とアニメーション制御が重要
3. **コンポーネント間通信**: props down, events up パターンの徹底
4. **デバッグ**: console.log を活用した段階的な問題解決
5. **CSS Grid**: `minmax()`と`gap`を使った柔軟なレイアウト

### 6. パフォーマンス考慮

- アニメーション無効化でリアルタイム更新を高速化
- 配列サイズ制限（100 件）でメモリ使用量抑制
- `immediate: true`で初期描画の確実な実行

### 7. 今後の拡張可能性

- 複数データセットの対応
- 時系列フィルタリング機能
- データエクスポート機能
- リアルタイム更新頻度の調整機能

---

## 注意点・トラブルシューティング

### よくあるエラー

1. **Chart.js コンポーネント未登録**: `LineElement`, `PointElement`の登録忘れ
2. **TypeScript エラー**: 初期配列の型指定 `[] as string[]`
3. **リアクティブ更新されない**: `watch`の`deep: true`オプション忘れ
4. **参照問題**: オブジェクトのシャローコピー忘れ

### デバッグ方法

```javascript
// データフローの確認
console.log("Altitude updated:", newData);
console.log("Total logs count:", altitudeLogs.value.length);
console.log("Chart updating with logs:", logs.length, "items");
```

この実装により、安定したリアルタイムグラフ表示システムが完成しました。
