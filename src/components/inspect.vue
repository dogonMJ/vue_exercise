<template>
  <div>
    <vue-slider v-model="value" />
    <div>{{ busval }}</div>
  </div>
</template>
 
<script>
import VueSlider from "vue-slider-component";
import "vue-slider-component/theme/antd.css";

export default {
  created() {
    this.$bus.on("theevent", (e) => {
      this.busval = e;
    });
  },
  components: {
    VueSlider,
  },
  data() {
    return {
      value: 50,
      busval: "",
    };
  },
};
</script>