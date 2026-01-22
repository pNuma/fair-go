<script setup lang="ts">
import { ref } from 'vue';
import { LMap, LTileLayer } from '@vue-leaflet/vue-leaflet';

// 地図の初期状態
const zoom = ref(13);
const center = ref([35.681236, 139.767125] as [number, number]);

const locations = ref([
  { postcode: '' },
  { postcode: '' }
]);

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


// ▼ テスト用の関数
const testApi = async () => {
  const result = await getCoordinates('100-0001');
  console.log('届いたデータ:', result); // コンソールに表示
};
</script>

<template>
  <header>
    <h1>フェアに行こう</h1>
  </header>

  <main>
    <button @click="testApi">APIテスト（100-0001）</button>
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
</style>