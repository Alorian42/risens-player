<template>
  <div id="app">
    <loading :active.sync="isLoading" :can-cancel="false" :is-full-page="true"></loading>
    <div class="head">
      <episode-selector class="episodes" :episodes="episodes" :currentEpisodeProp="currentEpisode"></episode-selector>
      <div @click="tryToggle" class="wx-25 download-icon" v-show="canToggle">
        <i v-show="type !== 'sub'" title="Субтитры" class="fas fa-comment"></i>
        <i v-show="type !== 'dub'" title="Озвучка" class="fas fa-microphone-alt"></i>
      </div>
      <div @click="downloadEpisode">
        <i title="Скачать" class="fas fa-cloud-download-alt download-icon"></i>
      </div>
      <div class="ml-auto" @click="alertError">
        <i title="Сообщить об ошибке" class="fas fa-exclamation-triangle alert-icon"></i>
      </div>
    </div>
    <div class="player">
      <iframe :src="episodeUrl" frameborder="0" allowfullscreen width="100%" height="100%"></iframe>
    </div>
  </div>
</template>

<script>
import Vue from "vue";
import VueRouter from "vue-router";
import VueCookies from "vue-cookies";
import Loading from "vue-loading-overlay";
import "vue-loading-overlay/dist/vue-loading.css";
import EpisodeSelector from "./EpisodeSelector";
import Swal from "sweetalert2";

Vue.use(VueRouter);
Vue.use(VueCookies);

var router = new VueRouter({
  mode: "history",
  routes: []
});

export default {
  name: "app",
  router,
  components: {
    EpisodeSelector,
    Loading,
    Swal
  },
  data() {
    return {
      isLoading: false,
      episodes: [],
      forceNightMode: false,
      id: false,
      currentEpisode: 0,
      type: "sub",
      isVip: false
    };
  },
  mounted() {
    fetch("https://risens.team/risensteam/api/get_info.php")
      .then(response => {
        return response.json();
      })
      .then(json => {
        if (json.group == 1 || json.group == 6 || json.group == 10) {
          this.isVip = true;
        }
      });

    this.$on("ChangeEpisode", episode => {
      this.changeEpisode(episode);
    });

    window.addEventListener("message", event => {
      if (event.origin != "https://risens.team") {
        return;
      }
      if (event.data == "night") {
        this.checkNightMode();
      }
    });

    const n = $cookies.get("night");
    this.forceNightMode = n == "yes";

    const query = this.$route.query;
    if (query.id !== undefined) {
      this.id = query.id;
      this.loadEpisodes(this.id);
    }
  },
  computed: {
    canToggle: function() {
      if (this.type === "sub") {
        return this.episodes[this.currentEpisode].dubvideoid !== "";
      } else {
        return this.episodes[this.currentEpisode].subvideoid !== "";
      }
    },
    episodeUrl: function() {
      if (this.episodes[this.currentEpisode] !== undefined) {
        return (
          "https://risens.team/risensteam/player.php?type=" +
          this.type +
          "&id=" +
          this.episodes[this.currentEpisode].id
        );
      }

      return "";
    },
    subsUrl: function() {
      let url = "https://risens.team/risensteam/vipdl.php?type=ass&target=";
      if (this.episodes[this.currentEpisode] !== undefined) {
        const videoId = this.episodes[this.currentEpisode].subvideoid;
        if (videoId === "") {
          return "";
        }

        url += videoId.replace("rtru", "");

        return url;
      } else {
        return "";
      }

      return url;
    },
    torrentUrl: function() {
      let url = "https://risens.team/risensteam/vipdl.php?type=torrent&target=";
      if (this.episodes[this.currentEpisode] !== undefined) {
        const videoId = this.episodes[this.currentEpisode].subvideoid;
        if (videoId === "") {
          return "";
        }

        url += videoId.replace("rtru", "");

        return url;
      } else {
        return "";
      }

      return url;
    }
  },
  methods: {
    changeEpisode: function(episode) {
      this.currentEpisode = episode;

      if (
        this.type === "sub" &&
        this.episodes[this.currentEpisode].subvideoid === ""
      ) {
        this.type = "dub";
      } else if (
        this.type === "dub" &&
        this.episodes[this.currentEpisode].dubvideoid === ""
      ) {
        this.type = "sub";
      }
    },
    checkNightMode: function() {
      const n = $cookies.get("night");
      this.forceNightMode = n == "yes";
    },
    loadEpisodes: function(id) {
      this.isLoading = true;

      fetch("https://risens.team/risensteam/api/anime.php?id=" + id)
        .then(function(response) {
          if (response.ok) {
            return response.json();
          } else {
            setTimeout(() => this.loadEpisodes(this.id), 2000);
          }
        })
        .then(json => {
          this.episodes = json;
          this.episodes.sort((a, b) => {
            if (parseFloat(a.number) > parseFloat(b.number)) {
              return -1;
            } else if (parseFloat(a.number) < parseFloat(b.number)) {
              return 1;
            } else {
              return 0;
            }
          });

          this.changeEpisode(0);
          this.isLoading = false;
          //this.statAction();
        });
    },
    alertError: function() {
      Swal.fire({
        input: "textarea",
        title: "С какой ошибкой вы столкнулись?",
        inputPlaceholder: "Опишите вашу проблему...",
        showCancelButton: true
      }).then(response => {
        const message = response.value;
        if (message) {
          fetch(
            "https://risens.team/risensteam/api/message.php?message=" +
              encodeURIComponent(message) +
              "&episode_id=" +
              this.episodes[this.currentEpisode].id +
              "&anime_id=" +
              this.id
          ).then(function(response) {
            alert("Спасибо!");
          });
        }
      });
    },
    downloadEpisode: function() {
      this.isVip = true;
      if (this.type !== "sub") {
        Swal.fire({
          type: "error",
          title: "Ошибка!",
          text: "Скачка аниме с озвучкой не доступна ;("
        });
        return;
      } else if (!this.isVip) {
        Swal.fire({
          type: "error",
          title: "Ошибка!",
          text: "Скачка доступна только для наших патронов!",
          footer:
            '<a href="https://patreon.risens.team" target="_blank">Поддержите нас на Патреоне...</a>'
        });
        return;
      } else {
        Swal.fire(
          "Скачать",
          'Скачать <a href="' +
            this.subsUrl +
            '" target="_blank">субтитры</a>, скачать <a href="' +
            this.torrentUrl +
            '" target="_blank">торрент</a>',
          "success"
        );
      }
    },
    tryToggle: function() {
      if (!this.canToggle) {
        return;
      }

      if (this.type === "sub") {
        this.type = "dub";
      } else {
        this.type = "sub";
      }
    }
  }
};
</script>

<style lang="scss">
#app,
body {
  padding: 0;
  margin: 0;
  width: 100vw;
  height: 100vh;
}

.head {
  height: 50px;
  width: 100%;

  position: absolute;
  top: 0;
  left: 0;

  z-index: 100;
  background-color: #eb5424;

  display: inline-flex;
  justify-items: start;
  justify-content: start;
  align-items: center;

  opacity: 0.5;
}

.head:hover {
  opacity: 1;
  transition: 600ms;
}

.night .head {
  background-color: #212529;
}

.dropdown {
  background-color: #fff;
  width: 150px;
}

.player {
  position: relative;
  padding-top: 56.25%;
}

.player iframe {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.alert-icon {
  margin-right: 10px;
  color: white;
  font-size: 25px;
  cursor: pointer;
}

.download-icon {
  margin-left: 10px;
  color: white;
  font-size: 25px;
  cursor: pointer;
}

.disabled {
  cursor: not-allowed;
}

.wx-25 {
  width: 25px;
}

.episodes {
  margin-left: 10px;
}

.ml-auto {
  margin-left: auto;
}
</style>
