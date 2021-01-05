<template>
  <div ref="map-root" style="width: 100%; height: 100%"></div>
  <select v-model="selectUnits">
    <option value="" selected disabled>map unit</option>
    <option v-for="option in options" :key="option.text" :value="option.value">
      <!-- v-bind:value 回傳給selectUnits-->
      {{ option.text }}
    </option>
  </select>

  <div v-for="(item, i) in checklist" :key="i">
    <input type="checkbox" :value="i" v-model="checklist[i].state" id="i" />
    <label for="i"> {{ checklist[i].name }} </label>
    <br />
    Opacity <input v-model="sliderVal[i]" style="width: 25px" /> %
    <vue-slider
      v-model="sliderVal[i]"
      :min="0"
      :max="100"
      :marks="[0, 50, 100]"
      :hide-label="true"
      :tooltip-formatter="sliderFormat"
      width="200px"
    >
    </vue-slider>
  </div>
</template>

<script>
import "ol/ol.css";
import View from "ol/View";
import Map from "ol/Map";
import TileLayer from "ol/layer/Tile";
import OSM from "ol/source/OSM";
import XYZ from "ol/source/XYZ";
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

const layers = [
  new TileLayer({
    name: "OpenStreetMap",
    source: new OSM(),
  }),
  new TileLayer({
    name: "魯地圖",
    opacity: 0.5,
    source: new XYZ({
      url: "http://rudy.tile.basecamp.tw/{z}/{x}/{y}.png",
    }),
  }),
];

function scaleControl(unit) {
  let control = new ScaleLine({
    units: unit,
  });
  return control;
}
function addPoints(gpxPointLayer, linestring, filename) {
  linestring.getCoordinates().forEach((coordinates) => {
    gpxPointLayer.getSource().addFeature(
      new Feature({
        geometry: new Point(coordinates),
        name: filename,
      })
    );
  });
}

let checklist = [];
layers.forEach((item) =>
  checklist.push({ state: true, name: item.get("name") })
);
let sliderval = [];
layers.forEach((item) => sliderval.push(item.getOpacity() * 100));

let dragAndDropInteraction = new DragAndDrop({
  formatConstructors: [GPX, GeoJSON, IGC, KML, TopoJSON],
});

export default {
  name: "MapContainer",

  components: {
    VueSlider,
  },

  data() {
    //component必須用function
    return {
      checklist,
      options: [
        { text: "metric", value: "metric" },
        { text: "degrees", value: "degrees" },
        { text: "nautical miles", value: "nautical" },
      ],
      map: null,
      selectUnits: "",
      sliderVal: sliderval,
      sliderFormat: "{value}%",
    };
  },

  mounted() {
    //mounted後DOM載入始可用ol, data變動會update重新渲染
    this.initmap();
  },

  watch: {
    sliderVal: {
      handler: function (newVal) {
        for (let i = 0; i < newVal.length; i++) {
          this.map.getLayers().array_[i].setOpacity(newVal[i] / 100);
        }
      },
      deep: true,
    },
    checklist: {
      handler: function (newVal) {
        for (let i = 0; i < newVal.length; i++) {
          //圖層開關
          this.map.getLayers().array_[i].setVisible(newVal[i].state);
        }
      },
      deep: true,
    },
    selectUnits: function (newVal) {
      //監聽selectUnits,  function(newVal, oldVal)
      this.map.removeControl(this.map.getControls().array_[3]);
      this.map.addControl(scaleControl(newVal));
    },
  },

  methods: {
    initmap: function () {
      this.map = new Map({
        controls: defaultControl().extend([scaleControl()]),
        interactions: defaultInteractions().extend([dragAndDropInteraction]),
        target: this.$refs["map-root"],
        layers: layers,
        view: new View({
          projection: "EPSG:4326",
          center: [121, 23.6],
          zoom: 7.5,
        }),
      });

      let that = this;

      dragAndDropInteraction.on("addfeatures", function (event) {
        let file_name = event.file.name;
        let pts_name = "pts_" + file_name;
        that.$bus.emit("theevent", file_name); //傳送給平行元件
        //增加空點圖層
        let gpxPointLayer = new VectorLayer({
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
        event.features.forEach(function (item) {
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
        let vectorSource = new VectorSource({
          features: event.features,
        });
        let linelayer = new VectorLayer({
          name: file_name,
          source: vectorSource,
          style: new Style({
            stroke: new Stroke({ color: "green", width: 4 }),
          }),
        });
        that.map.addLayer(linelayer);
        that.map.addLayer(gpxPointLayer);
        that.map.getView().fit(vectorSource.getExtent());
      });
    },
  },
};
</script>