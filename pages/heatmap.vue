<template>
  <div id="map-wrap" class="wrapper">
    <loading :active.sync="isLoading" :can-cancel="false" />
    <client-only>
      <l-map id="map" ref="Lmap" v-on:draw:created="addDrawnFence($event)" :zoom="13" @update:center="updateCenter">
        <v-container id="dataArea" fluid class="search" style="padding-left: 50px;">
          <v-row
            class="mb-3"
            style="width: 300px;"
            no-gutters
          >
            <v-col class="pa-1 search" md="15">
              <v-textarea
                id="searchBox"
                v-model="data"
                @keyup.enter="ProcessData()"
                left
                label="Coordinates"
                inline
              />
            </v-col>
            <v-col class="pa-3 search" md="auto">
              <v-btn v-on:click="ProcessData()" right color="green">
                Go
              </v-btn>
              <v-switch
                v-model="heatmap"
                :color="heatmap ? 'orange' : 'blue'"
                :label="`${heatmap ? 'Heatmap': 'Markers'}`"
                rounded
                class="potato"
                inset
                right
              />
            </v-col>
          </v-row>
        </v-container>

        <l-tile-layer url="https://tiles.poracle.world/tile/klokantech-basic/{z}/{x}/{y}/2/png" />

        <!-- error message snackbar -->
        <v-snackbar
          v-model="error"
          :timeout="3000"
          :top="true"
          color="red"
        >
          {{ errorMsg }}
          <v-btn
            @click="snackbar = false"
            color="black"
            text
          >
            Close
          </v-btn>
        </v-snackbar>
      </l-map>
    </client-only>
  </div>
</template>
<script>

import Loading from 'vue-loading-overlay'
import 'vue-loading-overlay/dist/vue-loading.css'
let HeatmapOverlay
let L = { icon () {} }
if (process.browser) {
  L = require('leaflet')
  HeatmapOverlay = require('leaflet-heatmap')
}

export default {
  components: {
    Loading
  },
  data () {
    return {
      data: '',
      heatmap: false,
      error: false,
      merkers: [],
      errorMsg: '',
      rawAreaLayer: { clearLayers: () => {} },
      heatmapLayer: { clearLayers: () => {} },
      L,
      isLoading: false,
      currentLocation: { lat: 0, lon: 0 }
    }
  },

  mounted () {
    this.initPosition()
  },
  updated () {
    // this.changeView('potato')
  },
  methods: {
    updateCenter (val) {
      this.currentLocation.lat = val.lat
      this.currentLocation.lon = val.lng
    },
    ProcessData () {
      this.mapStopScrolling()
      this.markers = []
      this.heatmapLayer.clearLayers()
      this.heatmapLayer = new L.FeatureGroup()
      const lines = this.data.split('\n').filter(l => l)
      lines.map((line) => {
        const trimmedLine = line.replace(/\s/gi, '')
        const value = trimmedLine.split(',')
        if (!+value[0] || !+value[1]) {
          console.error('can\'t parse coordinates from line ' + trimmedLine)
        } else {
          value[0] = +value[0]
          value[1] = +value[1]

          this.markers.push({ lat: value[0], lon: value[1], count: 1, sprite: value[2] })
        }
      })

      if (!this.markers.length) {
        console.error('no markers')
        return
      }

      const heatCfg = {
        // if scaleRadius is false it will be the constant radius used in pixels
        'radius': 0.007,
        'maxOpacity': 0.6,
        'scaleRadius': true,
        // if set to false the heatmap uses the global maximum for colorization
        // if activated: uses the data maximum within the current map boundaries
        //   (there will always be a red spot with useLocalExtremas true)
        'useLocalExtrema': true,
        latField: 'lat',
        lngField: 'lon',
        // which field name in your data represents the data value - default "value"
        valueField: 'count'
      }

      if (this.heatmap) {
        this.markers = this.markers.map((mark) => {
          mark.count = this.markers.filter(m => m.lat === mark.lat && m.lon === mark.lon).length
          return mark
        })
        const newHeat = new HeatmapOverlay(heatCfg)
        newHeat.addData(this.markers)

        this.heatmapLayer.addLayer(newHeat)
      } else {
        this.markers.forEach((mark) => {
          const markerIcon = this.L.icon({
            iconUrl: mark.sprite,
            iconSize: [38, 38], // size of the icon
            shadowSize: [40, 35], // size of the shadow
            iconAnchor: [19, 19], // point of the icon which will correspond to marker's location
            shadowAnchor: [4, 62] // the same for the shadow
          })
          const icon = mark.sprite ? this.L.marker([mark.lat, mark.lon], { icon: markerIcon }) : this.L.marker([mark.lat, mark.lon])
          this.heatmapLayer.addLayer(icon)
        })
      }

      this.heatmapLayer.addTo(this.$refs.Lmap.mapObject)

      const lats = this.markers.map(coords => coords.lat)
      const lons = this.markers.map(coords => coords.lon)
      const maxLat = Math.max(...lats)
      const minLat = Math.min(...lats)
      const maxLon = Math.max(...lons)
      const minLon = Math.min(...lons)

      this.$refs.Lmap.mapObject.flyToBounds(L.latLngBounds(L.latLng(minLat, minLon), L.latLng(maxLat, maxLon)))
    },
    mapStopScrolling () {
      const el = document.getElementById('dataArea')
      this.L.DomEvent.disableScrollPropagation(el)
    },
    async initPosition () {
      const { coords } = await this.$geolocation.getCurrentPosition()
      if (!this.$refs.Lmap) { return }
      this.$refs.Lmap.mapObject.setView(L.latLng(coords.latitude, coords.longitude), 12)
    }

  }
}

</script>

<style scoped>

  #searchBox{
    width: 100px;
    background: transparent;

  }
  .search{
    top: -10px;
    z-index: 9000;
    background: transparent;
  }
  .wrapper {
    height: 100%;
    overflow-Y: hidden;
    position: fixed;
    left:0;
    width: 100%;
    height: 100%;
  }

</style>
