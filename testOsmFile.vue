<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>OSM File Previewer - Vue</title>
  <script src="https://cdn.jsdelivr.net/npm/vue@3/dist/vue.global.js"></script>
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Arial, sans-serif;
      background: #f5f5f5;
    }

    #app {
      display: flex;
      flex-direction: column;
      height: 100vh;
    }

    .header {
      background: white;
      padding: 20px;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
      border-bottom: 1px solid #e0e0e0;
    }

    .header h1 {
      font-size: 24px;
      color: #333;
      margin-bottom: 12px;
    }

    .controls {
      display: flex;
      align-items: center;
      gap: 16px;
      flex-wrap: wrap;
    }

    .file-input-wrapper {
      position: relative;
    }

    .file-input-wrapper input[type="file"] {
      display: none;
    }

    .file-button {
      background: #2563eb;
      color: white;
      padding: 10px 20px;
      border-radius: 6px;
      cursor: pointer;
      transition: background 0.2s;
      font-size: 14px;
      font-weight: 500;
      border: none;
    }

    .file-button:hover {
      background: #1d4ed8;
    }

    .file-name {
      color: #666;
      font-size: 14px;
    }

    .stats {
      display: flex;
      gap: 16px;
      font-size: 14px;
      color: #666;
    }

    .stats span {
      font-weight: 500;
    }

    .error {
      margin-top: 12px;
      padding: 12px;
      background: #fee;
      border: 1px solid #fcc;
      border-radius: 6px;
      color: #c33;
      font-size: 14px;
    }

    .map-container {
      flex: 1;
      position: relative;
    }

    #map {
      width: 100%;
      height: 100%;
    }

    .loading {
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      background: rgba(255, 255, 255, 0.9);
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 1000;
    }

    .loading-content {
      text-align: center;
    }

    .spinner {
      width: 48px;
      height: 48px;
      border: 4px solid #e0e0e0;
      border-top-color: #2563eb;
      border-radius: 50%;
      animation: spin 1s linear infinite;
      margin: 0 auto 12px;
    }

    @keyframes spin {
      to { transform: rotate(360deg); }
    }

    .loading-text {
      color: #666;
      font-weight: 500;
    }

    .empty-state {
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      display: flex;
      align-items: center;
      justify-content: center;
      pointer-events: none;
      z-index: 10;
    }

    .empty-content {
      text-align: center;
      color: #999;
    }

    .empty-icon {
      width: 64px;
      height: 64px;
      margin: 0 auto 16px;
      color: #ccc;
    }

    .empty-title {
      font-size: 18px;
      font-weight: 500;
      margin-bottom: 8px;
    }

    .empty-subtitle {
      font-size: 14px;
    }

    .leaflet-popup-content {
      max-width: 250px;
      font-size: 13px;
    }

    .popup-row {
      margin: 3px 0;
    }

    .popup-row strong {
      color: #333;
    }

    .popup-row.secondary {
      color: #666;
    }
  </style>
</head>
<body>
  <div id="app">
    <div class="header">
      <h1>OSM File Previewer</h1>
      <div class="controls">
        <div class="file-input-wrapper">
          <label class="file-button">
            <input type="file" accept=".osm,.xml" @change="handleFileChange">
            Choose OSM File
          </label>
        </div>
        <span v-if="fileName" class="file-name">{{ fileName }}</span>
        <div v-if="stats.nodes > 0" class="stats">
          <span>Nodes: {{ stats.nodes }}</span>
          <span>Ways: {{ stats.ways }}</span>
          <span v-if="stats.relations > 0">Relations: {{ stats.relations }}</span>
        </div>
      </div>
      <div v-if="error" class="error">{{ error }}</div>
    </div>

    <div class="map-container">
      <div v-if="loading" class="loading">
        <div class="loading-content">
          <div class="spinner"></div>
          <div class="loading-text">Parsing OSM data...</div>
        </div>
      </div>

      <div v-if="!fileName && !loading" class="empty-state">
        <div class="empty-content">
          <svg class="empty-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 20l-5.447-2.724A1 1 0 013 16.382V5.618a1 1 0 011.447-.894L9 7m0 13l6-3m-6 3V7m6 10l4.553 2.276A1 1 0 0021 18.382V7.618a1 1 0 00-.553-.894L15 4m0 13V4m0 0L9 7" />
          </svg>
          <div class="empty-title">Select an OSM file to preview</div>
          <div class="empty-subtitle">Upload .osm or .xml files</div>
        </div>
      </div>

      <div id="map"></div>
    </div>
  </div>

  <script>
    const { createApp } = Vue;

    createApp({
      data() {
        return {
          fileName: '',
          loading: false,
          error: '',
          stats: { nodes: 0, ways: 0, relations: 0 },
          map: null,
          layerGroup: null
        };
      },

      mounted() {
        this.initMap();
      },

      beforeUnmount() {
        if (this.map) {
          this.map.remove();
        }
      },

      methods: {
        initMap() {
          this.map = L.map('map', {
            center: [20, 0],
            zoom: 2,
            zoomControl: true,
            preferCanvas: true
          });

          L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            attribution: '© OpenStreetMap contributors',
            maxZoom: 19,
            minZoom: 1,
            keepBuffer: 2,
            updateWhenIdle: false,
            updateWhenZooming: false
          }).addTo(this.map);

          this.layerGroup = L.layerGroup().addTo(this.map);
        },

        handleFileChange(event) {
          const file = event.target.files[0];
          if (file) {
            if (file.name.endsWith('.osm') || file.name.endsWith('.xml')) {
              this.fileName = file.name;
              this.error = '';
              this.parseOSMFile(file);
            } else {
              this.error = 'Please select a valid .osm or .xml file';
            }
          }
        },

        async parseOSMFile(file) {
          this.loading = true;
          this.stats = { nodes: 0, ways: 0, relations: 0 };

          try {
            const text = await file.text();
            const parser = new DOMParser();
            const xml = parser.parseFromString(text, 'text/xml');

            const parserError = xml.querySelector('parsererror');
            if (parserError) {
              throw new Error('Invalid XML format');
            }

            const nodes = {};
            const ways = [];
            const pois = [];
            const bounds = { minLat: 90, maxLat: -90, minLon: 180, maxLon: -180 };

            // Parse nodes
            const nodeElements = xml.querySelectorAll('node');
            nodeElements.forEach(node => {
              const id = node.getAttribute('id');
              const lat = parseFloat(node.getAttribute('lat'));
              const lon = parseFloat(node.getAttribute('lon'));

              const tags = {};
              node.querySelectorAll('tag').forEach(tag => {
                tags[tag.getAttribute('k')] = tag.getAttribute('v');
              });

              nodes[id] = { lat, lon, tags };

              bounds.minLat = Math.min(bounds.minLat, lat);
              bounds.maxLat = Math.max(bounds.maxLat, lat);
              bounds.minLon = Math.min(bounds.minLon, lon);
              bounds.maxLon = Math.max(bounds.maxLon, lon);

              if (Object.keys(tags).length > 0 && (tags.name || tags.amenity || tags.shop)) {
                pois.push({ lat, lon, tags });
              }
            });

            // Parse ways
            const wayElements = xml.querySelectorAll('way');
            wayElements.forEach(way => {
              const wayData = {
                id: way.getAttribute('id'),
                nodes: [],
                tags: {}
              };

              way.querySelectorAll('nd').forEach(nd => {
                const ref = nd.getAttribute('ref');
                if (nodes[ref]) {
                  wayData.nodes.push(nodes[ref]);
                }
              });

              way.querySelectorAll('tag').forEach(tag => {
                wayData.tags[tag.getAttribute('k')] = tag.getAttribute('v');
              });

              if (wayData.nodes.length > 0) {
                ways.push(wayData);
              }
            });

            const relationCount = xml.querySelectorAll('relation').length;

            this.stats = {
              nodes: nodeElements.length,
              ways: wayElements.length,
              relations: relationCount
            };

            this.renderMap(ways, pois, bounds);

          } catch (err) {
            this.error = 'Error parsing OSM file: ' + err.message;
            this.loading = false;
          }
        },

        renderMap(ways, pois, bounds) {
          if (!this.map || !this.layerGroup) return;

          this.layerGroup.clearLayers();

          // Add ways
          ways.forEach(way => {
            const coords = way.nodes.map(n => [n.lat, n.lon]);

            let options = {
              color: '#3388ff',
              weight: 2,
              opacity: 0.7,
              smoothFactor: 1
            };

            if (way.tags.highway) {
              const hwType = way.tags.highway;
              if (['motorway', 'trunk'].includes(hwType)) {
                options = { color: '#e892a2', weight: 5, opacity: 0.8 };
              } else if (['primary', 'secondary'].includes(hwType)) {
                options = { color: '#fca75d', weight: 4, opacity: 0.8 };
              } else if (['tertiary', 'residential'].includes(hwType)) {
                options = { color: '#fff68f', weight: 3, opacity: 0.7 };
              } else {
                options = { color: '#d3d3d3', weight: 2, opacity: 0.6 };
              }
            } else if (way.tags.building) {
              options = {
                color: '#ff6b6b',
                fillColor: '#ff6b6b',
                fillOpacity: 0.3,
                weight: 1,
                opacity: 0.8
              };
            } else if (way.tags.natural === 'water' || way.tags.waterway) {
              options = {
                color: '#4dabf7',
                fillColor: '#4dabf7',
                fillOpacity: 0.4,
                weight: 1,
                opacity: 0.8
              };
            } else if (way.tags.landuse === 'forest' || way.tags.natural === 'wood') {
              options = {
                color: '#51cf66',
                fillColor: '#51cf66',
                fillOpacity: 0.3,
                weight: 1,
                opacity: 0.7
              };
            } else if (way.tags.railway) {
              options = { color: '#868e96', weight: 2, opacity: 0.7, dashArray: '5, 5' };
            }

            let feature;
            const isClosed = coords.length > 2 &&
              coords[0][0] === coords[coords.length - 1][0] &&
              coords[0][1] === coords[coords.length - 1][1] &&
              (way.tags.building || way.tags.landuse || way.tags.natural === 'water');

            if (isClosed) {
              feature = L.polygon(coords, options);
            } else {
              feature = L.polyline(coords, options);
            }

            feature.addTo(this.layerGroup);

            if (Object.keys(way.tags).length > 0) {
              let popup = '<div>';
              const importantTags = ['name', 'highway', 'building', 'amenity', 'shop', 'natural', 'landuse'];

              importantTags.forEach(key => {
                if (way.tags[key]) {
                  popup += `<div class="popup-row"><strong>${key}:</strong> ${way.tags[key]}</div>`;
                }
              });

              Object.entries(way.tags).forEach(([key, value]) => {
                if (!importantTags.includes(key)) {
                  popup += `<div class="popup-row secondary"><strong>${key}:</strong> ${value}</div>`;
                }
              });

              popup += '</div>';
              feature.bindPopup(popup);
            }
          });

          // Add POI markers
          pois.forEach(poi => {
            const marker = L.circleMarker([poi.lat, poi.lon], {
              radius: 6,
              fillColor: '#fa5252',
              color: '#fff',
              weight: 2,
              opacity: 1,
              fillOpacity: 0.8
            }).addTo(this.layerGroup);

            let popup = '<div>';
            Object.entries(poi.tags).forEach(([key, value]) => {
              popup += `<div class="popup-row"><strong>${key}:</strong> ${value}</div>`;
            });
            popup += '</div>';
            marker.bindPopup(popup);
          });

          // Fit bounds
          if (bounds.minLat !== 90) {
            this.map.fitBounds(
              [[bounds.minLat, bounds.minLon], [bounds.maxLat, bounds.maxLon]],
              { padding: [50, 50], maxZoom: 16 }
            );
          }

          this.loading = false;
        }
      }
    }).mount('#app');
  </script>
</body>
</html>
