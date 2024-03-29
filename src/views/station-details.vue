<template>
  <section v-if="station" class="station-details">
    <h2 class="station-details-header">{{ station.title }}</h2>

    <div class="station-details-container">

      <side-menu-icon class="station-details-menu-icon" @click.native="toggleSidebar"></side-menu-icon>
      
      <station-player
        :station="station"
        :currSong="currSong"
        :isStationOwner="isStationOwner"
        :isPlaylistEmpty="isPlaylistEmpty"
        @order-changed="setPlaylist"
        @song-removed="removeSong"
        @switched-song="setCurrSong"
      ></station-player>

      <aside class="station-details-side-window" :class="{active: isSideMenuActive}">

        <nav>

          <router-link
            class="station-details-side-window-link chat"
            :to="'/station/' + station._id "
          >Chat</router-link>
          
          <router-link
            class="station-details-side-window-link search"
            :to="'/station/' + station._id + '/search'"
          >Add</router-link>

          <router-link
            class="station-details-side-window-link settings"
            v-if="isStationOwner"
            :to="'/station/' + station._id + '/settings'"
          >Edit</router-link>

        </nav>

        <router-view
          @add-song="addSong"
          :routesProps="routesProps"
          class="station-details-side-window-content"
          @station-updated="setStation"
          @station-removed="removeStation"
          @clear-chat="clearChat"
        ></router-view>
        
      </aside>

    </div>

    <!-- ** WORKING ** -->
    <player-controller
      v-if="station"
      class="player-controller"
      :elPlayer="elPlayer"
      :currStation="station"
      :currSong="currSong"
      :playerEv="playerEvNum"
      @playingStatusChanged="updatePlayigStatus"
      @songChanged="setCurrSong"
      @timeElapsed="sendNewTime"
    ></player-controller>
    <!-- ** WORKING ** -->
  </section>
</template>

<script>
import { stationService } from "@/services/station.service";
import { eventBusService } from "@/services/event-bus.service";
import stationPlayer from "@/cmps/station-player.cmp";
import { socketService } from "@/services/socket.service";
import playerController from "@/cmps/player-controller.cmp";
import { storageService } from '@/services/storage.service';
import sideMenuIcon from "@/cmps/icons/side-menu-icon.cmp.vue";

export default {
  data() {
    return {
      station: null,
      isStationOwner: false,
      currSong: null,
      isPlaylistEmpty: true,
      playerEvNum: -1,
      elPlayer: null,
      isSideMenuActive: false
      // playerVars: {
      //   // TODO: make video quality lowest for faster loading time
      // }
    };
  },
  computed: {
    routesProps() {
      switch (this.$route.name) {
        case "station-settings":
          {
            return {
              station: this.station,
              isStationOwner: this.isStationOwner
            };
          }
          break;
          break;
        case "chat-room":
          return {
            station: this.station,
            loggedInUser: this.loggedInUser,
            isStationOwner: this.isStationOwner
          };
          break;
        default:
          break;
      }
    },
    loggedInUser() {
      return this.$store.getters.loggedUser;
    }
    // getLastPlayingTime(){
    //   return this.$store.getters.getLastPlayingTime
    // }
  },
  watch: {
    "station.songs"() {
      this.isPlaylistEmpty = !this.station.songs.length;
      if (this.isPlaylistEmpty) this.currSong = null;
      else {
        if (!this.currSong)
          this.currSong = JSON.parse(JSON.stringify(this.station.songs[0]));
      }
    },
    loggedInUser() {
      this.isStationOwner =
        this.loggedInUser && this.station.owner._id === this.loggedInUser._id;
    }
  },
  methods: {
    // <!-- Controller and Player Functions -->

    updatePlayigStatus(bool) {
      this.$store.commit("setIsPlaying", bool);
    },
    sendNewTime(timeObj) {
      this.$store.commit("setNewTime", timeObj);
    },

    // <!-- Controler and Player Functions -->
    async loadStation(stationId) {
      const station = await this.$store.dispatch({
        type: "loadStation",
        stationId
      });
      this.station = JSON.parse(JSON.stringify(station));

      if (this.station.songs && this.station.songs.length) {
        var distroydeVideoInfo = this.$store.getters.getLastPlayingTime;
        if (distroydeVideoInfo) {
          if (distroydeVideoInfo.inStation === this.station._id) {
            // eventBusService.$emit('STARTING_POINT', distroydeVideoInfo.timeElapsed) need to get starting time
            this.currSong = this.station.songs.find(
              song => song.embedId === distroydeVideoInfo.inSong
            );
          } else {
            this.currSong = {
              embedId: this.station.songs[0].embedId,
              idx: 0,
              title: this.station.songs[0].title
            };
          }
        } else {
          this.currSong = {
            embedId: this.station.songs[0].embedId,
            idx: 0,
            title: this.station.songs[0].title
          };
        }
      } else {
        this.currSong = null;
      }
      
      if (!this.station.owner) {    
        const stationIds = storageService.getStationIds();
        if (stationIds && stationIds.includes(this.station._id)) this.isStationOwner = true;
      } else {
        if (this.loggedInUser && station.owner._id === this.loggedInUser._id)
          this.isStationOwner = true;
      }
    },
    async updateStation() {
      const savedStation = await this.$store.dispatch({
        type: "updateStation",
        station: JSON.parse(JSON.stringify(this.station))
      });
      this.station = JSON.parse(JSON.stringify(savedStation));
    },
    setStation(updatedStation) {
      this.station = updatedStation;
      this.updateStation();
    },
    setPlaylist(val) {
      this.station.songs = JSON.parse(JSON.stringify(val));
      this.updateStation();
    },
    addSong(song) {
      this.station.songs.push(song);
      this.updateStation();
    },
    removeSong(idx) {
      this.station.songs.splice(idx, 1);
      this.updateStation();
    },
    updateRate() {
      this.station.rate++;
      this.updateStation();
    },
    setCurrSong(song) {
      this.currSong = song;
      this.$store.commit("setCurrSong", song);
    },
    removeStation(stationId) {
      this.$store.dispatch({ type: "removeStation", stationId });
    },
    toggleSidebar() {
      this.isSideMenuActive = !this.isSideMenuActive;
    },
    clearChat() {
      this.station.chatHistory = [];
      this.updateStation();
    }
  },
  mounted() {
    eventBusService.$on("SENDING_ELPLAYER", playerElement => {
      this.elPlayer = playerElement;
    });

    eventBusService.$on("PLAYER_EVENT", playerEvent => { 
        this.playerEvNum = playerEvent
      });
    const stationId = this.$route.params.id; 
    if (stationId === 'new') {    
      eventBusService.$emit('station-opened');       
      this.station = stationService.getNewStation();                   
      eventBusService.$on('create-station', async ({ type, title }) => {
        this.station.type = type;
        this.station.title = title;
        const newStation = await this.$store.dispatch({
          type: "addStation",
          station: JSON.parse(JSON.stringify(this.station))
        });
        this.$router.push("/station/" + newStation._id);
        this.isStationOwner = true;
        this.loadStation(newStation._id);
      });
    } else this.loadStation(stationId);
    eventBusService.$on("updateRate", this.updateRate);
  },
  destroyed() {
    eventBusService.$off("create-station");
    eventBusService.$off("SENDING_ELPLAYER");
    eventBusService.$off("PLAYER_EVENT");
    eventBusService.$off("updateRate");
  },
  components: {
    stationPlayer,
    playerController,
    sideMenuIcon
  }
};
</script>
