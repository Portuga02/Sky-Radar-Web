<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

const mapElement = ref(null)
let mapInstance = null
let userMarker = null
let markersLayer = null
let dynamicMarkersLayer = null // Camada para os pinos temporários do radar de raio expandido

const riskAreas = ref([])
const activeFilter = ref('all')
const selectedArea = ref(null)

const criticalAlerts = computed(() => {
  let list = riskAreas.value.filter(area => area.level === 'red' || area.level === 'orange')
  if (activeFilter.value !== 'all') {
    list = riskAreas.value.filter(area => area.level === activeFilter.value && (area.level === 'red' || area.level === 'orange'))
  }
  return list
})

const alertColors = {
  green: '#22c55e',
  yellow: '#facc15',
  orange: '#f97316',
  red: '#ef4444',
  blue: '#3b82f6'
}

const fetchAlerts = async () => {
  try {
    const response = await fetch('http://localhost:8000/api/alerts')
    if (!response.ok) throw new Error('Falha ao conectar')
    riskAreas.value = await response.json()
    renderMarkers()
  } catch (error) {
    console.error("Erro:", error)
  }
}

const toggleFilter = (level) => {
  activeFilter.value = activeFilter.value === level ? 'all' : level
  renderMarkers()
}

const fecharCard = () => {
  selectedArea.value = null
  if (dynamicMarkersLayer) {
    dynamicMarkersLayer.clearLayers()
  }
}

const renderMarkers = () => {
  if (!markersLayer) return
  markersLayer.clearLayers()

  const areasToDraw = activeFilter.value === 'all'
    ? riskAreas.value
    : riskAreas.value.filter(a => a.level === activeFilter.value)

  areasToDraw.forEach(area => {
    const pinIcon = L.divIcon({
      className: 'custom-pin',
      html: `
        <div class="relative flex items-center justify-center w-8 h-8 drop-shadow-xl transition-all duration-300 hover:scale-110">
          <svg viewBox="0 0 24 24" fill="${alertColors[area.level]}" stroke="white" stroke-width="2" class="w-full h-full">
            <path stroke-linecap="round" stroke-linejoin="round" d="M15 10.5a3 3 0 11-6 0 3 3 0 016 0z" />
            <path stroke-linecap="round" stroke-linejoin="round" d="M19.5 10.5c0 7.142-7.5 11.25-7.5 11.25S4.5 17.642 4.5 10.5a7.5 7.5 0 1115 0z" />
          </svg>
          ${area.level === 'red' || area.level === 'orange' ? '<span class="absolute top-[25%] w-2.5 h-2.5 bg-white rounded-full animate-ping opacity-75"></span>' : ''}
        </div>
      `,
      iconSize: [32, 32],
      iconAnchor: [16, 32],
    })

    const marker = L.marker([area.lat, area.lng], { icon: pinIcon })

    marker.on('click', (e) => {
      L.DomEvent.stopPropagation(e)
      fecharCard()
      selectedArea.value = area
      if (mapInstance) {
        mapInstance.flyTo([area.lat, area.lng], 15, { duration: 1.5 })
      }
    })

    markersLayer.addLayer(marker)
  })
}

// 🎯 NOVA FUNÇÃO ATUALIZADA: Varredura de múltiplos pontos em um raio maior
const inspecionarCoordenadaExpandida = async (lat, lng) => {
  try {
    if (dynamicMarkersLayer) dynamicMarkersLayer.clearLayers()

    // Configuramos uma matriz de 5 pontos (Centro + 4 pontos cardeais ao redor)
    const malhaRadar = [
      { name: 'Ponto Central', dLat: 0, dLng: 0 },
      { name: 'Vetor Norte', dLat: 0.012, dLng: 0 },
      { name: 'Vetor Sul', dLat: -0.012, dLng: 0 },
      { name: 'Vetor Leste', dLat: 0, dLng: 0.015 },
      { name: 'Vetor Oeste', dLat: 0, dLng: -0.015 }
    ]

    const lats = malhaRadar.map(p => lat + p.dLat).join(',')
    const lngs = malhaRadar.map(p => lng + p.dLng).join(',')

    // Chamada em lote otimizada para o satélite
    const response = await fetch(`https://api.open-meteo.com/v1/forecast?latitude=${lats}&longitude=${lngs}&current=precipitation,temperature_2m`)
    const dataList = await response.json()

    const resultados = Array.isArray(dataList) ? dataList : [dataList]

    resultados.forEach((data, index) => {
      const config = malhaRadar[index]
      const pontoLat = lat + config.dLat
      const pontoLng = lng + config.dLng

      const chuva = data.current.precipitation || 0
      const temp = data.current.temperature_2m || '--'

      let nivel = 'green'
      let desc = `${config.name} analisado via varredura de raio. Condições normais.`

      if (chuva > 0 && chuva <= 5) {
        nivel = 'yellow'
        desc = `Atenção no ${config.name}: Chuva leve (${chuva}mm) detectada.`
      } else if (chuva > 5 && chuva <= 15) {
        nivel = 'orange'
        desc = `Alerta no ${config.name}: Chuva moderada (${chuva}mm) com possível acúmulo.`
      } else if (chuva > 15) {
        nivel = 'red'
        desc = `Risco Crítico no ${config.name}: Chuva forte (${chuva}mm) detectada por satélite.`
      }

      const areaObj = {
        id: 'dynamic-' + index + '-' + Date.now(),
        name: config.name === 'Ponto Central' ? 'Área Inspecionada' : config.name,
        lat: pontoLat,
        lng: pontoLng,
        level: nivel,
        desc: desc,
        temp: temp,
        rain: chuva
      }

      // O ponto central abre no painel lateral de vidro automaticamente
      if (index === 0) {
        selectedArea.value = areaObj
      }

      const corMapeada = alertColors[nivel]

      // Cria o marcador com a cor dinâmica do nível de chuva detectado!
      const radarPinIcon = L.divIcon({
        className: 'custom-pin',
        html: `
          <div class="relative flex items-center justify-center w-8 h-8 drop-shadow-md transition-all duration-300 hover:scale-125">
            <span class="absolute inline-flex h-full w-full rounded-full border border-dashed opacity-60 animate-[spin_4s_linear_infinite]" style="border-color: ${corMapeada}"></span>
            <svg viewBox="0 0 24 24" fill="${corMapeada}" stroke="white" stroke-width="2" class="w-7 h-7">
              <path stroke-linecap="round" stroke-linejoin="round" d="M15 10.5a3 3 0 11-6 0 3 3 0 016 0z" />
              <path stroke-linecap="round" stroke-linejoin="round" d="M19.5 10.5c0 7.142-7.5 11.25-7.5 11.25S4.5 17.642 4.5 10.5a7.5 7.5 0 1115 0z" />
            </svg>
          </div>
        `,
        iconSize: [32, 32],
        iconAnchor: [16, 32]
      })

      const marker = L.marker([pontoLat, pontoLng], { icon: radarPinIcon })

      marker.on('click', (e) => {
        L.DomEvent.stopPropagation(e) // Impede o mapa de interpretar como clique fora
        selectedArea.value = areaObj
        if (mapInstance) mapInstance.flyTo([pontoLat, pontoLng], 15, { duration: 1.2 })
      })

      if (dynamicMarkersLayer) {
        dynamicMarkersLayer.addLayer(marker)
      }
    })

    if (mapInstance) {
      mapInstance.flyTo([lat, lng], 13, { duration: 1.5 })
    }
  } catch (error) {
    console.error("Erro na varredura de raio estendido:", error)
  }
}

const resetView = () => {
  if (mapInstance) {
    mapInstance.flyTo([-8.05, -34.9], 12, { duration: 1.5 })
    fecharCard()
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

      if (mapInstance) {
        mapInstance.flyTo([lat, lng], 15, { duration: 1.5 })
      }

      if (userMarker && mapInstance) {
        mapInstance.removeLayer(userMarker)
      }

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

      if (mapInstance) {
        userMarker = L.marker([lat, lng], { icon: userPinIcon }).addTo(mapInstance)
      }
    },
    (error) => {
      console.error("Erro ao obter localização:", error)
    },
    { enableHighAccuracy: true, timeout: 10000 }
  )
}

onMounted(async () => {
  if (!mapElement.value) return
  mapInstance = L.map(mapElement.value, { zoomControl: false }).setView([-8.05, -34.9], 12)
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { maxZoom: 19 }).addTo(mapInstance)
  L.control.zoom({ position: 'topright' }).addTo(mapInstance)

  markersLayer = L.layerGroup().addTo(mapInstance)
  dynamicMarkersLayer = L.layerGroup().addTo(mapInstance) // Ativa a camada do radar estendido

  mapInstance.on('click', async (e) => {
    fecharCard()
    const { lat, lng } = e.latlng
    await inspecionarCoordenadaExpandida(lat, lng)
  })

  await fetchAlerts()
  locateUser()
})

onUnmounted(() => {
  if (mapInstance) mapInstance.remove()
})
</script>

<template>
  <div style="position: relative; width: 100vw; height: 100vh; overflow: hidden; background: #0f172a;">
    <div ref="mapElement" style="position: absolute; top: 0; left: 0; right: 0; bottom: 0; z-index: 0;"></div>

    <transition name="fade-slide">
      <div v-if="selectedArea"
        class="absolute top-4 right-4 md:top-6 md:right-6 z-[2000] w-[90%] md:w-80 bg-slate-900/90 backdrop-blur-xl text-white p-6 rounded-3xl shadow-[0_20px_50px_rgba(0,0,0,0.5)] border border-slate-700 mx-auto left-0 right-0 md:left-auto md:mx-0">

        <button @click="fecharCard"
          class="absolute top-4 right-4 text-slate-400 hover:text-white transition-colors bg-slate-800 rounded-full w-8 h-8 flex items-center justify-center">
          ✕
        </button>

        <h3 class="text-2xl font-black tracking-tighter pr-8">{{ selectedArea.name }}</h3>

        <div
          class="mt-2 inline-flex items-center gap-2 px-3 py-1 rounded-lg text-[10px] font-bold uppercase tracking-widest border"
          :style="{ borderColor: alertColors[selectedArea.level], backgroundColor: alertColors[selectedArea.level] + '20', color: alertColors[selectedArea.level] }">
          <span class="w-2 h-2 rounded-full animate-pulse"
            :style="{ backgroundColor: alertColors[selectedArea.level] }"></span>
          {{ selectedArea.level }}
        </div>

        <div class="flex justify-around items-center mt-6 p-4 bg-slate-800/50 rounded-2xl border border-slate-700/50">
          <div class="flex flex-col items-center">
            <span class="text-3xl filter drop-shadow-md">🌡️</span>
            <span class="text-xl font-black mt-1">{{ selectedArea.temp ?? '--' }}<span
                class="text-sm text-slate-400">°C</span></span>
            <span class="text-[9px] text-slate-500 uppercase font-bold tracking-widest">Temp</span>
          </div>
          <div class="w-px h-12 bg-slate-700"></div>
          <div class="flex flex-col items-center">
            <span class="text-3xl filter drop-shadow-md" :class="selectedArea.rain > 0 ? 'animate-bounce' : ''">
              {{ selectedArea.rain > 0 ? '🌧️' : '☀️' }}
            </span>
            <span class="text-xl font-black mt-1 text-blue-400">{{ selectedArea.rain ?? '0' }}<span
                class="text-sm text-slate-400">mm</span></span>
            <span class="text-[9px] text-slate-500 uppercase font-bold tracking-widest">Chuva</span>
          </div>
        </div>

        <p class="text-xs text-slate-300 mt-5 font-medium leading-relaxed bg-slate-800 p-4 rounded-xl border-l-4"
          :style="{ borderColor: alertColors[selectedArea.level] }">
          {{ selectedArea.desc }}
        </p>
      </div>
    </transition>

    <div
      class="absolute bottom-0 left-0 w-full rounded-t-3xl md:bottom-auto md:left-4 md:top-4 z-[1000] md:w-80 bg-slate-900/95 backdrop-blur-md text-white p-5 md:rounded-2xl shadow-[0_-10px_40px_rgba(0,0,0,0.3)] md:shadow-2xl border-t border-slate-700 md:border flex flex-col max-h-[45vh] md:max-h-[90vh] transition-all duration-300">
      <div class="w-12 h-1.5 bg-slate-600 rounded-full mx-auto mb-4 md:hidden shrink-0"></div>

      <div class="flex items-center justify-between mb-4 shrink-0">
        <h1 class="text-xl font-black tracking-wider text-blue-500">🛰️ SKY<span class="text-white">RADAR</span></h1>
        <span class="animate-pulse flex h-3 w-3 rounded-full"
          :class="riskAreas.length > 0 ? 'bg-green-500' : 'bg-red-500'"></span>
      </div>

   <div class="flex gap-2 md:mb-6 mb-3 shrink-0 text-[10px] uppercase font-bold text-slate-400 flex-wrap">
        <button @click="toggleFilter('all')"
          :class="activeFilter === 'all' ? 'opacity-100 scale-105 text-white' : 'opacity-50 hover:opacity-100'"
          class="flex items-center gap-1 transition-all duration-300 mr-2">
          <div class="w-2 h-2 rounded-full bg-slate-400 shadow-[0_0_5px_#94a3b8]"></div> Todos
        </button>

        <button @click="toggleFilter('green')"
          :class="activeFilter === 'green' || activeFilter === 'all' ? 'opacity-100 scale-105 text-white' : 'opacity-25 grayscale hover:opacity-100'"
          class="flex items-center gap-1 transition-all duration-300">
          <div class="w-2 h-2 rounded-full bg-green-500 shadow-[0_0_5px_#22c55e]"></div> Normal
        </button>
        <button @click="toggleFilter('yellow')"
          :class="activeFilter === 'yellow' || activeFilter === 'all' ? 'opacity-100 scale-105 text-white' : 'opacity-25 grayscale hover:opacity-100'"
          class="flex items-center gap-1 transition-all duration-300">
          <div class="w-2 h-2 rounded-full bg-yellow-400 shadow-[0_0_5px_#facc15]"></div> Atenção
        </button>
        <button @click="toggleFilter('orange')"
          :class="activeFilter === 'orange' || activeFilter === 'all' ? 'opacity-100 scale-105 text-white' : 'opacity-25 grayscale hover:opacity-100'"
          class="flex items-center gap-1 transition-all duration-300">
          <div class="w-2 h-2 rounded-full bg-orange-500 shadow-[0_0_5px_#f97316]"></div> Alerta
        </button>
        <button @click="toggleFilter('red')"
          :class="activeFilter === 'red' || activeFilter === 'all' ? 'opacity-100 scale-105 text-white' : 'opacity-25 grayscale hover:opacity-100'"
          class="flex items-center gap-1 transition-all duration-300">
          <div class="w-2 h-2 rounded-full bg-red-500 shadow-[0_0_5px_#ef4444]"></div> Risco
        </button>
      </div>

      <hr class="border-slate-700 mb-3 md:mb-4 shrink-0">

      <div class="flex-1 overflow-y-auto pr-1 space-y-3 custom-scrollbar">
        <h2 class="text-[10px] text-slate-500 uppercase tracking-widest font-black sticky top-0 bg-slate-900/95 py-1">
          Ocorrências Críticas</h2>

        <div v-if="riskAreas.length === 0" class="text-center py-4 text-slate-500 text-xs animate-pulse">Buscando dados
          no satélite...</div>
        <div
          v-if="riskAreas.length > 0 && criticalAlerts.length === 0 && (activeFilter === 'red' || activeFilter === 'orange' || activeFilter === 'all')"
          class="text-center py-4 text-slate-500 text-xs">Nenhuma ocorrência crítica.</div>

        <div v-for="alert in criticalAlerts" :key="alert.id"
          @click="selectedArea = alert; mapInstance.flyTo([alert.lat, alert.lng], 15)"
          class="p-4 rounded-xl border border-slate-700 bg-slate-800/50 hover:bg-slate-700 transition-colors cursor-pointer group">
          <div class="flex items-center gap-2 mb-2">
            <div class="w-2 h-2 rounded-full shadow-lg"
              :class="alert.level === 'red' ? 'bg-red-500 shadow-red-500/50' : 'bg-orange-500 shadow-orange-500/50'">
            </div>
            <p class="text-sm font-black">{{ alert.name }}</p>
          </div>
          <div class="flex justify-between items-end pl-4">
            <p class="text-[10px] text-slate-400 uppercase tracking-widest font-bold">Chuva: <span
                class="text-blue-400">{{ alert.rain }}mm</span></p>
            <span class="text-[10px] text-slate-500 group-hover:text-white transition-colors">VER NO MAPA ➔</span>
          </div>
        </div>
      </div>
    </div>

    <div
      class="absolute top-4 left-4 md:top-auto md:bottom-6 md:left-1/2 md:-translate-x-1/2 z-[1000] flex flex-col md:flex-row gap-3">
      <button @click="locateUser"
        class="bg-blue-600/90 backdrop-blur-md text-white px-4 py-3 md:px-6 rounded-2xl border border-blue-500 shadow-xl hover:bg-blue-700 transition-all font-black text-xs uppercase tracking-widest flex items-center justify-center gap-2">
        🎯 <span class="hidden md:inline">Onde Estou?</span>
      </button>
      <button @click="resetView"
        class="bg-slate-900/90 backdrop-blur-md text-white px-4 py-3 md:px-6 rounded-2xl border border-slate-700 shadow-xl hover:bg-slate-800 transition-all font-black text-xs uppercase tracking-widest flex items-center justify-center gap-2">
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

.fade-slide-enter-active,
.fade-slide-leave-active {
  transition: all 0.4s cubic-bezier(0.16, 1, 0.3, 1);
}

.fade-slide-enter-from {
  opacity: 0;
  transform: translateY(-20px) scale(0.95);
}

.fade-slide-leave-to {
  opacity: 0;
  transform: translateY(-20px) scale(0.95);
}
</style>