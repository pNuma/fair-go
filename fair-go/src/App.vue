<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';
import { LMap, LTileLayer, LMarker, LPopup, LPolyline } from '@vue-leaflet/vue-leaflet';

// 地図の初期状態
const zoom = ref(13);
const center = ref([35.681236, 139.767125] as [number, number]);

const locations = ref([
  { postcode: '' },
  { postcode: '' }
]);

const midpoint = ref<{ lat: number, lng: number } | null>(null);
const message = ref('');
const isLoading = ref(false);

// 入力欄を増やす
const addLocation = () => {
  locations.value.push({ postcode: '' });
};

// 入力欄を減らす（index番目の人を削除）
const removeLocation = (index: number) => {
  if (locations.value.length > 1) {
    locations.value.splice(index, 1);
  }
};

// 座標を取得する関数 (HeartRails Geo APIを使用)
const getCoordinates = async (postcode: string) => {
  const cleanPostcode = postcode.replace(/-/g, '');

  try {
    // HeartRails Geo API
    const url = `https://geoapi.heartrails.com/api/json?method=searchByPostal&postal=${cleanPostcode}`;

    const response = await fetch(url);
    const data = await response.json();
    if (data.response && data.response.location && data.response.location.length > 0) {
      const location = data.response.location[0];

      return {
        lat: parseFloat(location.y), // yが緯度
        lng: parseFloat(location.x)  // xが経度
      };
    } else {
      console.warn("場所が見つかりませんでした");
      return null;
    }
  } catch (error) {
    console.error("通信エラー", error);
    return null;
  }
};

// 座標から最寄り駅を探す関数 (HeartRails Express API)
const getNearestStation = async (lat: number, lng: number) => {
  try {
    const url = `https://express.heartrails.com/api/json?method=getStations&x=${lng}&y=${lat}`;

    const response = await fetch(url);
    const data = await response.json();
    if (data?.response?.station?.[0]) {
      const station = data.response.station[0];

      return `${station.line} ${station.name}駅`;
    } else {
      console.warn("近くに駅が見つかりませんでした");
      return null;
    }
  } catch (error) {
    console.error("通信エラー", error);
    return null;
  }
};


// 座標から住所を調べる関数（逆ジオコーディング）
const getAddress = async (lat: number, lng: number) => {
  try {
    // xが経度, yが緯度（HeartRailsのルール）
    const url = `https://geoapi.heartrails.com/api/json?method=searchByGeoLocation&x=${lng}&y=${lat}`;

    const response = await fetch(url);
    const data = await response.json();

    // データがあれば、最初の候補の「市区町村 + 町域」を返します
    if (data?.response?.location?.[0]) {
      const loc = data.response.location[0];
      return `${loc.city} ${loc.town}`;
    }
    return "海の上や山奥かも？";
  } catch (err) {
    return "住所不明";
  }
};

// リセットする関数
const resetAll = () => {
  locations.value = [
    { postcode: '' },
    { postcode: '' }
  ];

  markers.value = [];
  midpoint.value = null;
  message.value = '';

  // 地図を東京に戻す
  center.value = [35.681236, 139.767125];
  zoom.value = 13;
};

const updateUrl = () => {
  const codes = locations.value
    .map(l => l.postcode)
    .filter(c => c)
    .join(',');

  const url = new URL(window.location.href);
  url.searchParams.set('p', codes);

  window.history.replaceState({}, '', url);
};

const shareResult = async () => {
  const url = window.location.href;
  const text = `中間地点はここ！\n${message.value}`;


  try {
    await navigator.clipboard.writeText(url);
    alert('URLをコピー');
  } catch (err) {
    alert('コピーに失敗');
  }

};

onMounted(() => {
  const url = new URL(window.location.href);
  const p = url.searchParams.get('p');

  if (p) {
    const codes = p.split(',');
    locations.value = codes.map(code => ({ postcode: code }));

    fetchAllLocations();
  }
});


// 取得した全員の座標リスト
const markers = ref<{ lat: number, lng: number }[]>([]);


// 📏 線を引く
const lines = computed(() => {
  if (!midpoint.value || markers.value.length === 0) {
    return [];
  }

  const center = midpoint.value;

  return markers.value.map(marker => {
    const start = [marker.lat, marker.lng] as [number, number];
    const end = [center.lat, center.lng] as [number, number];

    return [start, end];
  });
});

//数字7桁か確認
const isValidPostcode = (code: string) => {
  const cleanCode = code.replace(/-/g, '');
  return /^\d{7}$/.test(cleanCode);
};


const fetchAllLocations = async () => {

  for (const [index, loc] of locations.value.entries()) {
    if (!isValidPostcode(loc.postcode)) {
      message.value = `エラー：参加者 ${index + 1} の郵便番号がおかしいようです（数字7桁を入力してください）`;
      return;
    }
  }

  isLoading.value = true;
  message.value = "計算中…";


  try {
    const promiseList = locations.value.map(loc => {
      return getCoordinates(loc.postcode);
    });

    const results = await Promise.all(promiseList);
    const validPoints = results.filter((p): p is { lat: number, lng: number } => p !== null);

    markers.value = validPoints;
    const centerResult = calculateCentroid(validPoints);

    if (centerResult) {
      midpoint.value = centerResult;
      center.value = [centerResult.lat, centerResult.lng];
      message.value = "中間地点を検索中...";

      // 住所を取得
      const [addressName, stationName] = await Promise.all([
        getAddress(centerResult.lat, centerResult.lng),
        getNearestStation(centerResult.lat, centerResult.lng)
      ]);
      message.value = `中間地点：${addressName} 付近 / 最寄り駅：${stationName}`;
    } else {
      alert("有効な座標が取れませんでした");
    }
  } catch (error) {
    console.error(error);
    message.value = "予期せぬエラーが発生しました…";
  } finally {
    isLoading.value = false;
  }

}

// 重心（平均）を計算する関数
const calculateCentroid = (points: { lat: number, lng: number }[]) => {
  if (points.length === 0) {
    return null;
  }

  const total = points.reduce(
    (acc, curr) => {
      return {
        lat: acc.lat + curr.lat,
        lng: acc.lng + curr.lng
      };
    },
    { lat: 0, lng: 0 }
  );

  const center = {
    lat: total.lat / points.length,
    lng: total.lng / points.length
  };

  return center;
};

</script>

<template>
  <div class="app-container">
    <header>
      <h1>フェアに行こう</h1>
    </header>

    <div class="main-content">
      <div class="sidebar">
        <div class="input-area">
          <div v-for="(item, index) in locations" :key="index" class="input-group">
            <span class="badge">{{ index + 1 }}</span>
            <input v-model="item.postcode" type="tel" placeholder="1000005" maxlength="7"
              @keydown.enter="fetchAllLocations">
            <button v-if="locations.length > 2" @click="removeLocation(index)" class="delete-btn">✕</button>
          </div>

          <button @click="addLocation" class="add-btn">＋ 人数を増やす</button>
        </div>

        <div class="action-area">
          <button @click="fetchAllLocations" class="calc-btn" :disabled="isLoading">
            {{ isLoading ? '計算中...' : '集合場所を検索' }}
          </button>

          <button v-if="midpoint" @click="shareResult" class="share-btn">シェア</button>
          <button @click="resetAll" class="reset-btn">リセット</button>
        </div>

        <p v-if="message" class="message-box">{{ message }}</p>
      </div>

      <div class="map-container">
        <l-map ref="map" v-model:zoom="zoom" v-model:center="center" :useGlobalLeaflet="false">
          <l-tile-layer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" layer-type="base"
            name="OpenStreetMap"></l-tile-layer>

          <l-polyline v-for="(line, index) in lines" :key="index" :lat-lngs="line" color="#2196F3" :weight="4"
            dash-array="10, 10"></l-polyline>

          <l-marker v-for="(marker, index) in markers" :key="index" :lat-lng="[marker.lat, marker.lng]">
            <l-popup>参加者 {{ index + 1 }}</l-popup>
          </l-marker>

          <l-marker v-if="midpoint" :lat-lng="[midpoint.lat, midpoint.lng]">
            <l-popup>ここが中間地点</l-popup>
          </l-marker>
        </l-map>
      </div>
    </div>
  </div>
</template>

<style scoped>
* {
  box-sizing: border-box;
}

/* アプリ全体 */
.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  font-family: sans-serif;
  background-color: #eee;
}

header {
  background-color: #4CAF50;
  color: white;
  text-align: center;
  padding: 10px;
  flex-shrink: 0;
}

header h1 {
  margin: 0;
  font-size: 1.2rem;
}

/* メイン部分（サイドバー ＋ 地図） */
.main-content {
  display: flex;
  flex: 1;

  padding: 10px;
  gap: 10px;

  min-height: 0;
}

.sidebar {
  flex: 1;

  background: #f9f9f9;
  display: flex;
  flex-direction: column;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.1);
  border-radius: 8px;

  min-width: 0;
  min-height: 0;
}

/* 入力リスト部分だけスクロールさせる */
.input-area {
  flex: 1;
  overflow-y: auto;
  padding: 15px;
}

.input-group {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 10px;
}

/* 番号バッジ */
.badge {
  background: #ddd;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  font-size: 12px;
  flex-shrink: 0;
}

/* 入力欄 */
input {
  flex: 1;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 16px;
  width: 100%;
  min-width: 0;
}

/* ボタン類のデザイン */
.add-btn {
  width: 100%;
  padding: 8px;
  background: white;
  border: 1px dashed #2196F3;
  color: #2196F3;
  border-radius: 4px;
  cursor: pointer;
}

.delete-btn {
  background: #ff5252;
  color: white;
  border: none;
  border-radius: 4px;
  width: 30px;
  height: 30px;
  cursor: pointer;
  flex-shrink: 0;
}

/* 下部のボタンエリア */
.action-area {
  padding: 15px;
  border-top: 1px solid #ddd;
  display: flex;
  gap: 10px;
  background: #fff;
  border-radius: 0 0 8px 8px;
}

.calc-btn {
  flex: 2;
  background: #4CAF50;
  color: white;
  padding: 12px;
  border: none;
  border-radius: 6px;
  font-weight: bold;
}

.calc-btn:disabled {
  background: #ccc;
}

.share-btn {
  background: #2196F3;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
  padding: 10px;
  margin-left: 10px;
}

.reset-btn {
  flex: 1;
  background: #888;
  color: white;
  border: none;
  border-radius: 6px;
}

.message-box {
  padding: 10px;
  text-align: center;
  font-size: 0.9rem;
  background: #e8f5e9;
  color: #2e7d32;
}

/* 地図エリア */
.map-container {
  flex: 1;

  position: relative;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.1);
}

/* ========== 縦画面（スマホ・タブレット縦） ========== */
@media (orientation: portrait) {

  .main-content {
    flex-direction: column;
  }

  .input-area {
    padding: 10px;
  }

  .action-area {
    padding: 10px;
  }
}
</style>