<template>
  <div id="app">
    <div class="cell cell-map">
      <MapContainer :layerP="layerP"></MapContainer>
    </div>
    <div class="cell cell-edit">
      Edit<br />
      <Editor :layerP="layerP"></Editor>
    </div>
    <div class="cell cell-inspect">
      Inspect<br />
      <Inspect></Inspect>
    </div>
  </div>
</template>

<script>
import MapContainer from "./components/container.vue";
import Inspect from "./components/inspect.vue";
import Editor from "./components/editor.vue";
import TileLayer from "ol/layer/Tile";
import OSM from "ol/source/OSM";
import XYZ from "ol/source/XYZ";
import { BingMaps } from "ol/source";
export default {
  name: "App",
  components: {
    MapContainer,
    Inspect,
    Editor,
  },
  data() {
    return {
      layerP: [
        new TileLayer({
          name: "OpenStreetMap",
          source: new OSM(),
          zIndex: 0,
        }),
        new TileLayer({
          name: "魯地圖",
          opacity: 0.5,
          source: new XYZ({
            url: "https://rs.happyman.idv.tw/map/moi_osm/{z}/{x}/{y}.png",
          }),
          zIndex: -1,
        }),
        new TileLayer({
          name: "BingMap",
          source: new BingMaps({
            imagerySet: "Aerial",
            key:
              "AtxhFL61gkrGg34Rd6hUnrZbAYu3s_fpbocD79mi7w3YEWzY0SoK2wD0HJJlgg4I",
          }),
          zIndex: -2,
        }),
      ],
    };
  },
};
</script>

<style>
html,
body {
  height: 100%;
  margin: 0;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  height: 100%;
  display: grid;
  grid-template-columns: 100vh;
  grid-auto-rows: 1fr;
  grid-gap: 1rem;
  padding: 1rem;
  box-sizing: border-box;
}

.cell {
  border-radius: 4px;
  background-color: lightgrey;
}

.cell-map {
  grid-column: 1;
  grid-row-start: 1;
  grid-row-end: 3;
}

.cell-edit {
  grid-column: 2;
  grid-row: 1;
}

.cell-inspect {
  grid-column: 2;
  grid-row: 2;
}
</style>