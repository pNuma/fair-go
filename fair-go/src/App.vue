<script setup lang="ts">
import { ref } from 'vue';
import { LMap, LTileLayer, LMarker, LPopup } from '@vue-leaflet/vue-leaflet';

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

// 取得した全員の座標リスト
const markers = ref<{ lat: number, lng: number }[]>([]);

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
      message.value = "中間地点の住所を検索中...";

      // 住所を取得
      const addressName = await getAddress(centerResult.lat, centerResult.lng);
      message.value = `中間地点は「${addressName}」付近です！`;
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

// 重心（平均）を計算する関数\\
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
  <header>
    <h1>フェアに行こう</h1>
  </header>

  <main>
    <div class="action-area">
      <button @click="fetchAllLocations" class="calc-btn" :disabled="isLoading">
        {{ isLoading ? '計算中...' : '集合場所を確認' }}
      </button>
      <p v-if="message" class="result-message">{{ message }}</p>

      <button @click="resetAll" class="reset-btn">
        やり直す
      </button>
    </div>


    <div class="input-area">
      <div v-for="(item, index) in locations" :key="index" class="input-group">

        <label>参加者 {{ index + 1 }}</label>

        <input v-model="item.postcode" type="text" placeholder="例: 1000001" maxlength="7">

        <button v-if="locations.length > 2" @click="removeLocation(index)" class="delete-btn">
          削除
        </button>

      </div>

      <button @click="addLocation" class="add-btn">＋ 人数を増やす</button>
    </div>
    <div class="map-wrapper">
    </div>


    <div class="map-wrapper">
      <l-map ref="map" v-model:zoom="zoom" v-model:center="center" :useGlobalLeaflet="false">

        <l-tile-layer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" layer-type="base"
          name="OpenStreetMap"></l-tile-layer>

        <l-marker v-for="(marker, index) in markers" :key="index" :lat-lng="[marker.lat, marker.lng]">
          <l-popup>参加者 {{ index + 1 }}</l-popup>
        </l-marker>

        <l-marker v-if="midpoint" :lat-lng="[midpoint.lat, midpoint.lng]">
          <l-popup>中間地点です</l-popup>
        </l-marker>

      </l-map>
    </div>
  </main>
</template>

<style scoped>
.input-area {
  margin-bottom: 20px;
  padding: 1rem;
  background-color: #f9f9f9;
  border-radius: 8px;
}

.input-group {
  margin-bottom: 10px;
  display: flex;
  gap: 10px;
  align-items: center;
}

input {
  padding: 5px;
  font-size: 1rem;
}

button {
  cursor: pointer;
}

.delete-btn {
  background-color: #ffcccc;
  border: none;
  border-radius: 4px;
}

.map-wrapper {
  height: 600px;
  width: 100%;
}

.reset-btn {
  margin-left: 10px;
  background-color: #888;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.reset-btn:hover {
  background-color: #666;
}

.calc-btn:disabled {
  background-color: #ccc; 
  cursor: not-allowed;    /* 禁止マーク */
  opacity: 0.7;
}
</style>