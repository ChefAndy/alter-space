<template>
  <div class="row light-levers">
    <div class="title-shape">
      <svgicon icon="triangle-light"
               width="100%"
               height="100%"
               title="Light Levers"
               class="triangle"
               :class="$route.params.name"
               stroke="0">
      </svgicon>

    </div>
    <div class="table cell-table table-top-single"
         :class="[$route.params.name, {
            options: showingList
         }]">
      <div class="tr">
        <div class="td col-4">
          <div class="btn-group-color"
               role="group"
               aria-label="color button group">

            <button type="button"
                    class="btn-round btn-color"
                    v-for="(color, idx) in colorPresets"
                    v-bind:key="idx"
                    @click="showList(idx)"
                    :class="{active: color === colorPresets[currentColorIdx] && showingList && currentColorIdx === idx}"
                    :style="{'backgroundColor': color}">
            </button>
            <label class="text-center">colors</label>
          </div>
        </div>
        <breathe-button
                v-show="$route.params.name !== 'wyrd'">
        </breathe-button>
        <chase-button
                v-show="$route.params.name === 'wyrd'">
        </chase-button>
        <div class="td col-6">
          <brightness-slider></brightness-slider>
          <label>brightness</label>
        </div>
      </div>
    </div>
    <table class="table cell-table table-bottom"
           :class="$route.params.name"
           v-show="showingList">
      <div class="btn-group-color-options col-centered"
           :class="$route.params.name"
           role="group"
           aria-label="color button group">
        <svgicon icon="arrow-up"
                 class="arrow-up colors"
                 :class="['color-'+currentColorIdx, $route.params.name]">
        </svgicon>
        <button type="button"
                class="btn-round-small btn-color-option"
                v-for="(hexVal, idx) in colors"
                v-bind:key="idx"
                @click="chooseNewColor(hexVal)"
                :style="{'backgroundColor': hexVal}">
        </button>
      </div>
    </table>
  </div>
</template>

<script>
  import axios from 'axios';

  import './icons/breathe';
  import './icons/triangle-light';
  import './icons/arrow-up';

  import EventBus from '../event-bus';

  import BrightnessSlider from './BrightnessSlider';
  import ChaseButton from "./ChaseButton";
  import BreatheButton from "./BreatheButton";

  const colorsUrl = process.env.VUE_APP_BACKEND_URL + "lights" + "/colors";
  const lightUrl = process.env.VUE_APP_BACKEND_URL + "lights" + "/set";

  // steps between color 1 and color 2
  const steps = 8;

  export default {
    name: "LightLevers",
    components: {
      BreatheButton,
      ChaseButton,
      BrightnessSlider
    },
    props: ["lightPresets", "collapseLightOptions"],
    data() {
      return {
        showColorPicker: false,
        colorPresets: [],
        colors: [],
        showingList: false,
        currentColorIdx: "",
        light: "",
        colorGradient: "",
        brightness: 100,
      }
    },
    watch: {
      lightPresets() {
        this.colorPresets = this.lightPresets.colors;
        this.createGradient();
      },
      collapseLightOptions() {
        if (this.collapseLightOptions) {
          this.showingList = false;
          this.$parent.showingLightOptions = false;
        }
      }
    },
    methods: {
      showList(idx) {
        if (!(this.showingList)) {
          this.showingList = true;
        } else if (this.currentColorIdx === idx || this.currentColorIdx === "") {
          this.showingList = !this.showingList;
        }
        this.currentColorIdx = idx;
        this.$parent.showingLightOptions = this.showingList;
      },

      createGradient() {
        let color0 = this.hex2rgb(this.colorPresets[0]);
        let color1 = this.hex2rgb(this.colorPresets[1]);
        let color2 = this.hex2rgb(this.colorPresets[2]);

        let colorGradient1 = this.interpolateColors(color0, color1, steps);
        let colorGradient2 = this.interpolateColors(color1, color2, steps);
        this.colorGradient = colorGradient1.concat(colorGradient2);
        this.createAndSendStates();
      },
      createAndSendStates() {
        let bodyFormData = new FormData();
        let d = {color_data: this.colorGradient};
        bodyFormData.set('colors', JSON.stringify(d));
        bodyFormData.set('id', this.light);
        bodyFormData.set('bright', this.brightness.toString());
        axios({
          method: "post",
          url: lightUrl,
          data: bodyFormData,
        }).then(function (results) {
          //TODO: disable everything until results are back
          console.log(results)
        })
      },
      chooseNewColor(hexVal) {
        this.colorPresets.splice([this.currentColorIdx], 1, hexVal);
        this.createGradient();
      },
      setLight() {
        this.light = localStorage.getItem('light');
      },

      // From https://codepen.io/njmcode/pen/axoyD?editors=0010
      rgb2hex(rgb) {
        return "#" + ((1 << 24) + (rgb[0] << 16) + (rgb[1] << 8) + rgb[2]).toString(16).slice(1);
      },

      hex2rgb(hex) {
        let result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
        return result ? [
          parseInt(result[1], 16),
          parseInt(result[2], 16),
          parseInt(result[3], 16)
        ] : null;
      },
      // From https://graphicdesign.stackexchange.com/a/83867
      interpolateColor(color1, color2, factor) {
        if (arguments.length < 3) {
          factor = 0.5;
        }
        let result = color1.slice();
        for (let i = 0; i < 3; i++) {
          result[i] = Math.round(result[i] + factor * (color2[i] - color1[i]));
        }
        return this.rgb2hex(result);
      },

      interpolateColors(color1, color2, steps) {
        let stepFactor = 1 / (steps - 1),
            interpolatedColorArray = [];

        for (let i = 0; i < steps; i++) {
          let interpolated = this.interpolateColor(color1, color2, stepFactor * i);
          interpolatedColorArray.push(interpolated);
        }
        return interpolatedColorArray;
      },

    },
    beforeMount() {
      /* get available colors */
      axios.get(colorsUrl).then((res) => {
        this.colors = res.data;
      });
    },
    mounted() {
      let self = this;
      EventBus.$on('update-brightness', (brightness) => {
        self.brightness = brightness;
      });
    },
    created() {
      this.setLight();
    },
  }

</script>