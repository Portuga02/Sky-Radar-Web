<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

const mapElement = ref(null)
let mapInstance = null

// Transformamos o Mock em um estado reativo do Vue
const riskAreas = ref([
  { id: 1, name: 'Marco Zero', lat: -8.0631, lng: -34.8711, level: 'green', desc: 'Vias liberadas. Sem acúmulo de água.' },
  { id: 2, name: 'Viaduto da Caxangá', lat: -8.0434, lng: -34.9332, level: 'yellow', desc: 'Atenção: Fluxo lento, risco moderado.' },
  { id: 3, name: 'Av. Mascarenhas de Morais', lat: -8.1068, lng: -34.9126, level: 'orange', desc: 'Alerta: Ponto de alagamento confirmando.' },
  { id: 4, name: 'Dois Irmãos / Macaxeira', lat: -8.0142, lng: -34.9458, level: 'red', desc: 'Risco Extremo: Via interditada. Risco de deslizamento.' }
])

// Filtramos apenas os alertas críticos para mostrar na lista lateral
const criticalAlerts = computed(() => {
  return riskAreas.value.filter(area => area.level === 'red' || area.level === 'orange')
})

const alertColors = {
  green: '#22c55e',
  yellow: '#facc15',
  orange: '#f97316',
  red: '#ef4444'
}

// Função para recentralizar o mapa
const resetView = () => {
  if (mapInstance) {
    mapInstance.flyTo([-8.05, -34.9], 12, {
      duration: 1.5 // Faz uma animação suave de voo
    })
  }
}

onMounted(() => {
  if (!mapElement.value) return

  mapInstance = L.map(mapElement.value, { zoomControl: false }).setView([-8.05, -34.9], 12)

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 19,
    attribution: '© OpenStreetMap - SkyRadar'
  }).addTo(mapInstance)

  // Movemos o controle de zoom para o canto inferior direito para não bater no nosso painel
  L.control.zoom({ position: 'bottomright' }).addTo(mapInstance)

  // Renderizar os marcadores baseados na nossa variável reativa
  riskAreas.value.forEach(area => {
    L.circleMarker([area.lat, area.lng], {
      radius: window.innerWidth < 768 ? 8 : 12, // Um pouco maior no desktop
      fillColor: alertColors[area.level],
      color: '#ffffff',
      weight: 2,
      opacity: 1,
      fillOpacity: 0.9
    })
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
      
      <!-- Cabeçalho -->
      <div class="flex items-center justify-between mb-4 shrink-0">
        <h1 class="text-xl font-bold tracking-wider">🛰️ SkyRadar</h1>
        <span class="animate-pulse flex h-3 w-3 rounded-full bg-red-500"></span>
      </div>

      <p class="text-xs text-slate-400 mb-4 uppercase tracking-widest font-semibold shrink-0">
        Monitoramento Recife
      </p>

      <!-- Legenda Simplificada -->
      <div class="flex gap-2 mb-6 shrink-0 text-[10px] uppercase font-bold text-slate-400">
        <div class="flex items-center gap-1"><div class="w-2 h-2 rounded-full bg-green-500"></div> Normal</div>
        <div class="flex items-center gap-1"><div class="w-2 h-2 rounded-full bg-yellow-400"></div> Atenção</div>
        <div class="flex items-center gap-1"><div class="w-2 h-2 rounded-full bg-orange-500"></div> Alerta</div>
        <div class="flex items-center gap-1"><div class="w-2 h-2 rounded-full bg-red-500"></div> Risco</div>
      </div>

      <!-- Divisória -->
      <hr class="border-slate-700 mb-4 shrink-0">

      <!-- Feed de Alertas Críticos (Rolável) -->
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

    <!-- Botão Flutuante de Recentralizar -->
    <button @click="resetView" 
            class="absolute bottom-6 left-1/2 -translate-x-1/2 z-[1000] bg-slate-900/90 backdrop-blur-md text-white px-6 py-2 rounded-full border border-slate-700 shadow-xl hover:bg-slate-800 transition-all font-semibold text-sm flex items-center gap-2">
      📍 Centralizar Visão
    </button>

  </div>
</template>

<style>
/* Estilizando a barra de rolagem do painel para ficar fina e escura */
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
</style>