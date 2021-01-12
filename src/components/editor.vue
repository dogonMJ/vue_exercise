<template>
  <div>
    <vue-slider v-model="value" />
    <div>{{ filename }}</div>
  </div>
</template>
 
<script>
import VueSlider from "vue-slider-component";
import "vue-slider-component/theme/antd.css";
import bus from "./eventBus";
export default {
  components: {
    VueSlider,
  },
  props: ["layerP"],
  data() {
    return {
      value: 50,
      filename: "",
      layers: this.layerP,
    };
  },
  created() {
    bus.on("filename", (e) => {
      this.filename = e;
    });
  },
};
</script>