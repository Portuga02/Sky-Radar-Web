<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

const mapElement = ref(null)
let mapInstance = null
let userMarker = null // Variável para guardar o pino azul do usuário

// Estado reativo com os nossos dados fictícios
const riskAreas = ref([
  { id: 1, name: 'Marco Zero', lat: -8.0631, lng: -34.8711, level: 'green', desc: 'Vias liberadas. Sem acúmulo de água.' },
  { id: 2, name: 'Viaduto da Caxangá', lat: -8.0434, lng: -34.9332, level: 'yellow', desc: 'Atenção: Fluxo lento, risco moderado.' },
  { id: 3, name: 'Av. Mascarenhas de Morais', lat: -8.1068, lng: -34.9126, level: 'orange', desc: 'Alerta: Ponto de alagamento confirmando.' },
  { id: 4, name: 'Dois Irmãos / Macaxeira', lat: -8.0142, lng: -34.9458, level: 'red', desc: 'Risco Extremo: Via interditada. Risco de deslizamento.' }
])

const criticalAlerts = computed(() => {
  return riskAreas.value.filter(area => area.level === 'red' || area.level === 'orange')
})

const alertColors = {
  green: '#22c55e',
  yellow: '#facc15',
  orange: '#f97316',
  red: '#ef4444'
}

// Retorna a visão para a região central de Recife
const resetView = () => {
  if (mapInstance) {
    mapInstance.flyTo([-8.05, -34.9], 12, { duration: 1.5 })
  }
}

// NOVO: Função de Geolocalização (Busca o GPS do navegador)
const locateUser = () => {
  if (!navigator.geolocation) {
    alert("Seu navegador não suporta geolocalização.")
    return
  }

  // Pede a localização
  navigator.geolocation.getCurrentPosition(
    (position) => {
      const lat = position.coords.latitude
      const lng = position.coords.longitude

      // Voa para a posição do usuário com bastante zoom (15)
      mapInstance.flyTo([lat, lng], 15, { duration: 1.5 })

      // Se já existir um marcador do usuário, remove antes de criar outro
      if (userMarker) {
        mapInstance.removeLayer(userMarker)
      }

      // Cria o ícone azul pulsante para o GPS do usuário
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

      // Adiciona o pino no mapa
      userMarker = L.marker([lat, lng], { icon: userPinIcon }).addTo(mapInstance)
        .bindPopup('<div style="font-family: sans-serif; font-size: 12px; font-weight: bold; color: #3b82f6;">Você está aqui</div>')
        .openPopup()
    },
    (error) => {
      console.error("Erro ao buscar localização:", error)
      alert("Não foi possível encontrar sua localização. Verifique as permissões do navegador.")
    }
  )
}

onMounted(() => {
  if (!mapElement.value) return

  mapInstance = L.map(mapElement.value, { zoomControl: false }).setView([-8.05, -34.9], 12)

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 19,
    attribution: '© OpenStreetMap - SkyRadar'
  }).addTo(mapInstance)

  L.control.zoom({ position: 'bottomright' }).addTo(mapInstance)

  // Renderiza os pinos de risco
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

    L.marker([area.lat, area.lng], { icon: pinIcon })
      .addTo(mapInstance)
      .bindPopup(`
        <div style="font-family: sans-serif;">
          <strong style="color: ${alertColors[area.level]}; text-transform: uppercase; font-size: 10px;">
            Nível: ${area.level}
          </strong><br>
          <b>${area.name}</b><br>
          <span style="font-size: 12px; color: #555;">${area.desc}</span>
        </div>
      `)
  })
})

onUnmounted(() => {
  if (mapInstance) mapInstance.remove()
})
</script>

<template>
  <div style="position: relative; width: 100vw; height: 100vh; overflow: hidden; background: #0f172a;">
    
    <div ref="mapElement" style="position: absolute; top: 0; left: 0; right: 0; bottom: 0; z-index: 0;"></div>

    <!-- Painel de Controle Principal -->
    <div class="absolute top-4 right-4 z-[1000] w-80 bg-slate-900/90 backdrop-blur-md text-white p-5 rounded-xl shadow-2xl border border-slate-700 flex flex-col max-h-[90vh]">
      
      <div class="flex items-center justify-between mb-4 shrink-0">
        <h1 class="text-xl font-bold tracking-wider">🛰️ SkyRadar</h1>
        <span class="animate-pulse flex h-3 w-3 rounded-full bg-red-500"></span>
      </div>

      <p class="text-xs text-slate-400 mb-4 uppercase tracking-widest font-semibold shrink-0">
        Monitoramento Recife
      </p>

      <div class="flex gap-2 mb-6 shrink-0 text-[10px] uppercase font-bold text-slate-400">
        <div class="flex items-center gap-1"><div class="w-2 h-2 rounded-full bg-green-500"></div> Normal</div>
        <div class="flex items-center gap-1"><div class="w-2 h-2 rounded-full bg-yellow-400"></div> Atenção</div>
        <div class="flex items-center gap-1"><div class="w-2 h-2 rounded-full bg-orange-500"></div> Alerta</div>
        <div class="flex items-center gap-1"><div class="w-2 h-2 rounded-full bg-red-500"></div> Risco</div>
      </div>

      <hr class="border-slate-700 mb-4 shrink-0">

      <div class="flex-1 overflow-y-auto pr-1 space-y-3 custom-scrollbar">
        <h2 class="text-xs text-slate-400 uppercase tracking-widest font-semibold sticky top-0 bg-slate-900/90 py-1">
          Ocorrências Críticas
        </h2>
        
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

    <!-- Container dos Botões de Ação na base da tela -->
    <div class="absolute bottom-6 left-1/2 -translate-x-1/2 z-[1000] flex gap-3">
      
      <!-- Botão Onde Estou (Geolocalização) -->
      <button @click="locateUser" 
              class="bg-blue-600/90 backdrop-blur-md text-white px-5 py-2 rounded-full border border-blue-500 shadow-xl hover:bg-blue-700 transition-all font-semibold text-sm flex items-center gap-2">
        🎯 Onde Estou?
      </button>

      <!-- Botão Centralizar Visão -->
      <button @click="resetView" 
              class="bg-slate-900/90 backdrop-blur-md text-white px-5 py-2 rounded-full border border-slate-700 shadow-xl hover:bg-slate-800 transition-all font-semibold text-sm flex items-center gap-2">
        📍 Visão Geral
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