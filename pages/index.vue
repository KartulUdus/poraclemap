<template>
  <div id="map-wrap" style="height: 100vh">
    <client-only>
      <l-map id="map" ref="Lmap" :zoom="13" @update:center="updateCenter">
        <v-container fluid class="search">
          <v-row class="mb-6" no-gutters>
            <v-col cols="6">
              <v-text-field
                id="searchBox"
                v-model="searchString"
                outlined
                class="search pa-2"
                label="search"
                filled
              />
            </v-col>
            <v-col>
              <v-btn
                v-on:click="search()"
                class="search"
              >
                search
              </v-btn>
            </v-col>
          </v-row>
          <span v-if="$geolocation.loading">        Loading location...</span>
          <span v-else-if="!$geolocation.supported">              Geolocation API is not supported</span>
        </v-container>
        <v-input class="search" />
        <v-flex class="float-right">
          <li v-for="area in foundAreas">
            <v-btn v-on:click="panToArea(area)" class="foundAreas" value="potato" justify-end>
              {{ area.class }} {{ area.type }}
            </v-btn>
          </li>
        </v-flex>
        <l-tile-layer url="https://tiles.poracle.world/tile/klokantech-basic/{z}/{x}/{y}/2/png" />
      </l-map>
    </client-only>
  </div>
</template>
<script>

import Axios from 'axios'
let L = { icon () {} }
if (process.browser) { L = require('leaflet') }
export default {

  data () {
    return {
      searchString: '',
      foundAreas: [],
      rawAreaLayer: { clearLayers: () => {} }
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
      console.log('my new center: ' + val)
      // trigger here your vuex mutations or use it directly in @update:center="vuexMutation"
    },
    async search () {
      const { data } = await Axios.get(`https://geocoding.poracle.world/nominatim/search?q=${encodeURIComponent(this.searchString)}&format=json&addressdetails=1&limit=10&polygon_geojson=1`)
      this.foundAreas = data
      console.log(this.foundAreas)
    },
    panToArea (area) {
      console.log(this.$refs.Lmap.mapObject)
      this.rawAreaLayer.clearLayers()
      this.rawAreaLayer = L.geoJSON(area.geojson)
      this.rawAreaLayer.addTo(this.$refs.Lmap.mapObject)
      this.$refs.Lmap.mapObject.flyToBounds(L.latLngBounds(L.latLng(area.boundingbox[0], area.boundingbox[2]), L.latLng(area.boundingbox[1], area.boundingbox[3])))
    },
    async initPosition () {
      const { coords } = await this.$geolocation.getCurrentPosition()
      await this.$refs.Lmap.mapObject.setView(L.latLng(coords.latitude, coords.longitude), 12)
    }
  }
}

</script>

<style scoped>

  #searchBox{
    width: 100px
  }
  .search{
    z-index: 999999999;
    left:0%;
    padding-left: 50%;
    background: transparent;
  }
  #searchBox{
    background:black;
  }
  .foundAreas{
    z-index: 999;

    right: 0;
  }
</style>
