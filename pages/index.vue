<template>
  <div id="map-wrap" style="height: 100%;">
    <loading :active.sync="isLoading" :can-cancel="false" />
    <client-only>
      <l-map id="map" ref="Lmap" :zoom="13" @update:center="updateCenter">
        <v-container fluid class="search">
          <v-row
            class="mb-6"
            justify="center"
            no-gutters
          >
            <v-col md="auto">
              <v-card
                class="pa-2"
                outlined
                tile
              >
                <v-text-field
                  id="searchBox"
                  v-model="searchString"
                  @keyup.enter="search()"
                  class="search"
                  label="search"
                  filled
                />
              </v-card>
            </v-col>
          </v-row>
          <span v-if="$geolocation.loading">        Loading location...</span>
          <span v-else-if="!$geolocation.supported">              Geolocation API is not supported</span>
        </v-container>
        <!-- end of search stuff -->

        <!-- beginning of search results -->
        <transition name="slide-fade">
          <v-card
            v-if="foundAreas.length"
            class="searchResultBox mx-auto"
            max-width="300"
          >
            <v-list rounded>
              <v-list-item-group v-model="foundAreas" color="primary">
                <v-list-item
                  v-for="area in foundAreas"
                  v-bind:key="area.osm_id"
                  class="foundAreas"
                >
                  <v-list-item-content v-on:click="panToArea(area)" rounded>
                    {{ area.class }} {{ area.type }}
                  </v-list-item-content>
                </v-list-item>
              </v-list-item-group>
            </v-list>
          </v-card>
        </transition>
        <!-- End of search results -->
        <!-- Beginning of created geofences -->
        <v-card
          class="geofencesArea transparent mx-auto"
          max-width="300"
        >
          <v-list rounded>
            <v-list-item-group color="primary">
              <v-list-item>
                <input ref="myFile" v-on:change="selectedFile()" type="file">
              </v-list-item>

              <v-list-item
                v-for="fence in geofences"
                v-bind:key="fence.name"
              >
                <v-list-item-content v-on:click="panToGeofence(fence)" v-text="fence.name" rounded />
              </v-list-item>
            </v-list-item-group>
          </v-list>
        </v-card>

        <l-tile-layer url="https://tiles.poracle.world/tile/klokantech-basic/{z}/{x}/{y}/2/png" />
      </l-map>

      <a ref="currentAreaPopup">
            <v-text-field
              v-model="minStep"
              label="min step"
              type="number"
              min="0"
              max="10000"
              class="popup flex"
            />
            <v-btn class="flex" v-on:click="addGeofence(currentResult)">add area</v-btn>
      </a>
    </client-only>
  </div>
</template>
<script>

import Axios from 'axios'
import Loading from 'vue-loading-overlay'
import 'vue-loading-overlay/dist/vue-loading.css'
let L = { icon () {} }
if (process.browser) { L = require('leaflet') }
export default {
  components: {
    Loading
  },

  data () {
    return {
      searchString: '',
      foundAreas: [],
      geofences: [],
      currentResult: {},
      minStep: 50,
      rawAreaLayer: { clearLayers: () => {} },
      rawGeofenceLayer: { clearLayers: () => {} },
      showResults: false,
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
    async search () {
      this.$nuxt.$loading.start()
      this.isLoading = true
      this.foundAreas = []
      const { data } = await Axios.get(`https://geocoding.poracle.world/nominatim/search?q=${encodeURIComponent(this.searchString)}&format=json&addressdetails=1&limit=10&polygon_geojson=1`)
      this.foundAreas = data
      this.showResults = !!data.length
      this.$nuxt.$loading.finish()
      this.isLoading = false
    },
    panToArea (area) {
      if (!this.$refs.Lmap) { return }
      this.currentResult = area
      this.rawAreaLayer.clearLayers()
      this.rawAreaLayer = L.geoJSON(area.geojson)
      this.rawAreaLayer.addTo(this.$refs.Lmap.mapObject)
      this.rawAreaLayer.bindPopup(this.$refs.currentAreaPopup)
      console.log(this.currentLocation, area)
      if (this.getDistance(this.currentLocation, area) > 1000000) { // 1000 km
        this.$refs.Lmap.mapObject.fitBounds(L.latLngBounds(L.latLng(area.boundingbox[0], area.boundingbox[2]), L.latLng(area.boundingbox[1], area.boundingbox[3])))
      } else {
        this.$refs.Lmap.mapObject.flyToBounds(L.latLngBounds(L.latLng(area.boundingbox[0], area.boundingbox[2]), L.latLng(area.boundingbox[1], area.boundingbox[3])))
      }
    },
    panToGeofence (geofence) {
      if (!this.$refs.Lmap) { return }

      this.rawGeofenceLayer.clearLayers()
      this.rawGeofenceLayer = L.geoJSON(L.polygon(geofence.path).toGeoJSON(), { color: geofence.color })
      this.rawGeofenceLayer.addTo(this.$refs.Lmap.mapObject)
      this.rawGeofenceLayer.bindPopup('potato')

      const lats = geofence.path.map(coords => coords[0])
      const lons = geofence.path.map(coords => coords[1])
      const maxLat = Math.max(...lats)
      const minLat = Math.min(...lats)
      const maxLon = Math.max(...lons)
      const minLon = Math.min(...lons)

      if (this.getDistance(this.currentLocation, { lat: minLat, lon: minLon }) > 1000000) { // 1000 km
        this.$refs.Lmap.mapObject.fitBounds(L.latLngBounds(L.latLng(minLat, minLon), L.latLng(maxLat, maxLon)))
      } else {
        this.$refs.Lmap.mapObject.flyToBounds(L.latLngBounds(L.latLng(minLat, minLon), L.latLng(maxLat, maxLon)))
      }
    },
    async initPosition () {
      const { coords } = await this.$geolocation.getCurrentPosition()
      if (!this.$refs.Lmap) { return }
      this.$refs.Lmap.mapObject.setView(L.latLng(coords.latitude, coords.longitude), 12)
    },

    selectedFile () {
      this.isLoading = true
      const file = this.$refs.myFile.files[0]
      this.isLoading = false
      if (!file || !(file.type === 'text/plain' || file.type === 'application/json')) { return }

      const reader = new FileReader()
      reader.readAsText(file, 'UTF-8')
      reader.onload = (evt) => {
        let updFile = evt.target.result
        if (file.type === 'application/json') {
          updFile = JSON.parse(updFile)
        } else {
          const areaNames = updFile.match(/\[.+]/g).map(n => n.replace(/[\]["]/g, ''))
          const areaLocs = updFile.split(/\[.+]/g).filter(a => a).map((loc) => {
            const locArray = loc.split('\n').filter(a => a)
            return locArray.map((line) => {
              const coords = line.split(',')
              return [+coords[0], +coords[1]]
            })
          })
          const result = []
          areaNames.forEach((key, idx) => { result.push({ name: key, color: `#${Math.floor(Math.random() * 16777215).toString(16)}`, id: idx.toString(), path: areaLocs[idx] }) })
          updFile = result
        }
        if (!updFile.length) { console.error('no areas in geofence file') }
        updFile.forEach((area) => {
          if (!area.name || !area.color || !area.id.toString() || !area.path || !area.path.length) { console.error('unhappy with geofence file') }
        })
        updFile.filter(area => !this.geofences.find(a => a.name === area.name))
        this.geofences = [...this.geofences, ...updFile]
        console.log(updFile)
      }
      reader.onerror = (evt) => {
        console.error(evt)
      }
    },

    addGeofence () {
      console.log('potato')
    },
    getDistance (start, end) {
      if (typeof (Number.prototype.toRad) === 'undefined') {
        Number.prototype.toRad = function toRad () { // eslint-disable-line
          return this * Math.PI / 180
        }
      }
      const earthRadius = 6371 * 1000 // m
      let lat1 = parseFloat(start.lat)
      let lat2 = parseFloat(end.lat)
      const lon1 = parseFloat(start.lon)
      const lon2 = parseFloat(end.lon)

      const dLat = (lat2 - lat1).toRad()
      const dLon = (lon2 - lon1).toRad()
      lat1 = lat1.toRad()
      lat2 = lat2.toRad()

      const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.sin(dLon / 2) * Math.sin(dLon / 2) * Math.cos(lat1) * Math.cos(lat2)
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
      const d = earthRadius * c
      return Math.ceil(d)
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
  .foundAreas{
    z-index: 999;
    align-items:right;
    right: 0;
    opacity: 100 !important;
  }
  .searchResultBox{
    right: 0;
    float: right;
    z-index: 9999;
    background: transparent;
    opacity: .94;
  }
  .geofencesArea{
    left: 0;
    float: left;
    z-index: 999999;
    background: transparent;
    opacity: .94;
  }
  .popup {
    padding: 12px 20px;
    margin: 8px 0;
    box-sizing: border-box;
    border: 2px solid gray;
    border-radius: 4px;
  }

  .slide-fade-enter-active {
    transition: all .6s ease;
  }
  .slide-fade-leave-active {
    transition: all .8s cubic-bezier(1.0, 0.5, 0.8, 1.0);
  }
  .slide-fade-enter{
    transform: translateX(10px);
    opacity: 1;
  }
  .transparent {
    background-color: transparent!important;
    border-color: transparent!important;
  }
  .flex {
    display: flex;
  }

</style>
