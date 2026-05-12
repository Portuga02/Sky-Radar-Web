<script setup>
import { onMounted } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

onMounted(() => {
  // Inicializa o mapa focado em Recife com zoom 13
  const map = L.map('map').setView([-8.047562, -34.877002], 13)

  // Carrega as imagens do mapa via OpenStreetMap (Gratuito)
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 19,
    attribution: '© OpenStreetMap - SkyRadar'
  }).addTo(map)

  // Um marcador de exemplo (nosso primeiro "Alerta")
  L.marker([-8.047562, -34.877002]).addTo(map)
    .bindPopup('<b>Marco Zero</b><br>Monitoramento Ativo.')
    .openPopup()
})
</script>

<template>
  <div id="map"></div>
</template>

<style>
/* Remove as margens padrão e deixa o mapa em tela cheia */
html, body {
  margin: 0;
  padding: 0;
  height: 100%;
}
#map {
  width: 100vw;
  height: 100vh;
}
</style>