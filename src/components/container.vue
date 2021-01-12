<template>
  <div
    ref="map-root"
    @mousemove="mouseCoord"
    style="width: 100%; height: 100%"
  ></div>
  <select v-model="mapUnits">
    <option value="" selected disabled>map unit</option>
    <option v-for="option in options" :key="option.text" :value="option.value">
      <!-- v-bind:value 回傳給mapUnits-->
      {{ option.text }}
    </option>
  </select>
  <div>Longitude:{{ coord[0] }}, Latitude: {{ coord[1] }}</div>
  <div v-for="(item, i) in checklist" :key="item">
    <input
      type="checkbox"
      @click="checkSwitch(i)"
      v-model="checklist[i].state"
    />
    <label> {{ item.name }} </label>
    <br />
    Opacity
    <input v-model="sliderOpacity[i]" style="width: 25px" />
    %
    <vue-slider
      @change="sliderChange(i)"
      v-model="sliderOpacity[i]"
      :min="0"
      :max="100"
      :marks="[0, 50, 100]"
      :hide-label="true"
      :tooltip-formatter="sliderFormat"
      width="200px"
    ></vue-slider>
  </div>
</template>

<script>
import "ol/ol.css";
import View from "ol/View";
import Map from "ol/Map";
import { defaults as defaultControl, ScaleLine } from "ol/control";
import { defaults as defaultInteractions, DragAndDrop } from "ol/interaction";
import { GPX, GeoJSON, IGC, KML, TopoJSON } from "ol/format";
import { Vector as VectorLayer } from "ol/layer";
import { Vector as VectorSource } from "ol/source";
import { Style, Circle, Fill, Stroke } from "ol/style";
import Point from "ol/geom/Point";
import Feature from "ol/Feature";
import VueSlider from "vue-slider-component";
import "vue-slider-component/theme/antd.css";
import bus from "./eventBus";

const scaleControl = (unit) => {
  const control = new ScaleLine({
    units: unit,
  });
  return control;
};

const addPoints = (gpxPointLayer, linestring, filename) => {
  linestring.getCoordinates().forEach((coordinates) => {
    gpxPointLayer.getSource().addFeature(
      new Feature({
        geometry: new Point(coordinates),
        name: filename,
      })
    );
  });
};

const dragAndDropInteraction = new DragAndDrop({
  formatConstructors: [GPX, GeoJSON, IGC, KML, TopoJSON],
});

const dragFeature = (mainmap) => {
  dragAndDropInteraction.on("addfeatures", function (event) {
    const file_name = event.file.name;
    const pts_name = "pts_" + file_name;
    bus.emit("filename", file_name); //傳送給平行元件
    //增加空點圖層
    const gpxPointLayer = new VectorLayer({
      name: pts_name,
      source: new VectorSource(),
      style: new Style({
        image: new Circle({
          fill: new Fill({ color: "red" }),
          radius: 3,
        }),
      }),
    });
    let coords = [];
    event.features.forEach((item) => {
      item.set("filename", file_name);
      if (item.getGeometry().getType() === "LineString") {
        addPoints(gpxPointLayer, item.getGeometry(), file_name);
      } else if (item.getGeometry().getType() === "MultiLineString") {
        item
          .getGeometry()
          .getLineStrings()
          .forEach(function (linestring) {
            coords = coords.concat(linestring.getCoordinates());
            addPoints(gpxPointLayer, linestring, file_name);
          });
      }
    });
    const vectorSource = new VectorSource({
      features: event.features,
    });
    const linelayer = new VectorLayer({
      name: file_name,
      source: vectorSource,
      style: new Style({
        stroke: new Stroke({ color: "green", width: 4 }),
      }),
    });
    mainmap.addLayer(linelayer);
    mainmap.addLayer(gpxPointLayer);
    mainmap.getView().fit(vectorSource.getExtent());
  });
};

const highlightStyle = new Style({
  stroke: new Stroke({
    color: [255, 0, 0, 0.5],
    width: 6,
  }),
  fill: new Fill({
    color: "rgba(255,0,0,0.1)",
  }),
});

export default {
  name: "MapContainer",

  components: {
    VueSlider,
  },
  props: ["layerP"],
  data() {
    //component必須用function
    return {
      checklist: [],
      sliderOpacity: [],
      layers: this.layerP,
      options: [
        { text: "metric", value: "metric" },
        { text: "degrees", value: "degrees" },
        { text: "nautical miles", value: "nautical" },
      ],
      mainmap: null,
      mapUnits: "",
      sliderFormat: "{value}%",
      coord: "",
    };
  },

  mounted() {
    this.layers.forEach((item) => {
      this.checklist.push({ state: true, name: item.get("name") });
      this.sliderOpacity.push(item.getOpacity() * 100);
    });
    //mounted後DOM載入始可用ol, data變動會update重新渲染
    this.initmap();
  },
  watch: {
    mapUnits: function (newVal) {
      //監聽mapUnits,  function(newVal, oldVal)
      this.mainmap.removeControl(this.mainmap.getControls().getArray()[3]);
      this.mainmap.addControl(scaleControl(newVal));
    },
  },

  methods: {
    sliderChange: function (i) {
      this.mainmap
        .getLayers()
        .getArray()
        [i].setOpacity(this.sliderOpacity[i] / 100);
    },
    checkSwitch: function (i) {
      const bool = this.mainmap.getLayers().getArray()[i].getVisible();
      this.checklist[i].state = !bool;
      this.mainmap.getLayers().getArray()[i].setVisible(!bool);
    },
    mouseCoord: function (ev) {
      const pixel = this.mainmap.getEventPixel(ev);
      const _coord = this.mainmap.getCoordinateFromPixel(pixel);
      const coord4 = (_coord || []).map(function (a) {
        //防止空值報錯
        return a.toFixed(4);
      }); //methods內不用箭頭函數
      this.coord = coord4;

      let selected = null;

      this.mainmap.forEachFeatureAtPixel(pixel, function (feature) {
        selected = feature;
        selected.setStyle(highlightStyle);
        console.log(selected);
      });
    },
    initmap: function () {
      this.mainmap = new Map({
        //連結到data的mainmap
        controls: defaultControl().extend([scaleControl()]),
        interactions: defaultInteractions().extend([dragAndDropInteraction]),
        target: this.$refs["map-root"],
        layers: this.layers,
        view: new View({
          projection: "EPSG:4326",
          center: [121, 23.6],
          zoom: 7.5,
        }),
      });
      dragFeature(this.mainmap);
    },
  },
};
</script>