<template>
  <div>
    <vue-slider v-model="value" />
    <div>{{ filename }}</div>
  </div>
</template>
 
<script>
import VueSlider from "vue-slider-component";
import "vue-slider-component/theme/antd.css";

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
    console.log(this.layers);
    this.$bus.on("filename", (e) => {
      this.filename = e;
    });
  },
};
</script>