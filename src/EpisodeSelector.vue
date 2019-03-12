<template>
  <v-select
    maxHeight="250px"
    :onChange="episodeSelected"
    v-model="currentEpisode"
    :options="options"
  ></v-select>
</template>
<script>
import vSelect from "vue-select";

export default {
  props: {
    episodes: {
      type: Array,
      default: []
    },
    currentEpisodeProp: {
      type: Number,
      default: 0
    }
  },

  data() {
    return {
      tempValue: 0
    };
  },
  computed: {
    currentEpisode: {
      get: function() {
        if (this.episodes[this.currentEpisodeProp] !== undefined) {
          return {
            value: this.currentEpisodeProp,
            label: this.episodes[this.currentEpisodeProp].name
          };
        } else {
          return 0;
        }
      },
      set: function(newValue) {
        this.tempValue = newValue;
      }
    },
    options: function() {
      let array = [];

      this.episodes.forEach((elem, index) => {
        array.push({
          value: index,
          label: elem.name
        });
      });

      return array;
    }
  },
  methods: {
    episodeSelected: function(val) {
      if (val === null || val === undefined) {
        val = {
          value: this.episodes.length - 1
        };
      }

      if (val.value != this.currentEpisodeProp) {
        this.$parent.$emit("ChangeEpisode", val.value);
      }
    }
  },
  updated() {},
  mounted() {},
  components: {
    vSelect
  }
};
</script>
<style lang="scss">
select {
  padding: 10px;
  margin: 5px;
}

.bottom .v-select {
  width: 150px;
  background-color: white;
}
.bottom .dropdown-menu {
  bottom: 100%;
  top: auto !important;
}
.clear {
  display: none;
}

.dropdown {
  border-radius: 5px;
}

.dropdown-toggle {
  max-height: 35px;
  text-overflow: clip;
  overflow: hidden;
  white-space: nowrap !important;
}
</style>
