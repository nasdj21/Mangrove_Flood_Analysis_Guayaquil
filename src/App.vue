<template>
  <div class="map-container">
    <div id="map" class="map"></div>
    <div class="control-panel">
      <h3 class="title">Flooding Exposure - Guayaquil</h3>
      
      <div class="layer-buttons">
        <button 
          v-for="layer in ['Exposure', 'Priority']"
          :key="layer"
          :class="{ active: currentLayer === layer}"
          @click="changeLayer(layer)"
        >
        {{ layer }}
        </button>
      </div>

      <div v-if="selectedParroquia" class="info-panel">
        Parroquia Seleccionada: <strong>{{ selectedParroquia }}</strong>
        <div class="stats-box">
           <span v-if="isCalculating">Analizando píxeles...</span>
           <span v-else-if="percentageVulnerable !== null">
              Área Vulnerable ({{ currentLayer }}): 
              <strong style="color: #ff4a4a; font-size: 1.2rem;">{{ percentageVulnerable }}%</strong>
           </span>
        </div>
      </div>

    </div>
  </div>
</template>

<script setup>
import { onMounted, ref } from 'vue';
import maplibregl from 'maplibre-gl';
import 'maplibre-gl/dist/maplibre-gl.css';
import { fromArrayBuffer } from 'geotiff';

const percentageVulnerable = ref(null);
const isCalculating = ref(false);
const rasterDataCache = ref({}); 
let map = null;
const currentLayer = ref('Exposure');
const selectedParroquia = ref(null);

const calculateBBox = (geometry) => {
  let minLng = Infinity, minLat = Infinity, maxLng = -Infinity, maxLat = -Infinity;
  const coords = geometry.type === 'MultiPolygon' ? geometry.coordinates.flat(2) :
                 geometry.type === 'Polygon' ? geometry.coordinates.flat(1) : [];

  coords.forEach(coord => {
    if (coord[0] < minLng) minLng = coord[0];
    if (coord[1] < minLat) minLat = coord[1];
    if (coord[0] > maxLng) maxLng = coord[0];
    if (coord[1] > maxLat) maxLat = coord[1];
  });

  return [[minLng, minLat], [maxLng, maxLat]];
};

const createRasterImage = async (layerName, url, hexColor, targetBand = 0, targetValue = 5) => {
  try {
    const response = await fetch(url);
    const arrayBuffer = await response.arrayBuffer();

    const tiff = await fromArrayBuffer(arrayBuffer);
    const image = await tiff.getImage();
    const bbox = image.getBoundingBox();
    const width = image.getWidth();
    const height = image.getHeight();

    const rasters = await image.readRasters();
    const data = rasters[targetBand]; 

    rasterDataCache.value[layerName] = {
      data: data,
      width: width,
      height: height,
      bbox: bbox,
      targetValue: targetValue
    };

    const canvas = document.createElement('canvas')
    canvas.width = width
    canvas.height = height
    const context = canvas.getContext('2d') 
    const imageData = context.createImageData(width, height)

    const rojo = parseInt(hexColor.slice(1, 3), 16);
    const verde = parseInt(hexColor.slice(3, 5), 16);
    const azul = parseInt(hexColor.slice(5, 7), 16);

    for (let i = 0; i < data.length; i++) {
      if (data[i] === targetValue) { 
        imageData.data[i * 4] = rojo;
        imageData.data[i * 4 + 1] = verde;
        imageData.data[i * 4 + 2] = azul;
        imageData.data[i * 4 + 3] = 255; 
      } else {
        imageData.data[i * 4 + 3] = 0; 
      }
    }

    context.putImageData(imageData, 0, 0);
    return {
      url: canvas.toDataURL(),
      coordinates: [
        [bbox[0], bbox[3]], 
        [bbox[2], bbox[3]], 
        [bbox[2], bbox[1]], 
        [bbox[0], bbox[1]]  
      ]
    };
  } catch (error) {
    console.error(`Error procesando ${url}:`, error);
    return null;
  }
};

const pointInPolygon = (point, vs) => {
  let x = point[0], y = point[1];
  let inside = false;
  for (let i = 0, j = vs.length - 1; i < vs.length; j = i++) {
    let xi = vs[i][0], yi = vs[i][1];
    let xj = vs[j][0], yj = vs[j][1];
    let intersect = ((yi > y) != (yj > y)) && (x < (xj - xi) * (y - yi) / (yj - yi) + xi);
    if (intersect) inside = !inside;
  }
  return inside;
};

onMounted(() => {
  map = new maplibregl.Map({
    container: 'map',
    style: 'https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json',
    center: [-79.8862, -2.19],
    zoom: 11,
    pitch: 0
  });

  map.on('load', async () => {
    const baseUrl = import.meta.env.BASE_URL;
    
    const [imgExposure, imgPriority] = await Promise.all([
      createRasterImage('Exposure', `${baseUrl}gye_mapbiomas_protection_exposure.tif`, '#e00d26', 3, 1), 
      createRasterImage('Priority', `${baseUrl}gye_mapbiomas_protection_exposure.tif`, '#edea1f', 4, 1)
    ]);

    const rasterImages = { 'Exposure': imgExposure, 'Priority': imgPriority };

    for (const [layerName, imgData] of Object.entries(rasterImages)) {
      if (!imgData) continue;

      map.addSource(`source-${layerName}`, {
        type: 'image',
        url: imgData.url,
        coordinates: imgData.coordinates
      });

      map.addLayer({
        id: `layer-${layerName}`,
        type: 'raster',
        source: `source-${layerName}`,
        layout: {
          'visibility': layerName === 'Exposure' ? 'visible' : 'none'
        },
        paint: {
          'raster-opacity': 0.8,
          'raster-resampling': 'nearest' 
        }
      });
    }

    const geojsonUrl = `${baseUrl}${baseUrl.endsWith('/') ? '' : '/'}parroquias.geojson`;
    
    try {
      const response = await fetch(geojsonUrl);
      if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
      const geojsonData = await response.json();

      map.addSource('source-parroquias', {
        type: 'geojson',
        data: geojsonData 
      });

      map.addLayer({
        id: 'layer-parroquias-fill',
        type: 'fill',
        source: 'source-parroquias',
        paint: {
          'fill-color': '#ffffff',
          'fill-opacity': 0.01
        }
      });

      map.addLayer({
        id: 'layer-parroquias-line',
        type: 'line',
        source: 'source-parroquias',
        paint: {
          'line-color': '#ffffff',
          'line-width': 1,
          'line-opacity': 0.5
        }
      });

    } catch (error) {
      console.error(error);
    }

    map.on('mouseenter', 'layer-parroquias-fill', () => {
      map.getCanvas().style.cursor = 'pointer';
    });
    map.on('mouseleave', 'layer-parroquias-fill', () => {
      map.getCanvas().style.cursor = '';
    });

    map.on('click', 'layer-parroquias-fill', (e) => {
      const feature = e.features[0];
      
      selectedParroquia.value = feature.properties.PARROQUIA || feature.properties.DPA_PARROQ || 'Parroquia sin nombre';
      const bbox = calculateBBox(feature.geometry);
      map.fitBounds(bbox, { padding: 50, duration: 1200 });
     
      const currentRaster = rasterDataCache.value[currentLayer.value];
      
      if (currentRaster) {
        isCalculating.value = true;
        percentageVulnerable.value = null;

        const polygonCoords = feature.geometry.type === 'Polygon' ? feature.geometry.coordinates[0] : null;

        if (polygonCoords) {
           setTimeout(() => {
              let totalPixelsInPolygon = 0;
              let vulnerablePixels = 0;

              const { data, width, height, bbox: rBbox, targetValue } = currentRaster;
              const rMinLng = rBbox[0];
              const rMinLat = rBbox[1];
              const rMaxLng = rBbox[2];
              const rMaxLat = rBbox[3];

              const pixelWidthDeg = (rMaxLng - rMinLng) / width;
              const pixelHeightDeg = (rMaxLat - rMinLat) / height; 

              for (let y = 0; y < height; y++) {
                for (let x = 0; x < width; x++) {
                  const pixelLng = rMinLng + (x * pixelWidthDeg);
                  const pixelLat = rMaxLat - (y * Math.abs(pixelHeightDeg)); 

                  if (pixelLng >= bbox[0][0] && pixelLng <= bbox[1][0] && 
                      pixelLat >= bbox[0][1] && pixelLat <= bbox[1][1]) {
                      
                      if (pointInPolygon([pixelLng, pixelLat], polygonCoords)) {
                        totalPixelsInPolygon++;
                        
                        const pixelIndex = (y * width) + x;
                        if (data[pixelIndex] === targetValue) {
                          vulnerablePixels++;
                        }
                      }
                  }
                }
              }

              if (totalPixelsInPolygon > 0) {
                const percent = (vulnerablePixels / totalPixelsInPolygon) * 100;
                percentageVulnerable.value = percent.toFixed(2); 
              } else {
                percentageVulnerable.value = "0.00";
              }
              isCalculating.value = false;

           }, 50); 
        } else {
            isCalculating.value = false;
        }
      }
    });
  });
});

const changeLayer = (layer) => {
  if (!map) return;
  currentLayer.value = layer;

  ['Exposure', 'Priority'].forEach(l => {
    if (map.getLayer(`layer-${l}`)) {
      map.setLayoutProperty(`layer-${l}`, 'visibility', 'none');
    }
  });

  if (map.getLayer(`layer-${layer}`)) {
    map.setLayoutProperty(`layer-${layer}`, 'visibility', 'visible');
  }
};
</script>

