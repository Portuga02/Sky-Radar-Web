<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

const mapElement = ref(null)
let mapInstance = null
let userMarker = null
let markersLayer = null
let dynamicMarkersLayer = null

const riskAreas = ref([])
const activeFilter = ref('all')
const selectedArea = ref(null)
const searchQuery = ref('')
const searchResults = ref([])
const localRadarData = ref([])

const alertColors = {
  green: '#22c55e',
  yellow: '#facc15',
  orange: '#f97316',
  red: '#ef4444',
  blue: '#3b82f6'
}

const nivelNomes = {
  green: 'Normal',
  yellow: 'Atenção',
  orange: 'Alerta',
  red: 'Risco'
}

const criticalAlerts = computed(() => {
  let list = []
  if (riskAreas.value && riskAreas.value.length > 0) list = [...riskAreas.value]

  if (localRadarData.value && localRadarData.value.length > 0) {
    const existingIds = new Set(list.map(a => a.id))
    localRadarData.value.forEach(area => {
      if (!existingIds.has(area.id)) list.push(area)
    })
  }

  if (activeFilter.value !== 'all') list = list.filter(area => area.level === activeFilter.value)
  const pesoAlerta = { red: 4, orange: 3, yellow: 2, green: 1, blue: 0 }
  return list.sort((a, b) => (pesoAlerta[b.level] || 0) - (pesoAlerta[a.level] || 0))
})

const fetchAlerts = async () => {
  try {
    const response = await fetch('http://localhost:8000/api/alerts')
    if (response.ok) {
      const dbData = await response.json()
      
      // 🎯 NORMALIZADOR DE DADOS SÊNIOR
      // Intercepta os dados do banco e força o nível visual (cor) a obedecer a matemática da chuva
      riskAreas.value = dbData.map(area => {
        let nivelCorrigido = 'green'; // Padrão seguro
        const chuvaDoBanco = parseFloat(area.rain) || 0;

        if (chuvaDoBanco > 0 && chuvaDoBanco <= 5) nivelCorrigido = 'yellow';
        else if (chuvaDoBanco > 5 && chuvaDoBanco <= 15) nivelCorrigido = 'orange';
        else if (chuvaDoBanco > 15) nivelCorrigido = 'red';

        // Retorna o objeto do alerta corrigindo o 'level' para bater com a cor exata
        return { 
          ...area, 
          level: nivelCorrigido 
        }
      })
      
      renderMarkers()
    }
  } catch (error) {
    console.warn("⚠️ Laravel indisponível. Operando em modo local.")
  }
}

const toggleFilter = (level) => {
  activeFilter.value = activeFilter.value === level ? 'all' : level
  renderMarkers()
}

const fecharCard = () => {
  selectedArea.value = null

}

const focarNoAlerta = (alert) => {

  selectedArea.value = alert;
  if (mapInstance) {
    mapInstance.flyTo([alert.lat, alert.lng], 15, { duration: 1.2 });
  }
  const ehVarreduraLocal = localRadarData.value.some(a => a.id === alert.id);

  if (ehVarreduraLocal && dynamicMarkersLayer) {
    dynamicMarkersLayer.clearLayers(); 

    const corMapeada = alertColors[alert.level] || '#10B981';
    const radarPinIcon = L.divIcon({
      className: 'marker-custom-radar', 
      html: `
        <div style="position: relative; width: 42px; height: 42px; display: flex; align-items: center; justify-content: center;">
          <span style="position: absolute; width: 100%; height: 100%; border-radius: 50%; border: 2px dashed ${corMapeada}; opacity: 0.7; animation: spin 6s linear infinite;"></span>
          <svg viewBox="0 0 24 24" fill="${corMapeada}" stroke="white" stroke-width="1.5" style="width: 28px; height: 28px; z-index: 10;">
            <path stroke-linecap="round" stroke-linejoin="round" d="M15 10.5a3 3 0 11-6 0 3 3 0 016 0z" />
            <path stroke-linecap="round" stroke-linejoin="round" d="M19.5 10.5c0 7.142-7.5 11.25-7.5 11.25S4.5 17.642 4.5 10.5a7.5 7.5 0 1115 0z" />
          </svg>
          <div style="position: absolute; top: -2px; right: -2px; width: 18px; height: 18px; background-color: rgba(15, 23, 42, 0.9); border-radius: 50%; border: 1px solid #475569; display: flex; align-items: center; justify-content: center; padding: 2px; z-index: 20; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.5);">
            <img src="https://assets.hgbrasil.com/weather/icons/conditions/${alert.iconSlug || 'cloudly_day'}.svg" style="width: 100%; height: 100%; object-fit: contain;" />
          </div>
        </div>
      `,
      iconSize: [42, 42],
      iconAnchor: [21, 42]
    });

    const marker = L.marker([alert.lat, alert.lng], { icon: radarPinIcon });
    marker.on('click', (e) => {
      L.DomEvent.stopPropagation(e);
      selectedArea.value = alert;
    });
    dynamicMarkersLayer.addLayer(marker);
  }
}
const renderMarkers = () => {
  if (!markersLayer) return
  markersLayer.clearLayers()

  const areasToDraw = activeFilter.value === 'all' ? riskAreas.value : riskAreas.value.filter(a => a.level === activeFilter.value)

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
      if (mapInstance) mapInstance.flyTo([area.lat, area.lng], 15, { duration: 1.5 })
    })
    markersLayer.addLayer(marker)
  })
}

const inspecionarCoordenadaExpandida = async (lat, lng, nomeCentral = 'Área Inspecionada') => {
  try {
    if (dynamicMarkersLayer) dynamicMarkersLayer.clearLayers(); else dynamicMarkersLayer = L.layerGroup().addTo(mapInstance);

    const HGBRASIL_KEY = '631a8bba';
    const urlHG = `https://api.hgbrasil.com/weather?format=json-cors&key=${HGBRASIL_KEY}&lat=${lat}&lon=${lng}`;

    let tempApi = '--', chuvaApi = 0, condicaoDesc = 'Buscando condições...', iconSlug = 'cloudly_day';

    try {
      const response = await fetch(urlHG);
      if (response.ok) {
        const data = await response.json();
        if (data.results) {
          tempApi = data.results.temp || '--';
          condicaoDesc = data.results.description || 'Condições normais';
          iconSlug = data.results.condition_slug || 'cloudly_day';
          const descLower = condicaoDesc.toLowerCase();
          if (descLower.includes('tempestade') || descLower.includes('forte')) chuvaApi = 25;
          else if (descLower.includes('chuva') || descLower.includes('chuvisco')) chuvaApi = 12;
          else if (descLower.includes('nublado') || descLower.includes('tempo limpo')) chuvaApi = 0;
        }
      }
    } catch (apiError) { console.warn("⚠️ HG Brasil falhou.", apiError); }

    const malhaRadar = [
      { name: 'Ponto Central', dLat: 0, dLng: 0 },
      { name: 'Vetor Norte', dLat: 0.008, dLng: 0 },
      { name: 'Vetor Sul', dLat: -0.008, dLng: 0 },
      { name: 'Vetor Leste', dLat: 0, dLng: 0.010 },
      { name: 'Vetor Oeste', dLat: 0, dLng: -0.010 }
    ];

    malhaRadar.forEach((config, index) => {
      const pontoLat = lat + config.dLat, pontoLng = lng + config.dLng;
      let chuvaDoPonto = index === 0 ? chuvaApi : Math.max(0, chuvaApi - Math.floor(Math.random() * 3));

      let nivel = 'green';
      if (chuvaDoPonto > 0 && chuvaDoPonto <= 5) nivel = 'yellow';
      else if (chuvaDoPonto > 5 && chuvaDoPonto <= 15) nivel = 'orange';
      else if (chuvaDoPonto > 15) nivel = 'red';

      const areaObj = {
        id: 'dynamic-' + index + '-' + Date.now(),
        name: config.name === 'Ponto Central' ? nomeCentral : config.name,
        lat: pontoLat, lng: pontoLng, level: nivel,
        desc: `${config.name === 'Ponto Central' ? nomeCentral : config.name} (HG Brasil). ${condicaoDesc}`,
        temp: tempApi, rain: chuvaDoPonto, iconSlug: iconSlug
      };

      if (index === 0) selectedArea.value = areaObj;
      localRadarData.value.unshift(areaObj);

      const corMapeada = alertColors[nivel] || '#10B981';
      const radarPinIcon = L.divIcon({
        className: 'marker-custom-radar',
        html: `
          <div style="position: relative; width: 42px; height: 42px; display: flex; align-items: center; justify-content: center;">
            <span style="position: absolute; width: 100%; height: 100%; border-radius: 50%; border: 2px dashed ${corMapeada}; opacity: 0.7; animation: spin 6s linear infinite;"></span>
            <svg viewBox="0 0 24 24" fill="${corMapeada}" stroke="white" stroke-width="1.5" style="width: 28px; height: 28px; z-index: 10;">
              <path stroke-linecap="round" stroke-linejoin="round" d="M15 10.5a3 3 0 11-6 0 3 3 0 016 0z" />
              <path stroke-linecap="round" stroke-linejoin="round" d="M19.5 10.5c0 7.142-7.5 11.25-7.5 11.25S4.5 17.642 4.5 10.5a7.5 7.5 0 1115 0z" />
            </svg>
            <div style="position: absolute; top: -2px; right: -2px; width: 18px; height: 18px; background-color: rgba(15, 23, 42, 0.9); border-radius: 50%; border: 1px solid #475569; display: flex; align-items: center; justify-content: center; padding: 2px; z-index: 20; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.5);">
              <img src="https://assets.hgbrasil.com/weather/icons/conditions/${iconSlug}.svg" style="width: 100%; height: 100%; object-fit: contain;" />
            </div>
          </div>
        `,
        iconSize: [42, 42], iconAnchor: [21, 42]
      });

      const marker = L.marker([pontoLat, pontoLng], { icon: radarPinIcon });
      marker.on('click', (e) => {
        L.DomEvent.stopPropagation(e);
        selectedArea.value = areaObj;
        if (mapInstance) mapInstance.flyTo([pontoLat, pontoLng], 15, { duration: 1.2 });
      });
      dynamicMarkersLayer.addLayer(marker);
    });

    if (mapInstance) mapInstance.flyTo([lat, lng], 14, { duration: 1.5 });
  } catch (error) { console.error("Erro na varredura:", error) }
}

const resetView = () => {
  if (mapInstance) { mapInstance.flyTo([-8.05, -34.9], 12, { duration: 1.5 }); fecharCard(); }
}

const buscarLocal = async () => {
  if (!searchQuery.value.trim()) return
  try {
    const url = `https://nominatim.openstreetmap.org/search?format=json&addressdetails=1&limit=5&q=${encodeURIComponent(searchQuery.value)}`;
    const response = await fetch(url, { headers: { 'Accept-Language': 'pt-BR,pt;q=0.9' } });
    if (!response.ok) return
    const data = await response.json();
    if (data && data.length > 0) searchResults.value = data;
    else { searchResults.value = []; alert('Local não encontrado. Tente digitar a cidade e estado.'); }
  } catch (error) { console.error("Erro na busca:", error) }
}

const selecionarLocal = async (local) => {
  searchResults.value = []; searchQuery.value = ''; fecharCard();
  const lat = parseFloat(local.lat), lng = parseFloat(local.lon);
  let nomeFormatado = local.display_name.split(',').slice(0, 3).join(', ');
  await inspecionarCoordenadaExpandida(lat, lng, nomeFormatado);
}

const locateUser = () => {
  if (!navigator.geolocation) return
  navigator.geolocation.getCurrentPosition(
    async (position) => {
      const lat = position.coords.latitude, lng = position.coords.longitude;
      if (mapInstance) mapInstance.flyTo([lat, lng], 15, { duration: 1.5 })
      if (userMarker && mapInstance) mapInstance.removeLayer(userMarker)

      const userPinIcon = L.divIcon({
        className: 'custom-pin',
        html: `
          <div class="relative flex items-center justify-center w-6 h-6">
            <span class="absolute inline-flex h-full w-full rounded-full bg-blue-400 opacity-75 animate-ping"></span>
            <span class="relative inline-flex rounded-full h-4 w-4 bg-blue-500 border-2 border-white shadow-lg"></span>
          </div>
        `,
        iconSize: [24, 24], iconAnchor: [12, 12]
      })

      if (mapInstance) userMarker = L.marker([lat, lng], { icon: userPinIcon }).addTo(mapInstance)
      await inspecionarCoordenadaExpandida(lat, lng, "Sua Localização Atual")
    },
    (error) => console.error(error), { enableHighAccuracy: true, timeout: 10000 }
  )
}

onMounted(async () => {
  if (!mapElement.value || mapInstance) return
  mapInstance = L.map(mapElement.value, { zoomControl: false, trackResize: true }).setView([-8.05, -34.9], 12)
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { maxZoom: 19 }).addTo(mapInstance)
  L.control.zoom({ position: 'topright' }).addTo(mapInstance)

  markersLayer = L.layerGroup().addTo(mapInstance)
  dynamicMarkersLayer = L.layerGroup().addTo(mapInstance)

 mapInstance.on('click', async (e) => {
    fecharCard()
    const latLngCorrigido = e.latlng.wrap()
    const lat = latLngCorrigido.lat
    const lng = latLngCorrigido.lng
    
    let nomeIdentificado = `Coordenada: ${lat.toFixed(2)}, ${lng.toFixed(2)}` 
    
    try {
      const urlReverse = `https://api.bigdatacloud.net/data/reverse-geocode-client?latitude=${lat}&longitude=${lng}&localityLanguage=pt`
      const res = await fetch(urlReverse);
      if (res.ok) {
        const data = await res.json()
        nomeIdentificado = data.locality || data.city || data.principalSubdivision || nomeIdentificado;
      }
    } catch (err) { 
      console.warn("⚠️ Geocodificação falhou."); 
    }

    await inspecionarCoordenadaExpandida(lat, lng, nomeIdentificado)
  })

  await fetchAlerts()
  locateUser()
})

onUnmounted(() => { if (mapInstance) { mapInstance.remove(); mapInstance = null; } })
</script>

<template>
  <main class="relative w-full h-screen overflow-hidden bg-slate-900 font-sans text-slate-100">
    <section ref="mapElement" class="absolute inset-0 z-0"></section>

    <header class="absolute top-4 left-1/2 -translate-x-1/2 z-[2000] w-[92%] md:w-[28rem]">
      <form @submit.prevent="buscarLocal" class="relative flex items-center w-full">
        <input v-model="searchQuery" type="search" placeholder="Digite a cidade (ex: Recife)..."
          class="w-full bg-slate-900/95 backdrop-blur-xl text-white px-5 py-3.5 pr-12 rounded-2xl border border-slate-700 shadow-2xl focus:outline-none focus:border-blue-500 transition-all text-sm placeholder-slate-500">
        <button type="submit" class="absolute right-2 p-2 text-slate-400 hover:text-blue-400 transition-colors">
          <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2.5"
            stroke="currentColor" class="w-5 h-5">
            <path stroke-linecap="round" stroke-linejoin="round"
              d="M21 21l-5.197-5.197m0 0A7.5 7.5 0 105.196 5.196a7.5 7.5 0 0010.607 10.607z" />
          </svg>
        </button>
      </form>

      <transition name="fade-slide">
        <nav v-if="searchResults.length > 0"
          class="absolute top-full left-0 w-full mt-2 bg-slate-900/95 backdrop-blur-xl border border-slate-700 rounded-2xl shadow-2xl overflow-hidden max-h-60 overflow-y-auto custom-scrollbar">
          <ul>
            <li @click="searchResults = []"
              class="px-4 py-2 bg-slate-800 text-xs text-slate-400 text-right cursor-pointer hover:text-white">Fechar ✕
            </li>
            <li v-for="(result, index) in searchResults" :key="index" @click="selecionarLocal(result)"
              class="px-5 py-3 hover:bg-slate-800 cursor-pointer border-b border-slate-800/50 last:border-0 flex flex-col group">
              <span class="text-sm font-bold text-blue-400 group-hover:text-white truncate">{{
                result.display_name.split(',')[0] }}</span>
              <span class="text-[11px] text-slate-400 truncate mt-0.5">{{
                result.display_name.split(',').slice(1).join(', ') }}</span>
            </li>
          </ul>
        </nav>
      </transition>
    </header>

    <transition name="slide-up">
      <aside v-if="selectedArea"
        class="absolute z-[3000] bottom-0 left-0 w-full md:bottom-auto md:top-6 md:right-6 md:left-auto md:w-80 bg-slate-900/95 backdrop-blur-xl text-white p-6 pb-8 md:pb-6 rounded-t-3xl md:rounded-3xl shadow-[0_-20px_50px_rgba(0,0,0,0.5)] md:shadow-2xl border-t md:border border-slate-700">

        <button @click="fecharCard"
          class="absolute top-4 right-4 text-slate-400 hover:text-white bg-slate-800 rounded-full w-8 h-8 flex items-center justify-center">✕</button>

        <h3 class="text-2xl font-black tracking-tighter pr-8 truncate">{{ selectedArea.name }}</h3>

        <div
          class="mt-2 inline-flex items-center gap-2 px-3 py-1 rounded-lg text-[10px] font-bold uppercase tracking-widest border"
          :style="{ borderColor: alertColors[selectedArea.level], backgroundColor: alertColors[selectedArea.level] + '20', color: alertColors[selectedArea.level] }">
          <span class="w-2 h-2 rounded-full animate-pulse"
            :style="{ backgroundColor: alertColors[selectedArea.level] }"></span>
          {{ nivelNomes[selectedArea.level] }}
        </div>

        <article
          class="flex justify-around items-center mt-6 p-4 bg-slate-800/50 rounded-2xl border border-slate-700/50">
          <div class="flex flex-col items-center">
            <span class="text-3xl filter drop-shadow-md">🌡️</span>
            <span class="text-xl font-black mt-1">{{ selectedArea.temp ?? '--' }}<span
                class="text-sm text-slate-400">°C</span></span>
            <span class="text-[9px] text-slate-500 uppercase font-bold tracking-widest">Temp</span>
          </div>
          <div class="w-px h-12 bg-slate-700"></div>
          <div class="flex flex-col items-center">
            <img v-if="selectedArea.iconSlug"
              :src="'https://assets.hgbrasil.com/weather/icons/conditions/' + selectedArea.iconSlug + '.svg'"
              class="w-11 h-11 filter drop-shadow-md object-contain" alt="clima" />
            <span v-else class="text-3xl">☀️</span>
            <span class="text-xl font-black mt-1 text-blue-400">{{ selectedArea.rain ?? '0' }}<span
                class="text-sm text-slate-400">mm</span></span>
            <span class="text-[9px] text-slate-500 uppercase font-bold tracking-widest">Chuva</span>
          </div>
        </article>

        <p class="text-xs text-slate-300 mt-5 font-medium leading-relaxed bg-slate-800 p-4 rounded-xl border-l-4"
          :style="{ borderColor: alertColors[selectedArea.level] }">
          {{ selectedArea.desc }}
        </p>
      </aside>
    </transition>

    <aside
      class="absolute z-[1000] bottom-0 left-0 w-full rounded-t-3xl border-t border-slate-700 bg-slate-900/95 backdrop-blur-md p-5 pb-6 md:pb-5 flex flex-col transition-all duration-300 shadow-[0_-10px_40px_rgba(0,0,0,0.3)]
             max-h-[40vh] md:max-h-[90vh] md:bottom-auto md:top-4 md:left-4 md:w-80 md:rounded-2xl md:border md:shadow-2xl">

      <div class="w-12 h-1.5 bg-slate-600 rounded-full mx-auto mb-4 md:hidden shrink-0"></div>

      <header class="flex items-center justify-between mb-4 shrink-0">
        <h1 class="text-xl font-black tracking-wider text-blue-500">🛰️ SKY<span class="text-white">RADAR</span></h1>
        <span class="animate-pulse flex h-3 w-3 rounded-full"
          :class="criticalAlerts.length > 0 ? 'bg-green-500' : 'bg-slate-500'"></span>
      </header>

      <nav
        class="flex gap-2 md:mb-6 mb-3 shrink-0 text-[10px] uppercase font-bold text-slate-400 overflow-x-auto custom-scrollbar pb-1">
        <button @click="toggleFilter('all')" :class="activeFilter === 'all' ? 'text-white' : 'opacity-50'"
          class="flex items-center gap-1 shrink-0 mr-2">
          <div class="w-2 h-2 rounded-full bg-slate-400"></div> Todos
        </button>
        <button @click="toggleFilter('green')" :class="activeFilter === 'green' ? 'text-white' : 'opacity-50'"
          class="flex items-center gap-1 shrink-0">
          <div class="w-2 h-2 rounded-full bg-green-500"></div> Normal
        </button>
        <button @click="toggleFilter('yellow')" :class="activeFilter === 'yellow' ? 'text-white' : 'opacity-50'"
          class="flex items-center gap-1 shrink-0">
          <div class="w-2 h-2 rounded-full bg-yellow-400"></div> Atenção
        </button>
        <button @click="toggleFilter('orange')" :class="activeFilter === 'orange' ? 'text-white' : 'opacity-50'"
          class="flex items-center gap-1 shrink-0">
          <div class="w-2 h-2 rounded-full bg-orange-500"></div> Alerta
        </button>
        <button @click="toggleFilter('red')" :class="activeFilter === 'red' ? 'text-white' : 'opacity-50'"
          class="flex items-center gap-1 shrink-0">
          <div class="w-2 h-2 rounded-full bg-red-500"></div> Risco
        </button>
      </nav>

      <hr class="border-slate-700 mb-3 md:mb-4 shrink-0">

      <section class="flex-1 overflow-y-auto pr-1 space-y-3 custom-scrollbar">
        <div v-if="criticalAlerts.length === 0" class="text-center py-4 text-slate-500 text-xs animate-pulse">Aguardando
          varredura...</div>
       <article v-for="alert in criticalAlerts" :key="alert.id" 
          @click="focarNoAlerta(alert)"
          class="p-4 rounded-xl border border-slate-700 bg-slate-800/50 hover:bg-slate-700 transition-colors cursor-pointer group">
          <div class="flex items-center gap-2 mb-2">
            <div class="w-2 h-2 rounded-full shadow-lg"
              :style="{ backgroundColor: alertColors[alert.level], boxShadow: `0 0 8px ${alertColors[alert.level]}` }">
            </div>
            <p class="text-sm font-black truncate w-[85%]">{{ alert.name }}</p>
          </div>
          <div class="flex justify-between items-end pl-4">
            <p class="text-[10px] text-slate-400 uppercase tracking-widest font-bold">Chuva: <span
                class="text-blue-400">{{ alert.rain }}mm</span></p>
            <span class="text-[10px] text-slate-500 group-hover:text-white transition-colors">VER NO MAPA ➔</span>
          </div>
        </article>
      </section>
    </aside>

    <nav
      class="absolute z-[1500] top-24 right-4 flex flex-col gap-3 md:top-auto md:bottom-6 md:left-1/2 md:-translate-x-1/2 md:right-auto md:flex-row">
      <button @click="locateUser"
        class="bg-blue-600/90 backdrop-blur-md text-white w-10 h-10 md:w-auto md:px-6 md:py-3 rounded-full md:rounded-2xl border border-blue-500 shadow-xl hover:bg-blue-700 transition-all flex items-center justify-center gap-2"
        title="Onde Estou?">
        <span class="text-lg md:text-base">🎯</span> <span
          class="hidden md:inline font-black text-xs uppercase tracking-widest">Onde Estou?</span>
      </button>
      <button @click="resetView"
        class="bg-slate-900/90 backdrop-blur-md text-white w-10 h-10 md:w-auto md:px-6 md:py-3 rounded-full md:rounded-2xl border border-slate-700 shadow-xl hover:bg-slate-800 transition-all flex items-center justify-center gap-2"
        title="Visão Geral">
        <span class="text-lg md:text-base">📍</span> <span
          class="hidden md:inline font-black text-xs uppercase tracking-widest">Visão Geral</span>
      </button>
    </nav>
  </main>
</template>

<style>
.custom-scrollbar::-webkit-scrollbar {
  width: 4px;
  height: 4px;
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

@keyframes spin {
  from {
    transform: rotate(0deg);
  }

  to {
    transform: rotate(360deg);
  }
}

/* Animação responsiva do Card de Detalhes */
.slide-up-enter-active,
.slide-up-leave-active {
  transition: all 0.4s cubic-bezier(0.16, 1, 0.3, 1);
}

.slide-up-enter-from,
.slide-up-leave-to {
  opacity: 0;
  transform: translateY(100%);
}

@media (min-width: 768px) {

  .slide-up-enter-from,
  .slide-up-leave-to {
    opacity: 0;
    transform: translateY(-20px) scale(0.95);
  }
}

.fade-slide-enter-active,
.fade-slide-leave-active {
  transition: all 0.3s ease;
}

.fade-slide-enter-from,
.fade-slide-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}
</style>