<template>
  <div class="btn-container">
    <button class="btn btn-success button" v-on:click="changeTilePaneStyle()" >Change Color</button>
  </div>
  <div id="container" style="height: 700px" ref="mapContainerRef">
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue'

import 'leaflet/dist/leaflet.css'
import { Map, TileLayer, Marker, Control, LayerGroup, GeoJSON, DivIcon, Point, LatLng } from 'leaflet'

const TILE_URL = 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png'
const ATTRIBUTION = '&copy; <a href="https://www.openstreetmap.org/copyright">OSM</a>'

const map = ref<Map | null>()
const mapContainerRef = ref<null|HTMLElement>();

interface placeDataType {
  id: string
  title: string
  data: object
  changedName: boolean | null
  known: boolean | null
}

const props = defineProps<{
  places: placeDataType[]
}>()


defineExpose({moveToMapLocation})

const placeData = ref<placeDataType[]>()

const squareIcon = new DivIcon({
  html: '□',
  iconSize: new Point(14, 22),
  iconAnchor: new Point(7, 22)
})

const tiles = new TileLayer(TILE_URL, { attribution: ATTRIBUTION, maxZoom: 18 })
const overlay = new TileLayer('https://cawm.lib.uiowa.edu/tiles/{z}/{x}/{y}.png')
const scale = new Control.Scale()
const markerLayer = new LayerGroup()
// TODO: fixme
// @ts-ignore
const geoJSON = new GeoJSON(null, {
  pointToLayer: function (feature, latlng) {
    // TODO: fixme
    // @ts-ignore
    return new Marker(latlng, { icon: squareIcon }).bindTooltip(feature.title, {
      permanent: true,
      direction: 'right'
    })
  }
})

function setupMap() {
  let zoom = 6

  // Find center point... very janky
  let [max_lat, min_lat, max_long, min_long] = [0, 180, 0, 180]
  let [center_lat, center_long] = [37.9, 23.7]
  let known_places = false
  for (const place of props.places) {
    if (place.known) {
      known_places = true
      // TODO: fixme
      // @ts-ignore
      const [long, lat] = place.data.features[0].geometry.coordinates
      if (max_long < long) {
        max_long = long
      }
      if (min_long > long) {
        min_long = long
      }
      if (max_lat < lat) {
        max_lat = lat
      }
      if (min_lat > lat) {
        min_lat = lat
      }
    }
  }
  if (known_places) {
    ;[center_lat, center_long] = [(max_lat + min_lat) / 2, (max_long + min_long) / 2]
  }

  map.value = new Map('container', {
    preferCanvas: true,
    zoomAnimation: false,
    zoomControl: false,
    attributionControl: false,
    zoomSnap: 0.25,
    zoom: zoom
  })

  moveToMapLocation(center_lat, center_long)

  tiles.addTo(map.value)
  overlay.addTo(map.value)
  scale.addTo(map.value)
  markerLayer.addTo(map.value)
  geoJSON.addTo(map.value)
}

function moveToMapLocation(x: number, y: number) {

  const mapValue = map.value;

  if(!mapValue) return

  const mapZoom = mapValue.getZoom()

  mapValue.setView(new LatLng(x, y), mapZoom)
}

function reloadMarkers() {
  markerLayer.clearLayers()
  geoJSON.clearLayers()
  for (const place of props.places) {
    if (place.known) {
      // TODO: fixme
      // @ts-ignore
      place.data.features[0].title = place.title

      // TODO: fixme
      // @ts-ignore
      let geoJSONdata = place.data.features[0]
      // TODO: fixme
      // @ts-ignore
      geoJSONdata.title = place.title
      geoJSON.addData(geoJSONdata)
    }
  }
}

function changeMapColor(percentageNum:number,element:HTMLElement){
    element.style.filter = `grayscale(${percentageNum}%)`;
    element.style.webkitFilter = `grayscale(${percentageNum}%)`;
}

function changeTilePaneStyle() {
  const element = mapContainerRef.value?.childNodes[0].childNodes[0] as HTMLElement;
  if(element && ( element.style.filter === "grayscale(100%)" || element.style.filter === "" )){
    changeMapColor(0,element);
  }else{
    changeMapColor(100,element);
  }
}

onMounted(() => {
  setupMap()
  reloadMarkers()
})
</script>

<style>
.button{
  width: 150px;
  margin: auto;
  margin-bottom: 1em;
}
.leaflet-div-icon {
  background-color: transparent;
  border: transparent;
  font-size: 2em;
}

.leaflet-tooltip {
  background-color: transparent;
  border: transparent;
  box-shadow: none;
  color: white;
  font-size: 1.4em;
  -webkit-text-stroke: 1px black;
}

.leaflet-popup-tip-container {
  display: none;
}

.leaflet-tooltip-top:before,
.leaflet-tooltip-bottom:before,
.leaflet-tooltip-left:before,
.leaflet-tooltip-right:before {
  position: absolute;
  pointer-events: none;
  border: 6px solid transparent;
  background: transparent;
  content: '';
}

.leaflet-tile-pane {
  -webkit-filter: grayscale(100%);
  filter: grayscale(100%);
}

</style>
