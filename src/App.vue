<script setup>
import { onMounted, onUnmounted } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

// Variável global para guardar a instância do mapa
let mapInstance = null;

onMounted(() => {
  // 1. Limpeza de Cache do Vite (HMR)
  // Se a div já tiver um mapa, nós "matamos" ele antes de recriar
  const mapContainer = document.getElementById('map')
  if (mapContainer != null && mapContainer._leaflet_id !== null) {
    mapContainer._leaflet_id = null
  }

  // 2. Inicializa o mapa com segurança
  mapInstance = L.map('map').setView([-8.05, -34.9], 12)

  // 3. Carrega o tema do OpenStreetMap
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 19,
    attribution: '© OpenStreetMap - SkyRadar'
  }).addTo(mapInstance)

  // 4. MOCK DE DADOS: O que o nosso DTO do Laravel vai retornar
  const riskAreas = [
    { id: 1, name: 'Marco Zero', lat: -8.0631, lng: -34.8711, level: 'green', desc: 'Vias liberadas. Sem acúmulo de água.' },
    { id: 2, name: 'Viaduto da Caxangá', lat: -8.0434, lng: -34.9332, level: 'yellow', desc: 'Atenção: Fluxo lento, risco moderado de acúmulo na base do viaduto e vias de acesso.' },
    { id: 3, name: 'Av. Mascarenhas de Morais', lat: -8.1068, lng: -34.9126, level: 'orange', desc: 'Alerta: Ponto de alagamento confirmando próximo ao aeroporto. Evite a rota.' },
    { id: 4, name: 'Dois Irmãos / Macaxeira', lat: -8.0142, lng: -34.9458, level: 'red', desc: 'Risco Extremo: Via interditada. Risco de deslizamento.' }
  ]

  // 5. Dicionário de cores táticas
  const alertColors = {
    green: '#22c55e',
    yellow: '#facc15',
    orange: '#f97316',
    red: '#ef4444'
  }

  // 6. Renderizar os pontos no mapa com o visual de "Radar"
  riskAreas.forEach(area => {
    L.circleMarker([area.lat, area.lng], {
      radius: 10,
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

// 7. Boa prática do Vue: Destruir o mapa quando sair da tela
onUnmounted(() => {
  if (mapInstance) {
    mapInstance.remove()
  }
})
</script>