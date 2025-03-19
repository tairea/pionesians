<template>
  <div ref="globeContainer" class="globe-container"></div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import Globe from 'globe.gl'
import { FrontSide } from 'three'
import * as THREE from 'three'

const globeContainer = ref(null)

// Country to flag emoji mapping
const countryFlags = {
  'American Samoa': '🇦🇸',
  'Cook Islands': '🇨🇰',
  'Fiji': '🇫🇯',
  'French Polynesia': '🇵🇫',
  'Guam': '🇬🇺',
  'Kiribati': '🇰🇮',
  'Marshall Islands': '🇲🇭',
  'Federated States of Micronesia': '🇫🇲',
  'Nauru': '🇳🇷',
  'New Caledonia': '🇳🇨',
  'Niue': '🇳🇺',
  'Northern Mariana Islands': '🇲🇵',
  'Palau': '🇵🇼',
  'Papua New Guinea': '🇵🇬',
  'Pitcairn Islands': '🇵🇳',
  'Samoa': '🇼🇸',
  'Solomon Islands': '🇸🇧',
  'Tokelau': '🇹🇰',
  'Tonga': '🇹🇴',
  'Tuvalu': '🇹🇻',
  'Vanuatu': '🇻🇺',
  'Wallis and Futuna': '🇼🇫',
  'USA': '🇺🇸',
  'New Zealand': '🇳🇿'
}

// Country to color mapping (using pastel colors)
const countryColors = {
  'American Samoa': '#FFB3BA', // Pastel pink
  'Cook Islands': '#BAFFC9', // Pastel green
  'Fiji': '#BAE1FF', // Pastel blue
  'French Polynesia': '#FFFFBA', // Pastel yellow
  'Guam': '#FFE4BA', // Pastel orange
  'Kiribati': '#E8BAFF', // Pastel purple
  'Marshall Islands': '#BAE1FF', // Pastel blue
  'Federated States of Micronesia': '#BAFFD9', // Pastel mint
  'Nauru': '#FFD9BA', // Pastel peach
  'New Caledonia': '#D4BAFF', // Pastel lavender
  'Niue': '#FFE4BA', // Pastel orange
  'Northern Mariana Islands': '#FFB3BA', // Pastel pink
  'Palau': '#BAE1FF', // Pastel blue
  'Papua New Guinea': '#E8BAFF', // Pastel purple
  'Pitcairn Islands': '#D4BAFF', // Pastel lavender
  'Samoa': '#FFB3BA', // Pastel pink
  'Solomon Islands': '#BAFFC9', // Pastel green
  'Tokelau': '#BAE1FF', // Pastel blue
  'Tonga': '#FFD9BA', // Pastel peach
  'Tuvalu': '#BAE1FF', // Pastel blue
  'Vanuatu': '#BAFFC9', // Pastel green
  'Wallis and Futuna': '#D4BAFF', // Pastel lavender
  'USA': '#FFB3BA', // Pastel pink
  'New Zealand': '#BAE1FF' // Pastel blue
}

// Arc height configuration
const ARC_HEIGHT_FACTOR = 0.4 // Adjust this to increase/decrease all arc heights

// Calculate distance between two points on a sphere (Haversine formula)
const calculateDistance = (lat1, lon1, lat2, lon2) => {
  const R = 6371 // Earth's radius in km
  const dLat = (lat2 - lat1) * Math.PI / 180
  const dLon = (lon2 - lon1) * Math.PI / 180
  const a = 
    Math.sin(dLat/2) * Math.sin(dLat/2) +
    Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) * 
    Math.sin(dLon/2) * Math.sin(dLon/2)
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a))
  return R * c
}

// Calculate relative arc height based on distance with exponential scaling
const getArcHeight = (startLat, startLng, endLat, endLng) => {
  const distance = calculateDistance(startLat, startLng, endLat, endLng)
  // Use exponential scaling for better short-distance handling
  // Normalize to 0-1 range with stronger effect on shorter distances
  const normalizedHeight = Math.pow(distance / 5000, 0.5) * ARC_HEIGHT_FACTOR
  return Math.max(0.05, Math.min(0.5, normalizedHeight)) // Clamp between 0.05 and 0.5
}

// Add arc animation data
const arcData = [{
  endLat: -20.1361, // Mauke, Cook Islands
  endLng: -157.3458,
  startLat: -21.9119, // Mangaia, Cook Islands
  startLng: -157.9203,
  color: [`rgba(186, 255, 201, 1)`, `rgba(186, 255, 201, 1)`], // Using Cook Islands color
  height: getArcHeight(-21.9119, -157.9203, -20.1361, -157.3458)
}, {
  startLat: -40.3523, // Palmerston North, New Zealand
  startLng: 175.6087,
  endLat: -13.8506, // Upolu, Samoa
  endLng: -171.7513,
  color: [`rgba(255, 179, 186, 1)`, `rgba(255, 179, 186, 1)`], // Using Samoa's color
  height: getArcHeight(-40.3523, 175.6087, -13.8506, -171.7513)
}, {
  startLat: -20.1361, // Mauke, Cook Islands
  startLng: -157.3458,
  endLat: -13.8506, // Upolu, Samoa
  endLng: -171.7513,
  color: [`rgba(186, 255, 201, 1)`, `rgba(186, 255, 201, 1)`], // Using Cook Islands color
  height: getArcHeight(-20.1361, -157.3458, -13.8506, -171.7513)
}, {
  startLat: 5.7923, // Jericho, Colombia
  startLng: -75.5417,
  endLat: -20.1361, // Mauke, Cook Islands
  endLng: -157.3458,
  color: [`rgba(255, 228, 186, 1)`, `rgba(255, 228, 186, 1)`], // Using pastel orange
  height: getArcHeight(5.7923, -75.5417, -20.1361, -157.3458)
}]

onMounted(async () => {
  try {
    // Fetch base countries data and islands data
    const [countries, islandsData] = await Promise.all([
      fetch('https://raw.githubusercontent.com/vasturiano/globe.gl/master/example/datasets/ne_110m_admin_0_countries.geojson').then(res => res.json()),
      fetch('/pionesians/data/islands.geojson').then(res => res.json())
    ])
    
    const globe = Globe()
      .backgroundColor('#ffffff')
      .width(window.innerWidth)
      .height(window.innerHeight)
      .globeImageUrl(null)
      .showAtmosphere(true)
      .atmosphereColor('#e0e0e0')
      .atmosphereAltitude(0.15)
      .globeMaterial({
        color: '#ffffff',
        transparent: false,
        opacity: 1,
        side: FrontSide,
        wireframe: false,
        flatShading: true
      })
      // Add white background layer
      .customLayerData([{ type: 'Feature', geometry: { type: 'Sphere' } }])
      .customThreeObject(() => {
        const geometry = new THREE.SphereGeometry(100, 75, 75)
        const material = new THREE.MeshBasicMaterial({ 
          color: '#ffffff',
          side: FrontSide
        })
        return new THREE.Mesh(geometry, material)
      })
      // Combine countries and islands data
      .hexPolygonsData([
        ...countries.features.map(f => ({ ...f, isPacificIsland: false })),
        ...islandsData.features.map(f => ({
          type: 'Feature',
          geometry: {
            type: 'Polygon',
            coordinates: [[
              [f.geometry.coordinates[0] - 0.5, f.geometry.coordinates[1] - 0.5],
              [f.geometry.coordinates[0] + 0.5, f.geometry.coordinates[1] - 0.5],
              [f.geometry.coordinates[0] + 1, f.geometry.coordinates[1]],
              [f.geometry.coordinates[0] + 0.5, f.geometry.coordinates[1] + 0.5],
              [f.geometry.coordinates[0] - 0.5, f.geometry.coordinates[1] + 0.5],
              [f.geometry.coordinates[0] - 1, f.geometry.coordinates[1]],
              [f.geometry.coordinates[0] - 0.5, f.geometry.coordinates[1] - 0.5]
            ]]
          },
          properties: f.properties,
          isPacificIsland: true
        }))
      ])
      .hexPolygonResolution(3)
      .hexPolygonMargin(0.3)
      .hexPolygonUseDots(false)
      .hexPolygonColor(d => d.isPacificIsland ? countryColors[d.properties.parent] || '#e0e0e0' : 'rgba(224, 224, 224, 0.3)') // Color islands by country
      .hexPolygonAltitude(d => d.isPacificIsland ? 0.002 : 0.001) // Different heights
      .hexPolygonLabel(d => {
        if (!d.isPacificIsland) return '';
        const countryName = Object.keys(countryFlags).find(key => 
          d.properties.parent.includes(key)
        );
        return `${d.properties.name}, ${d.properties.parent} ${countryFlags[countryName] || ''}`;
      })
      // Add arc configuration
      .arcsData(arcData)
      .arcColor('color')
      .arcAltitude(d => d.height)
      .arcStroke(0.3)
      (globeContainer.value)

    // Center the globe on Polynesia
    globe.pointOfView({
      lat: -17.6797,
      lng: 175,
      altitude: 2.5
    }, 0)

    // Handle window resize
    window.addEventListener('resize', () => {
      globe
        .width(window.innerWidth)
        .height(window.innerHeight)
    })
  } catch (error) {
    console.error('Error initializing globe:', error)
  }
})
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@500&display=swap');

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.globe-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: radial-gradient(circle at 50% 50%, #ffffff, #f8f8f8);
}
</style> 