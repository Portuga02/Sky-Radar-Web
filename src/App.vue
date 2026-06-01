<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

const mapElement = ref(null)
let mapInstance = null, userMarker = null, markersLayer = null, dynamicMarkersLayer = null
const isRadarActive = ref(false)
let radarTileLayer = null
const riskAreas = ref([]), activeFilter = ref('all'), selectedArea = ref(null)
const searchQuery = ref(''), searchResults = ref([]), localRadarData = ref([])
const anguloBussola = ref(0)

const alertColors = { green: '#22c55e', yellow: '#facc15', orange: '#f97316', red: '#ef4444', blue: '#3b82f6' }
const nivelNomes = { green: 'Normal', yellow: 'Atenção', orange: 'Alerta', red: 'Risco' }

const criticalAlerts = computed(() => {
  let list = riskAreas.value?.length ? [...riskAreas.value] : []
  if (localRadarData.value?.length) {
    const existingIds = new Set(list.map(a => a.id))
    localRadarData.value.forEach(area => { if (!existingIds.has(area.id)) list.push(area) })
  }
  if (activeFilter.value !== 'all') list = list.filter(a => a.level === activeFilter.value)
  const peso = { red: 4, orange: 3, yellow: 2, green: 1, blue: 0 }
  return list.sort((a, b) => (peso[b.level] || 0) - (peso[a.level] || 0))
})

const toggleRadarLayer = () => {
  isRadarActive.value = !isRadarActive.value

  if (isRadarActive.value && mapInstance) {
    // Adiciona a camada de precipitação do OpenWeather
    // DICA DE SÉNIOR: Nunca exponha a chave no código. Use a variável de ambiente.
    const owmKey = import.meta.env.VITE_OWM_KEY || 'd5e403cc1dc62d5c39e5d208a16f5388'

    radarTileLayer = L.tileLayer(`https://tile.openweathermap.org/map/precipitation_new/{z}/{x}/{y}.png?appid=${owmKey}`, {
      maxZoom: 19,
      opacity: 0.7, // 70% para não tapar as ruas debaixo da chuva
      attribution: '&copy; OpenWeather'
    }).addTo(mapInstance)
  } else {
    // Remove a camada se for desativada
    if (radarTileLayer && mapInstance) {
      mapInstance.removeLayer(radarTileLayer)
      radarTileLayer = null
    }
  }
}
// Função para procurar dados quando clica no mapa ou na barra de pesquisa
const fetchAggregatedData = async (lat, lng) => {
  try {
    // 1. Chamamos a NOSSA API unificada
    const apiUrl = import.meta.env.VITE_API_URL || 'http://localhost:8000';
    const response = await fetch(`${apiUrl}/api/radar-data/${lat}/${lng}`);
    const data = await response.json();

    // 2. Extração Inteligente do Volume de Chuva (Fallback Múltiplo)
    let volumeChuva = 0;

    // Tenta pegar a chuva de 1 hora do OpenWeather
    if (data.chuva_detalhada && data.chuva_detalhada['1h']) {
        volumeChuva = data.chuva_detalhada['1h'];
    } 
    // Se não tiver de 1h, tenta pegar a de 3 horas e faz a média
    else if (data.chuva_detalhada && data.chuva_detalhada['3h']) {
        volumeChuva = +(data.chuva_detalhada['3h'] / 3).toFixed(2);
    } 
    // Se o OpenWeather falhar, pede socorro à HG Brasil
    else if (data.temperatura_base && data.temperatura_base.rain) {
        volumeChuva = data.temperatura_base.rain;
    }

    // Console para depuração: veja no F12 exatamente o que a API respondeu
    console.log(`🌧️ Auditoria Chuva [${lat}, ${lng}]:`, data);

    // 3. Atualizamos o painel lateral com o volume extraído
    selectedArea.value = {
      name: data.temperatura_base.city_name,
      temp: data.temperatura_base.temp + '°C',
      desc: data.temperatura_base.description,
      iconSlug: data.temperatura_base.condition_slug,

      // Combinamos o melhor das duas APIs
      wind: data.vento && data.vento.speed ? `${data.vento.speed} m/s` : data.temperatura_base.wind_speedy,
      rain: volumeChuva,

      // Definimos a cor do alerta com base na chuva real
      level: calcularNivelRisco(volumeChuva)
    };

    // 4. Centralizar o mapa na coordenada clicada
    if (mapInstance) {
      mapInstance.flyTo([lat, lng], 14, { duration: 1.5 });
    }

  } catch (error) {
    console.error("Erro ao comunicar com a API Gateway:", error);
  }
};

// Função simples para calcular o perigo de alagamento com base nos milímetros
const calcularNivelRisco = (chuvaMm) => {
  if (chuvaMm === 0) return 'normal';
  if (chuvaMm > 0 && chuvaMm <= 10) return 'yellow';
  if (chuvaMm > 10 && chuvaMm <= 30) return 'orange';
  return 'red'; // Chuva forte, alto risco de problemas nas infraestruturas urbanas
};
const iniciarBussolaDinamica = () => {
  if (window.DeviceOrientationEvent) {
    window.addEventListener('deviceorientation', (e) => {
      if (e.webkitCompassHeading) anguloBussola.value = e.webkitCompassHeading
      else if (e.alpha !== null) anguloBussola.value = 360 - e.alpha
    }, true)
  }
}

const fetchAlerts = async () => {
  try {
    const baseUrl = import.meta.env.VITE_API_URL || 'https://sky-radar-api-production.up.railway.app';
    const res = await fetch(`${baseUrl}/api/alerts`);

    if (res.ok) {
      riskAreas.value = (await res.json()).map(area => {
        const c = parseFloat(area.rain) || 0;
        return { ...area, level: c > 15 ? 'red' : c > 5 ? 'orange' : c > 0 ? 'yellow' : 'green' };
      });
      renderMarkers();
    }
  } catch (e) {
    console.warn("⚠️ API de alertas indisponível.", e);
  }
}

const toggleFilter = (level) => {
  activeFilter.value = activeFilter.value === level ? 'all' : level
  renderMarkers()
}

const fecharCard = () => { selectedArea.value = null }

const focarNoAlerta = (alert) => {
  selectedArea.value = alert
  if (mapInstance) mapInstance.flyTo([alert.lat, alert.lng], 15, { duration: 1.2 })
  if (localRadarData.value.some(a => a.id === alert.id) && dynamicMarkersLayer) {
    dynamicMarkersLayer.clearLayers()
    const cor = alertColors[alert.level] || '#10B981'
    const icon = L.divIcon({
      className: 'marker-custom-radar',
      html: `<div style="position: relative; width: 42px; height: 42px; display: flex; align-items: center; justify-content: center;"><span style="position: absolute; width: 100%; height: 100%; border-radius: 50%; border: 2px dashed ${cor}; opacity: 0.7; animation: spin 6s linear infinite;"></span><svg viewBox="0 0 24 24" fill="${cor}" stroke="white" stroke-width="1.5" style="width: 28px; height: 28px; z-index: 10;"><path stroke-linecap="round" stroke-linejoin="round" d="M15 10.5a3 3 0 11-6 0 3 3 0 016 0z"/><path stroke-linecap="round" stroke-linejoin="round" d="M19.5 10.5c0 7.142-7.5 11.25-7.5 11.25S4.5 17.642 4.5 10.5a7.5 7.5 0 1115 0z"/></svg><div style="position: absolute; top: -2px; right: -2px; width: 18px; height: 18px; background-color: rgba(15, 23, 42, 0.9); border-radius: 50%; border: 1px solid #475569; display: flex; align-items: center; justify-content: center; padding: 2px; z-index: 20; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.5);"><img src="https://assets.hgbrasil.com/weather/icons/conditions/${alert.iconSlug || 'cloudly_day'}.svg" style="width: 100%; height: 100%; object-fit: contain;" /></div></div>`,
      iconSize: [42, 42], iconAnchor: [21, 42]
    })
    const marker = L.marker([alert.lat, alert.lng], { icon }).on('click', (e) => {
      L.DomEvent.stopPropagation(e); selectedArea.value = alert
    })
    dynamicMarkersLayer.addLayer(marker)
  }
}

const renderMarkers = () => {
  if (!markersLayer) return
  markersLayer.clearLayers()
  const areas = activeFilter.value === 'all' ? riskAreas.value : riskAreas.value.filter(a => a.level === activeFilter.value)
  areas.forEach(area => {
    const icon = L.divIcon({
      className: 'custom-pin',
      html: `<div class="relative flex items-center justify-center w-8 h-8 drop-shadow-xl transition-all duration-300 hover:scale-110"><svg viewBox="0 0 24 24" fill="${alertColors[area.level]}" stroke="white" stroke-width="2" class="w-full h-full"><path stroke-linecap="round" stroke-linejoin="round" d="M15 10.5a3 3 0 11-6 0 3 3 0 016 0z"/><path stroke-linecap="round" stroke-linejoin="round" d="M19.5 10.5c0 7.142-7.5 11.25-7.5 11.25S4.5 17.642 4.5 10.5a7.5 7.5 0 1115 0z"/></svg>${['red', 'orange'].includes(area.level) ? '<span class="absolute top-[25%] w-2.5 h-2.5 bg-white rounded-full animate-ping opacity-75"></span>' : ''}</div>`,
      iconSize: [32, 32], iconAnchor: [16, 32],
    })
    const marker = L.marker([area.lat, area.lng], { icon }).on('click', (e) => {
      L.DomEvent.stopPropagation(e); fecharCard(); selectedArea.value = area
      if (mapInstance) mapInstance.flyTo([area.lat, area.lng], 15, { duration: 1.5 })
    })
    markersLayer.addLayer(marker)
  })
}

const inspecionarCoordenadaExpandida = async (lat, lng, nomeCentral = 'Área Inspecionada') => {
  try {
    if (dynamicMarkersLayer) dynamicMarkersLayer.clearLayers()
    else dynamicMarkersLayer = L.layerGroup().addTo(mapInstance)

    let tempApi = '--', chuvaApi = 0, condicaoDesc = 'Buscando condições...', iconSlug = 'cloudly_day', ventoApi = '0 km/h', turnoApi = 'dia'


    try {
      const baseUrl = import.meta.env.VITE_API_URL || 'https://sky-radar-api-production.up.railway.app';
      const res = await fetch(`${baseUrl}/api/weather/${lat}/${lng}`);
      if (res.ok) {
        const fullData = await res.json()
        const data = fullData.results || fullData
        if (data) {
          tempApi = data.temp || '--'
          condicaoDesc = data.description || 'Condições normais'
          iconSlug = data.condition_slug || 'cloudly_day'
          ventoApi = data.wind_speedy || '0 km/h'
          turnoApi = data.currently || 'dia'

          const desc = condicaoDesc.toLowerCase();  //eu tinha colocado aqui amigo
          chuvaApi = /tempestade|forte|toró/.test(desc) ? 28 : /chuva|chuvisco|fustada|pancada|chuvoso/.test(desc) ? 12 : /neblina|garoa|instável/.test(desc) ? 4 : 0;
        }
      }
    } catch (e) { console.warn("⚠️ API falhou.", e) }

    const malha = [
      { n: 'Ponto Central', dl: 0, dg: 0 }, { n: 'Vetor Norte', dl: 0.024, dg: 0 }, { n: 'Vetor Sul', dl: -0.024, dg: 0 },
      { n: 'Vetor Leste', dl: 0, dg: 0.030 }, { n: 'Vetor Oeste', dl: 0, dg: -0.030 }, { n: 'Vetor Nordeste', dl: 0.018, dg: 0.022 },
      { n: 'Vetor Noroeste', dl: 0.018, dg: -0.022 }, { n: 'Vetor Sudeste', dl: -0.018, dg: 0.022 }, { n: 'Vetor Sudoeste', dl: -0.018, dg: -0.022 },
      { n: 'Extremo Norte', dl: 0.048, dg: 0 }, { n: 'Extremo Sul', dl: -0.048, dg: 0 }, { n: 'Extremo Leste', dl: 0, dg: 0.060 },
      { n: 'Extremo Oeste', dl: 0, dg: -0.060 }, { n: 'Extremo Nordeste', dl: 0.036, dg: 0.044 }, { n: 'Extremo Noroeste', dl: 0.036, dg: -0.044 },
      { n: 'Extremo Sudeste', dl: -0.036, dg: 0.044 }, { n: 'Extremo Sudoeste', dl: -0.036, dg: -0.044 }
    ]

    malha.forEach((cfg, i) => {
      const ptLat = lat + cfg.dl, ptLng = lng + cfg.dg
      const chuvaPt = i === 0 ? chuvaApi : Math.max(0, chuvaApi - Math.floor(Math.random() * 5))
      const nivel = chuvaPt > 15 ? 'red' : chuvaPt > 5 ? 'orange' : chuvaPt > 0 ? 'yellow' : 'green'

      const obj = {
        id: `dyn-${i}-${Date.now()}-${Math.random().toString(36).substring(2, 9)}`,
        name: i === 0 ? nomeCentral : cfg.n, lat: ptLat, lng: ptLng, level: nivel,
        desc: `${i === 0 ? nomeCentral : cfg.n} (SkyRadar). ${condicaoDesc}`,
        temp: tempApi, rain: chuvaPt, wind: ventoApi, turn: turnoApi, iconSlug
      }

      if (i === 0) selectedArea.value = obj
      localRadarData.value.unshift(obj)

      const cor = alertColors[nivel] || '#10B981'
      const icon = L.divIcon({
        className: 'marker-custom-radar',
        html: `<div style="position: relative; width: 42px; height: 42px; display: flex; align-items: center; justify-content: center;"><span style="position: absolute; width: 100%; height: 100%; border-radius: 50%; border: 2px dashed ${cor}; opacity: 0.7; animation: spin 6s linear infinite;"></span><svg viewBox="0 0 24 24" fill="${cor}" stroke="white" stroke-width="1.5" style="width: 28px; height: 28px; z-index: 10;"><path stroke-linecap="round" stroke-linejoin="round" d="M15 10.5a3 3 0 11-6 0 3 3 0 016 0z"/><path stroke-linecap="round" stroke-linejoin="round" d="M19.5 10.5c0 7.142-7.5 11.25-7.5 11.25S4.5 17.642 4.5 10.5a7.5 7.5 0 1115 0z"/></svg><div style="position: absolute; top: -2px; right: -2px; width: 18px; height: 18px; background-color: rgba(15, 23, 42, 0.9); border-radius: 50%; border: 1px solid #475569; display: flex; align-items: center; justify-content: center; padding: 2px; z-index: 20; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.5);"><img src="https://assets.hgbrasil.com/weather/icons/conditions/${iconSlug}.svg" style="width: 100%; height: 100%; object-fit: contain;" /></div></div>`,
        iconSize: [42, 42], iconAnchor: [21, 42]
      })

      const marker = L.marker([ptLat, ptLng], { icon }).on('click', (e) => {
        L.DomEvent.stopPropagation(e); selectedArea.value = obj; mapInstance?.flyTo([ptLat, ptLng], 15, { duration: 1.2 })
      })
      dynamicMarkersLayer.addLayer(marker)
    })
    mapInstance?.flyTo([lat, lng], 12, { duration: 1.5 })
  } catch (e) { console.error("Erro:", e) }
}

const resetView = () => { if (mapInstance) { mapInstance.flyTo([-8.05, -34.9], 12, { duration: 1.5 }); fecharCard() } }


const buscarLocal = async () => {
  if (!searchQuery.value.trim()) return;

  try {

    const baseUrl = import.meta.env.VITE_API_URL || 'https://sky-radar-api-production.up.railway.app';

    const response = await fetch(`${baseUrl}/api/search/${encodeURIComponent(searchQuery.value)}`);

    if (!response.ok) throw new Error('Falha na requisição');

    const data = await response.json();

    searchResults.value = data?.length ? data : [];
    if (searchResults.value.length === 0) alert('Nenhum local encontrado.');

  } catch (error) {
    console.error("Erro na busca:", error);
    alert('Erro ao conectar com o servidor de busca.');
  }
};

const selecionarLocal = async (local) => {
  searchResults.value = []; searchQuery.value = ''; fecharCard()
  await inspecionarCoordenadaExpandida(parseFloat(local.lat), parseFloat(local.lon), local.display_name.split(',').slice(0, 3).join(', '))
}

const locateUser = () => {
  if (!navigator.geolocation) return
  navigator.geolocation.getCurrentPosition(
    async (pos) => {
      const { latitude: lat, longitude: lng } = pos.coords
      mapInstance?.flyTo([lat, lng], 15, { duration: 1.5 })
      if (userMarker && mapInstance) mapInstance.removeLayer(userMarker)
      const icon = L.divIcon({
        className: 'custom-pin',
        html: `<div class="relative flex items-center justify-center w-6 h-6"><span class="absolute inline-flex h-full w-full rounded-full bg-blue-400 opacity-75 animate-ping"></span><span class="relative inline-flex rounded-full h-4 w-4 bg-blue-500 border-2 border-white shadow-lg"></span></div>`,
        iconSize: [24, 24], iconAnchor: [12, 12]
      })
      if (mapInstance) userMarker = L.marker([lat, lng], { icon }).addTo(mapInstance)
      await inspecionarCoordenadaExpandida(lat, lng, "Sua Localização Atual")
    },
    (err) => console.error(err), { enableHighAccuracy: true, timeout: 10000 }
  )
}

onMounted(async () => {
  iniciarBussolaDinamica()
  if (!mapElement.value || mapInstance) return
  mapInstance = L.map(mapElement.value, { zoomControl: false, trackResize: true }).setView([-8.05, -34.9], 12)
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { maxZoom: 19 }).addTo(mapInstance)
  L.control.zoom({ position: 'bottomright' }).addTo(mapInstance)
  markersLayer = L.layerGroup().addTo(mapInstance)
  dynamicMarkersLayer = L.layerGroup().addTo(mapInstance)

  mapInstance.on('click', async (e) => {
    fecharCard()
    const { lat, lng } = e.latlng.wrap()
    let nome = `Coordenada: ${lat.toFixed(2)}, ${lng.toFixed(2)}`
    try {
      const res = await fetch(`https://api.bigdatacloud.net/data/reverse-geocode-client?latitude=${lat}&longitude=${lng}&localityLanguage=pt`)
      if (res.ok) {
        const data = await res.json()
        nome = data.locality || data.city || data.principalSubdivision || nome
      }
    } catch (err) { console.warn("⚠️ Geocodificação falhou.") }
    await inspecionarCoordenadaExpandida(lat, lng, nome)
  })

  await fetchAlerts()
  locateUser()
})

onUnmounted(() => { if (mapInstance) { mapInstance.remove(); mapInstance = null } })
</script>

<template>
  <main class="grid grid-cols-1 md:grid-cols-4 h-screen w-full overflow-hidden bg-slate-900 text-slate-100 font-sans">

    <aside
      class="col-span-1 flex flex-col h-full bg-slate-950 border-r border-slate-800 p-5 overflow-hidden z-10 shadow-2xl">

      <header class="mb-5 shrink-0 flex justify-between items-center">
        <div>
          <h1 class="text-2xl font-black tracking-wider text-blue-500">🛰️ SKY<span class="text-white">RADAR</span></h1>
          <p class="text-[10px] text-slate-500 uppercase font-bold tracking-widest mt-0.5">Centro de Comando</p>
        </div>
        <span class="animate-pulse flex h-3 w-3 rounded-full bg-green-500" title="Sistema Operacional"></span>
      </header>

      <form @submit.prevent="buscarLocal" class="relative w-full mb-6 shrink-0">
        <input v-model="searchQuery" type="search" placeholder="Pesquisar cidade..."
          class="w-full bg-slate-900 text-white px-4 py-3.5 pr-10 rounded-xl border border-slate-700 shadow-inner focus:outline-none focus:border-blue-500 transition-all text-sm placeholder-slate-500">
        <button type="submit" class="absolute right-3 top-1/2 -translate-y-1/2 text-slate-400 hover:text-blue-400">
          <svg fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="w-5 h-5">
            <path stroke-linecap="round" stroke-linejoin="round"
              d="M21 21l-5.197-5.197m0 0A7.5 7.5 0 105.196 5.196a7.5 7.5 0 0010.607 10.607z" />
          </svg>
        </button>
      </form>

      <div v-if="selectedArea"
        class="mb-6 p-4 rounded-2xl bg-slate-800/80 border border-slate-700 shadow-lg shrink-0 transition-all">
        <div class="flex justify-between items-start mb-3">
          <h3 class="text-lg font-black truncate pr-2">{{ selectedArea.name }}</h3>
          <button @click="fecharCard"
            class="text-slate-400 hover:text-white bg-slate-700/50 rounded-full w-6 h-6 flex items-center justify-center text-xs transition-colors">✕</button>
        </div>

        <div
          class="mb-4 inline-flex items-center gap-2 px-2 py-1 rounded-md text-[10px] font-bold uppercase tracking-widest border"
          :style="{ borderColor: alertColors[selectedArea.level], backgroundColor: alertColors[selectedArea.level] + '20', color: alertColors[selectedArea.level] }">
          <span class="w-2 h-2 rounded-full animate-pulse"
            :style="{ backgroundColor: alertColors[selectedArea.level] }"></span>
          {{ nivelNomes[selectedArea.level] }}
        </div>

        <div class="grid grid-cols-3 gap-2 text-center bg-slate-900/50 p-2 rounded-xl border border-slate-700/50">
          <div class="flex flex-col items-center justify-center">
            <span class="text-xl">🌡️</span>
            <span class="font-bold mt-1 text-sm">{{ selectedArea.temp ?? '--' }}</span>
            <span class="text-[9px] text-slate-500 uppercase font-bold tracking-widest mt-0.5">Temp</span>
          </div>
          <div class="flex flex-col items-center justify-center border-l border-r border-slate-700/50">
            <img v-if="selectedArea.iconSlug"
              :src="'https://assets.hgbrasil.com/weather/icons/conditions/' + selectedArea.iconSlug + '.svg'"
              class="w-6 h-6 object-contain" />
            <span v-else class="text-xl">☁️</span>
            <span class="font-bold mt-1 text-sm text-blue-400">{{ selectedArea.rain ?? '0' }}<span
                class="text-[10px] text-slate-400">mm</span></span>
            <span class="text-[9px] text-slate-500 uppercase font-bold tracking-widest mt-0.5">Chuva</span>
          </div>
          <div class="flex flex-col items-center justify-center">
            <span class="text-xl">💨</span>
            <span class="font-bold mt-1 text-sm text-teal-400">{{ selectedArea.wind ? selectedArea.wind.replace(' km/h',
              '') : '0' }}</span>
            <span class="text-[9px] text-slate-500 uppercase font-bold tracking-widest mt-0.5">Vento</span>
          </div>
        </div>

        <p class="text-xs text-slate-300 mt-3 p-3 bg-slate-900/50 rounded-lg border-l-2 shadow-inner"
          :style="{ borderColor: alertColors[selectedArea.level] }">
          {{ selectedArea.desc }}
        </p>
      </div>

      <div v-else
        class="mb-6 p-5 rounded-2xl bg-slate-800/30 border border-slate-700/50 border-dashed text-center shrink-0 flex flex-col items-center justify-center">
        <span class="text-3xl mb-3 block opacity-40">🗺️</span>
        <p class="text-xs text-slate-400 leading-relaxed">Selecione um ponto no mapa do Recife para inspecionar os dados
          de risco e precipitação.</p>
      </div>

      <section class="flex-1 overflow-y-auto custom-scrollbar pr-1">
        <div
          class="flex items-center justify-between border-b border-slate-800 pb-2 mb-3 sticky top-0 bg-slate-950 z-10">
          <span class="text-xs font-bold text-slate-400 uppercase tracking-widest">Sensores & Alertas</span>

          <div class="flex gap-1.5 bg-slate-900 p-1 rounded-full border border-slate-800">
            <button @click="toggleFilter('all')"
              :class="activeFilter === 'all' ? 'opacity-100 ring-2 ring-white/50 scale-110' : 'opacity-30 hover:opacity-60'"
              class="w-3 h-3 rounded-full bg-slate-400 transition-all" title="Todos"></button>
            <button @click="toggleFilter('yellow')"
              :class="activeFilter === 'yellow' ? 'opacity-100 ring-2 ring-white/50 scale-110' : 'opacity-30 hover:opacity-60'"
              class="w-3 h-3 rounded-full bg-yellow-400 transition-all" title="Atenção"></button>
            <button @click="toggleFilter('orange')"
              :class="activeFilter === 'orange' ? 'opacity-100 ring-2 ring-white/50 scale-110' : 'opacity-30 hover:opacity-60'"
              class="w-3 h-3 rounded-full bg-orange-500 transition-all" title="Alerta"></button>
            <button @click="toggleFilter('red')"
              :class="activeFilter === 'red' ? 'opacity-100 ring-2 ring-white/50 scale-110' : 'opacity-30 hover:opacity-60'"
              class="w-3 h-3 rounded-full bg-red-500 transition-all" title="Risco"></button>
          </div>
        </div>

        <div v-if="criticalAlerts.length === 0" class="text-center py-6 text-slate-500 text-xs animate-pulse">A aguardar
          varredura dos pluviómetros...</div>

        <div class="space-y-2 pb-4">
          <article v-for="alert in criticalAlerts" :key="alert.id" @click="focarNoAlerta(alert)"
            class="p-3 rounded-xl border border-slate-700/60 bg-slate-800/40 hover:bg-slate-700/80 transition-all cursor-pointer group flex flex-col gap-1.5 shadow-sm hover:shadow-md">
            <div class="flex items-center gap-2">
              <div class="w-2.5 h-2.5 rounded-full shadow-lg shrink-0"
                :style="{ backgroundColor: alertColors[alert.level], boxShadow: `0 0 8px ${alertColors[alert.level]}` }">
              </div>
              <p class="text-[13px] font-black truncate text-slate-200 group-hover:text-white">{{ alert.name }}</p>
            </div>
            <div class="flex justify-between items-center pl-4.5">
              <p class="text-[10px] text-slate-400 uppercase tracking-widest font-bold">Chuva: <span
                  class="text-blue-400">{{ alert.rain }}mm</span></p>
              <span
                class="text-[9px] font-bold text-slate-500 group-hover:text-blue-400 transition-colors uppercase tracking-wider">Ver
                no Mapa ➔</span>
            </div>
          </article>
        </div>
      </section>

    </aside>

    <section class="col-span-1 md:col-span-3 relative h-full w-full bg-slate-800">

      <div ref="mapElement" class="absolute inset-0 z-0"></div>

      <nav class="absolute z-[1500] bottom-6 right-6 flex flex-col items-center gap-3">

        <button @click="toggleRadarLayer"
          :class="isRadarActive ? 'bg-purple-600 border-purple-400' : 'bg-slate-900/90 border-slate-700'"
          class="backdrop-blur-md text-white w-12 h-12 rounded-full border shadow-xl hover:bg-purple-700 transition-all flex items-center justify-center relative group"
          title="Radar de Precipitação">

          <span class="text-xl relative z-10">🌧️</span>
          <span v-if="isRadarActive"
            class="absolute inset-0 rounded-full border-2 border-purple-400 animate-ping opacity-75"></span>

          <span
            class="absolute right-14 bg-slate-900 text-[10px] font-bold uppercase px-2 py-1 rounded border border-slate-700 opacity-0 group-hover:opacity-100 transition-opacity whitespace-nowrap pointer-events-none">
            {{ isRadarActive ? 'Desativar Radar' : 'Ativar Radar de Chuva' }}
          </span>
        </button>

        <button @click="locateUser"
          class="bg-blue-600/90 backdrop-blur-md text-white w-12 h-12 rounded-full border border-blue-500 shadow-xl hover:bg-blue-700 transition-all flex items-center justify-center"
          title="A Minha Localização">
          <span class="text-xl">🎯</span>
        </button>
        <div title="Norte Verdadeiro"
          class="bg-slate-900/90 backdrop-blur-md border border-slate-700 w-20 h-20 rounded-full flex items-center justify-center shadow-2xl transition-all relative overflow-hidden ring-2 ring-slate-900/50 shrink-0">

          <svg viewBox="0 0 120 120" class="absolute inset-0 w-full h-full p-1.5">
            <defs>
              <g id="sub">
                <polygon points="60,25 62.5,60 60,60" fill="#475569" />
                <polygon points="60,25 57.5,60 60,60" fill="#64748b" />
              </g>
              <g id="col">
                <polygon points="60,18 64,60 60,60" fill="#a16207" />
                <polygon points="60,18 56,60 60,60" fill="#eab308" />
              </g>
              <g id="car">
                <polygon points="60,10 66,60 60,60" fill="#ca8a04" />
                <polygon points="60,10 54,60 60,60" fill="#fef08a" />
              </g>
            </defs>

            <use href="#sub" transform="rotate(22.5 60 60)" />
            <use href="#sub" transform="rotate(67.5 60 60)" />
            <use href="#sub" transform="rotate(112.5 60 60)" />
            <use href="#sub" transform="rotate(157.5 60 60)" />
            <use href="#sub" transform="rotate(202.5 60 60)" />
            <use href="#sub" transform="rotate(247.5 60 60)" />
            <use href="#sub" transform="rotate(292.5 60 60)" />
            <use href="#sub" transform="rotate(337.5 60 60)" />

            <use href="#col" transform="rotate(45 60 60)" />
            <use href="#col" transform="rotate(135 60 60)" />
            <use href="#col" transform="rotate(225 60 60)" />
            <use href="#col" transform="rotate(315 60 60)" />

            <use href="#car" transform="rotate(90 60 60)" />
            <use href="#car" transform="rotate(180 60 60)" />
            <use href="#car" transform="rotate(270 60 60)" />

            <polygon points="60,10 66,60 60,60" fill="#991b1b" />
            <polygon points="60,10 54,60 60,60" fill="#ef4444" />

            <circle cx="60" cy="60" r="14" fill="#0f172a" stroke="#ca8a04" stroke-width="1.5" />

            <text x="60" y="8" text-anchor="middle" font-size="8" fill="#ef4444" font-weight="900"
              style="text-shadow: 0 1px 2px black;">N</text>
            <text x="60" y="117" text-anchor="middle" font-size="7" fill="#cbd5e1" font-weight="bold">S</text>
            <text x="5" y="62.5" text-anchor="middle" font-size="7" fill="#cbd5e1" font-weight="bold">O</text>
            <text x="115" y="62.5" text-anchor="middle" font-size="7" fill="#cbd5e1" font-weight="bold">L</text>

            <text x="97" y="27" text-anchor="middle" font-size="5.5" fill="#94a3b8" font-weight="bold">NE</text>
            <text x="97" y="99" text-anchor="middle" font-size="5.5" fill="#94a3b8" font-weight="bold">SE</text>
            <text x="23" y="99" text-anchor="middle" font-size="5.5" fill="#94a3b8" font-weight="bold">SO</text>
            <text x="23" y="27" text-anchor="middle" font-size="5.5" fill="#94a3b8" font-weight="bold">NO</text>
          </svg>

          <svg viewBox="0 0 120 120"
            class="absolute inset-0 w-full h-full z-10 transition-transform duration-200 ease-out drop-shadow-2xl"
            :style="{ transform: `rotate(${anguloBussola}deg)` }">
            <polygon points="60,22 63,60 57,60" fill="#ef4444" />
            <polygon points="60,98 63,60 57,60" fill="#f8fafc" />
            <circle cx="60" cy="60" r="3" fill="#0f172a" stroke="#ef4444" stroke-width="1.5" />
          </svg>

        </div>


      </nav>

    </section>

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