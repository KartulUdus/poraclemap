<template>
  <div id="map-wrap" style="height: 100vh">
    <client-only>
      <l-map :zoom="13" :center="[59.413220, 24.219482]">
        <l-tile-layer url="https://tiles.poracle.world/tile/klokantech-basic/{z}/{x}/{y}/2/png" />
        <l-marker :lat-lng="[59.413220, 24.219482]" />
      </l-map>
    </client-only>
  </div>
</template>
<script>
export default {
  mounted () {
    console.log(this.$geolocation)
  }
}
</script>
