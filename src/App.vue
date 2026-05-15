<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

const mapElement = ref(null)
let mapInstance = null
let userMarker = null
let markersLayer = null // Grupo para os pinos do mapa

// Começa vazio! Os dados agora vêm do Backend V8!
const riskAreas = ref([])

const criticalAlerts = computed(() => {
  return riskAreas.value.filter(area => area.level === 'red' || area.level === 'orange')
})

const alertColors = {
  green: '#22c55e',
  yellow: '#facc15',
  orange: '#f97316',
  red: '#ef4444'
}

// 🌐 NOVO: Função que consome a API do Laravel
const fetchAlerts = async () => {
  try {
    const response = await fetch('http://localhost:8000/api/alerts')
    if (!response.ok) throw new Error('Falha ao conectar com o Radar API')

    const data = await response.json()
    riskAreas.value = data // Salva os dados do backend

    renderMarkers() // Chama a função para desenhar os pinos no mapa
  } catch (error) {
    console.error("Erro de conexão:", error)
    alert("Falha ao comunicar com o servidor do SkyRadar.")
  }
}

// 📍 NOVO: Função que desenha os pinos depois que a API responde
const renderMarkers = () => {
  markersLayer.clearLayers() // Limpa o mapa antes de desenhar novos

  riskAreas.value.forEach(area => {
    const pinIcon = L.divIcon({
      className: 'custom-pin',
      html: `
        <div class="relative flex items-center justify-center w-8 h-8 drop-shadow-xl">
          <svg viewBox="0 0 24 24" fill="${alertColors[area.level]}" stroke="white" stroke-width="2" class="w-full h-full">
            <path stroke-linecap="round" stroke-linejoin="round" d="M15 10.5a3 3 0 11-6 0 3 3 0 016 0z" />
            <path stroke-linecap="round" stroke-linejoin="round" d="M19.5 10.5c0 7.142-7.5 11.25-7.5 11.25S4.5 17.642 4.5 10.5a7.5 7.5 0 1115 0z" />
          </svg>
          ${area.level === 'red' || area.level === 'orange' ? '<span class="absolute top-[25%] w-2.5 h-2.5 bg-white rounded-full animate-ping opacity-75"></span>' : ''}
        </div>
      `,
      iconSize: [32, 32],
      iconAnchor: [16, 32],
      popupAnchor: [0, -32]
    })

    const marker = L.marker([area.lat, area.lng], { icon: pinIcon })
      .bindPopup(`
        <div style="font-family: sans-serif;">
          <strong style="color: ${alertColors[area.level]}; text-transform: uppercase; font-size: 10px;">
            Nível: ${area.level}
          </strong><br>
          <b>${area.name}</b><br>
          <span style="font-size: 12px; color: #555;">${area.desc}</span>
        </div>
      `)

    markersLayer.addLayer(marker) // Adiciona no grupo de camadas
  })
}

const resetView = () => {
  if (mapInstance) {
    mapInstance.flyTo([-8.05, -34.9], 12, { duration: 1.5 })
  }
}

const locateUser = () => {
  if (!navigator.geolocation) {
    alert("Seu navegador não suporta geolocalização.")
    return
  }

  navigator.geolocation.getCurrentPosition(
    (position) => {
      const lat = position.coords.latitude
      const lng = position.coords.longitude
      mapInstance.flyTo([lat, lng], 15, { duration: 1.5 })
      if (userMarker) mapInstance.removeLayer(userMarker)

      const userPinIcon = L.divIcon({
        className: 'custom-pin',
        html: `
          <div class="relative flex items-center justify-center w-6 h-6">
            <span class="absolute inline-flex h-full w-full rounded-full bg-blue-400 opacity-75 animate-ping"></span>
            <span class="relative inline-flex rounded-full h-4 w-4 bg-blue-500 border-2 border-white shadow-lg"></span>
          </div>
        `,
        iconSize: [24, 24],
        iconAnchor: [12, 12]
      })

      userMarker = L.marker([lat, lng], { icon: userPinIcon }).addTo(mapInstance)
        .bindPopup('<div style="font-family: sans-serif; font-size: 12px; font-weight: bold; color: #3b82f6;">Você está aqui</div>')
        .openPopup()
    },
    (error) => {
      console.error("Erro:", error)
    }
  )
}

onMounted(() => {
  if (!mapElement.value) return

  mapInstance = L.map(mapElement.value, { zoomControl: false }).setView([-8.05, -34.9], 12)
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { maxZoom: 19 }).addTo(mapInstance)
  L.control.zoom({ position: 'topright' }).addTo(mapInstance)

  // Inicializa o grupo de marcadores vazio no mapa
  markersLayer = L.layerGroup().addTo(mapInstance)

  // Dispara a busca no Backend!
  fetchAlerts()
})

onUnmounted(() => {
  if (mapInstance) mapInstance.remove()
})
</script>

<template>
  <div style="position: relative; width: 100vw; height: 100vh; overflow: hidden; background: #0f172a;">

    <div ref="mapElement" style="position: absolute; top: 0; left: 0; right: 0; bottom: 0; z-index: 0;"></div>

    <div
      class="absolute bottom-0 left-0 w-full rounded-t-3xl md:bottom-auto md:left-auto md:top-4 md:right-4 z-[1000] md:w-80 bg-slate-900/95 backdrop-blur-md text-white p-5 md:rounded-xl shadow-[0_-10px_40px_rgba(0,0,0,0.3)] md:shadow-2xl border-t border-slate-700 md:border flex flex-col max-h-[45vh] md:max-h-[90vh] transition-all duration-300">

      <div class="w-12 h-1.5 bg-slate-600 rounded-full mx-auto mb-4 md:hidden shrink-0"></div>

      <div class="flex items-center justify-between mb-4 shrink-0">
        <h1 class="text-xl font-bold tracking-wider">🛰️ SkyRadar</h1>
        <span class="animate-pulse flex h-3 w-3 rounded-full"
          :class="riskAreas.length > 0 ? 'bg-green-500' : 'bg-red-500'"></span>
      </div>

      <p class="hidden md:block text-xs text-slate-400 mb-4 uppercase tracking-widest font-semibold shrink-0">
        Monitoramento Recife
      </p>

      <div class="flex gap-2 md:mb-6 mb-3 shrink-0 text-[10px] uppercase font-bold text-slate-400 flex-wrap">
        <div class="flex items-center gap-1">
          <div class="w-2 h-2 rounded-full bg-green-500"></div> Normal
        </div>
        <div class="flex items-center gap-1">
          <div class="w-2 h-2 rounded-full bg-yellow-400"></div> Atenção
        </div>
        <div class="flex items-center gap-1">
          <div class="w-2 h-2 rounded-full bg-orange-500"></div> Alerta
        </div>
        <div class="flex items-center gap-1">
          <div class="w-2 h-2 rounded-full bg-red-500"></div> Risco
        </div>
      </div>

      <hr class="border-slate-700 mb-3 md:mb-4 shrink-0">

      <div class="flex-1 overflow-y-auto pr-1 space-y-3 custom-scrollbar">
        <h2 class="text-xs text-slate-400 uppercase tracking-widest font-semibold sticky top-0 bg-slate-900/95 py-1">
          Ocorrências Críticas
        </h2>

        <div v-if="riskAreas.length === 0" class="text-center py-4 text-slate-500 text-xs animate-pulse">
          Buscando dados no satélite...
        </div>

        <div v-for="alert in criticalAlerts" :key="alert.id"
          class="p-3 rounded-lg border border-slate-700 bg-slate-800/50 hover:bg-slate-800 transition-colors">
          <div class="flex items-center gap-2 mb-1">
            <div class="w-2 h-2 rounded-full" :class="alert.level === 'red' ? 'bg-red-500' : 'bg-orange-500'"></div>
            <p class="text-sm font-bold">{{ alert.name }}</p>
          </div>
          <p class="text-xs text-slate-400 pl-4">{{ alert.desc }}</p>
        </div>
      </div>
    </div>

    <div
      class="absolute top-4 left-4 md:top-auto md:bottom-6 md:left-1/2 md:-translate-x-1/2 z-[1000] flex flex-col md:flex-row gap-3">
      <button @click="locateUser"
        class="bg-blue-600/90 backdrop-blur-md text-white px-4 py-2 md:px-5 rounded-full border border-blue-500 shadow-xl hover:bg-blue-700 transition-all font-semibold text-xs md:text-sm flex items-center justify-center gap-2">
        🎯 <span class="hidden md:inline">Onde Estou?</span>
      </button>
      <button @click="resetView"
        class="bg-slate-900/90 backdrop-blur-md text-white px-4 py-2 md:px-5 rounded-full border border-slate-700 shadow-xl hover:bg-slate-800 transition-all font-semibold text-xs md:text-sm flex items-center justify-center gap-2">
        📍 <span class="hidden md:inline">Visão Geral</span>
      </button>
    </div>

  </div>
</template>

<style>
.custom-scrollbar::-webkit-scrollbar {
  width: 4px;
}

.custom-scrollbar::-webkit-scrollbar-track {
  background: rgba(30, 41, 59, 0.5);
  border-radius: 4px;
}

.custom-scrollbar::-webkit-scrollbar-thumb {
  background: rgba(71, 85, 105, 0.8);
  border-radius: 4px;
}

.custom-pin {
  background: transparent !important;
  border: none !important;
}
</style>