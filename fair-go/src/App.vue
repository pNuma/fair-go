<script setup lang="ts">
import { ref } from 'vue';
import { LMap, LTileLayer } from '@vue-leaflet/vue-leaflet';

// 地図の初期状態
const zoom = ref(13);
const center = ref([35.681236, 139.767125] as [number, number]);
</script>

<template>
  <header>
    <h1>フェアに行こう</h1>
  </header>
  
  <main>
    <div class="map-wrapper">
      <l-map ref="map" v-model:zoom="zoom" v-model:center="center" :useGlobalLeaflet="false">
        
        <l-tile-layer
          url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
          layer-type="base"
          name="OpenStreetMap"
        ></l-tile-layer>

      </l-map>
    </div>
  </main>
</template>

<style scoped>
.map-wrapper {
  height: 600px;
  width: 100%;
}
</style>