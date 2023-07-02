<script setup lang="ts">
import { onBeforeMount, onMounted, getCurrentInstance, ref  } from 'vue';
import type { LngLat, YMap, SearchOptions, SearchResponse, YMapMarker } from "@yandex/ymaps3-types"

interface ImportMetaEnv {
  readonly VITE_APP_API_KEY: string
  // more env variables...
}
interface ImportMeta {
  readonly env: ImportMetaEnv
}

const v = import.meta.env.VITE_APP_API_KEY as ImportMeta
const map_exist = ref(false);
const location = ref<String>("");
let marker = ref<YMapMarker>();
let map: YMap | null = null;

const load_maps = async () => {
    return new Promise((resolve, reject) => {
      let recaptchaScript: HTMLScriptElement = document.createElement('script');
      recaptchaScript.setAttribute('type', 'text/javascript');
      recaptchaScript.setAttribute('src', `https://api-maps.yandex.ru/3.0/?apikey=${v}&lang=ru_RU`);
      document.head.appendChild(recaptchaScript);
      recaptchaScript.addEventListener("load", async () => {
        await ymaps3.ready.then( () => {
          resolve({ status: true })
        });
      });
      recaptchaScript.addEventListener('error', () => {
        reject({
          status: false,
          message: `Failed to load the script`
        })
      })
    })
}
await load_maps().then(data => {
  console.log("lodaded", data);
})


const create_map = async () => {
  const {
      YMap,
      YMapDefaultSchemeLayer,
      YMapControls,
      YMapListener,
    } = ymaps3;
  const { YMapZoomControl } = await ymaps3.import('@yandex/ymaps3-controls@0.0.1')
  const my_coords = await ymaps3.geolocation.getPosition()

  const raw_location = await ymaps3.search({ text: my_coords.coords} as unknown as SearchOptions) as SearchResponse
  console.log(raw_location);
  
  map = new YMap(document.getElementById('map')!,  {location: { center: my_coords.coords, zoom: 10 }},[
    new YMapDefaultSchemeLayer({}),
    new YMapControls({position: 'right'}, [
      new YMapZoomControl({})
    ]),
  ]);

  const content = document.createElement('div');
  content.innerHTML = '<div style="width: 10px; height: 10px; border: 1px solid red; border-radius: 50%; background-color: red"></div>';
  map.addChild(new ymaps3.YMapDefaultFeaturesLayer({zIndex: 1800}))
  marker.value = new ymaps3.YMapMarker({
      coordinates: my_coords.coords,
      draggable: true
  }, content)
  map.addChild(marker.value);

  // bind events
  const clickCallback = async (e: any) => {
    if(e?.entity?.geometry) {
      const raw_location = await ymaps3.search({text: e?.entity?.geometry?.coordinates} as SearchOptions) as SearchResponse
      console.log(raw_location);
      location.value = raw_location ? raw_location[0].properties.name + ' ' + raw_location[1].properties.description : "Can't find location"
      marker.value?.update({ coordinates: e?.entity?.geometry?.coordinates})
    }
  }
  // Создание объекта-слушателя.
  const mapListener = new YMapListener({
    layer: 'any',
    // Добавление обработчиков на слушатель.
    onClick: clickCallback,
  });
  // Добавление слушателя на карту.
  map.addChild(mapListener);
  map_exist.value = true
}

onMounted(async () => {
  console.log('mounted');
  console.log("ready", ymaps3);
  console.log(document.getElementById('map'));
  await create_map();
  console.log(map);
})

//delete create map
const func = async () => {
  console.log(map);
  
  if(map_exist.value) {
    map!.destroy();
    map_exist.value = false;
    console.log(map);
  }
  else {
    await create_map()
  }
}

</script>

<template>
  <div id="map"></div>
  <div>Result: {{ location }}</div>
  <button @click="func">button</button>
</template>

<style>
#map {
  width: 300px;
  height: 300px;
  border: 1px solid black;
  overflow: hidden;
}
</style>
