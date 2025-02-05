<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'

import MapContainer from './MapContainer.vue'

// TODO: fixme
// @ts-ignore
import place_names from '../name_index.json'

const selectedPlaces = ref<string[][]>([])
const filterInput = ref('')

interface placeDataType {
  id: string
  title: string
  data: object
  changedName: boolean | null
  known: boolean | null
}

const placeData = ref<placeDataType[]>([])

function clearFilter() {
  filterInput.value = ''
}

function clearSelectedPlaces() {
  selectedPlaces.value = []
  placeData.value = []
  mapUpdate.value += 1
  updateURL()
}

function removePlace(index: number) {
  selectedPlaces.value.splice(index, 1)
  placeData.value.splice(index, 1)
  updateURL()
  mapUpdate.value += 1
}

function editPlaceName(index: number, title: string) {
  editingIndex.value = index
  editName.value = title
}

function savePlaceName(index: number) {
  placeData.value[index].title = editName.value
  placeData.value[index].changedName = true
  editingIndex.value = null
  updateURL()
  mapUpdate.value += 1
}

const filteredPlaces = computed(() => {
  if (filterInput.value.length > 0) {
    const filter_text = filterInput.value.toLowerCase()
    // TODO: fixme
    // @ts-ignore
    const data = place_names.filter((place) => {
      return place[0].toLowerCase().startsWith(filter_text)
    })
    // TODO: fixme
    // @ts-ignore
    return data.sort((placeA, placeB) => {
      if (placeA[0] < placeB[0]) {
        return -1
      } else {
        return 1
      }
    })
  } else {
    return []
  }
})

function moveToPlace(marker: placeDataType) {

  if(!location) return

  let [max_lat, min_lat, max_long, min_long] = [0, 180, 0, 180]
  let [center_lat, center_long] = [37.9, 23.7]

  // @ts-ignore
  const [long, lat] = marker.data.features[0].geometry.coordinates
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

  [center_lat, center_long] = [(max_lat + min_lat) / 2, (max_long + min_long) / 2]

  mapRef.value.moveToMapLocation(center_lat, center_long)
}

async function addPlace(place: string[], name: string | null) {
  await fetch(
    `https://raw.githubusercontent.com/ryanfb/pleiades-geojson/gh-pages/geojson/${place[1]}.geojson`
  )
    .then((response) => response.json())
    .then((data) => {
      const known = data.features[0].geometry ? true : false
      if (name) {
        placeData.value.push({
          id: place[1],
          title: name,
          data: data,
          changedName: true,
          known: known
        })
      } else {
        placeData.value.push({
          id: place[1],
          title: data.title,
          data: data,
          changedName: false,
          known: known
        })
      }
      mapUpdate.value += 1
      updateURL()
    })
  selectedPlaces.value.push(place)
}

const editingIndex = ref<number | null>(null)
const editName = ref('')

// This ref is just incremented every change we make to force a map reload, feels janky
const mapUpdate = ref(0)
const mapRef = ref()

import { useRouter, useRoute } from 'vue-router'

const router = useRouter()

function updateURL() {
  let placeList = []
  for (const place of placeData.value) {
    if (place.changedName) {
      placeList.push(`${place.id}|${place.title}`)
    } else {
      placeList.push(place.id)
    }
  }
  router.push({
    query: {
      places: placeList
    }
  })
}

const route = useRoute()

onMounted(async () => {
  await router.isReady()
  if (route.query.places) {
    if (Array.isArray(route.query.places)) {
      for (const placeString of route.query.places) {
        // TODO: fixme
        // @ts-ignore
        if (placeString.includes('|')) {
          // TODO: fixme
          // @ts-ignore
          const [id, newTitle] = placeString.split('|')
          addPlace(
            // TODO: fixme
            // @ts-ignore
            place_names.find((place) => place[1] == id),
            newTitle
          ).then(() => {
            // TODO: fixme
            // @ts-ignore
            placeData.value.find((p) => p.id == id).title = newTitle
            // TODO: fixme
            // @ts-ignore
            placeData.value.find((p) => p.id == id).changedName = true
            mapUpdate.value += 1
          })
        } else {
          // TODO: fixme
          // @ts-ignore
          await addPlace(place_names.find((place) => place[1] == placeString))
        }
      }
    } else {
      if (route.query.places.includes('|')) {
        const [id, newTitle] = route.query.places.split('|')
        await addPlace(
          // TODO: fixme
          // @ts-ignore
          place_names.find((place) => place[1] == id),
          newTitle
        )
        // TODO: fixme
        // @ts-ignore
        placeData.value.find((p) => p.id == id).title = newTitle
        // TODO: fixme
        // @ts-ignore
        placeData.value.find((p) => p.id == id).changedName = true
        mapUpdate.value += 1
      } else {
        // TODO: fixme
        // @ts-ignore
        await addPlace(place_names.find((place) => place[1] == route.query.places))
      }
    }
    mapUpdate.value += 1
  }
})
</script>

<template>
  <h4>Introduction:</h4>
  <p>
    This tool allows you to make simple maps of points derived from the
    <a href="https://pleiades.stoa.org/">Pleiades project</a>. It uses a
    <a href="https://github.com/ryanfb/pleiades-geojson">dump</a> of the Pleiades data provided by
    <a href="https://github.com/ryanfb">ryanfb</a>. It was created by
    <a href="https://willismonroe.com">Willis Monroe</a>.
  </p>
  <h4>Instructions:</h4>
  <p>
    Use the search bar below to find places in the Pleiades data and add them to your "Selected
    Places" to see them appear on the map. You can edit the name of each place to change the string
    displayed in the label. You can remove each place manually, or clear all of the selected places.
    As you modify the map the URL will change to reflect the current status of the map. You can save
    or distribute this URL to come back to the same map (it preserves name changes as well); here's
    an
    <a
      href="https://willismonroe.github.io/anax/?places=912986|Uruk&places=893964|Borsippa&places=893951|Babylon+(%F0%92%86%8D%F0%92%80%AD%F0%92%8A%8F%F0%92%86%A0)&places=912910|Nippur&places=893976|Ctesiphon"
      >example url</a
    >
    featuring some cuneiform in a place label. To use this map in a presentation or publication,
    screenshot the resulting map and include the relevant attribution (and the URL if you want).
  </p>
  <p>
    If you encounter bugs or think of improvements to this tool please add them to the
    <a href="https://github.com/willismonroe/anax/issues">issues page</a> on Github. If you want to
    help out with development feel free to pitch in as well!
  </p>
  <div>
    <div class="container">
      <div class="row">
        <div class="col-sm">
          <h4>Search Places:</h4>
          <input type="text" v-model.trim="filterInput" autofocus />
          <button class="btn btn-danger" v-on:click="clearFilter">Clear</button>

          <ul v-if="filterInput">
            <li v-for="(place, index) in filteredPlaces.slice(0, 10)" v-bind:key="index">
              {{ place[0] }}
              <button class="btn btn-primary" v-on:click="addPlace(place, null)">→</button>
            </li>
          </ul>
        </div>
        <div class="col-sm">
          <h4>Selected Places:</h4>
          <ul v-if="placeData.length > 0">
            <li v-for="(place, index) in placeData" v-bind:key="index">
              <span v-if="editingIndex == index">
                <input v-model="editName" type="text" />
                <button class="btn btn-success" @click="savePlaceName(index)">Save</button>
              </span>
              <span v-else
                >{{ place.title }}
                <span v-if="!place.known">
                  <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="16"
                    height="16"
                    fill="currentColor"
                    class="bi bi-exclamation-triangle-fill"
                    viewBox="0 0 16 16"
                  >
                    <path
                      d="M8.982 1.566a1.13 1.13 0 0 0-1.96 0L.165 13.233c-.457.778.091 1.767.98 1.767h13.713c.889 0 1.438-.99.98-1.767L8.982 1.566zM8 5c.535 0 .954.462.9.995l-.35 3.507a.552.552 0 0 1-1.1 0L7.1 5.995A.905.905 0 0 1 8 5zm.002 6a1 1 0 1 1 0 2 1 1 0 0 1 0-2z"
                    />
                  </svg>
                </span>
                <button class="btn btn-primary" @click="editPlaceName(index, place.title)">
                  Edit
                </button>
                <button class="btn btn-success" @click="moveToPlace(place)">
                  Move to
                </button>
                <button class="btn btn-danger" @click="removePlace(index)">Remove</button>
              </span>
            </li>
          </ul>
          <span class="clear-btn">
            <button
              class="btn btn-danger"
              v-if="placeData.length > 0"
              v-on:click="clearSelectedPlaces"
            >
              Clear Selected Places
            </button>
          </span>
        </div>
      </div>
    </div>
  </div>
  <MapContainer :places="placeData" :key="mapUpdate" ref="mapRef"/>
  <h4>Attribution:</h4>
  <p>
    The map data is provided by <a href="https://pleiades.stoa.org/credits">Pleiades</a> under a
    CC-BY license. The map tiles are provided by
    <a href="http://cawm.lib.uiowa.edu/index.html">Consortium of Ancient World Mappers</a> also
    under a CC-BY license.
  </p>
</template>

<style scoped>
h4 {
  padding-top: 1rem;
}
span {
  padding: 1rem;
}
.btn {
  margin: 0.2rem;
}
.clear-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
</style>
