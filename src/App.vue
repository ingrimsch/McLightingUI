<template>
  <div id="app">

    <v-app v-bind:dark="dark_theme">
      <v-toolbar color="primary" dark>
        <!-- <v-toolbar-side-icon></v-toolbar-side-icon> -->
        <v-toolbar-title>Mc Lighting UI</v-toolbar-title>
        <v-spacer></v-spacer>
        <v-btn-toggle v-model="dark_theme">
          <v-btn class="elevation-6" flat v-bind:value="true" v-if="dark_theme === false"><v-icon>brightness_1</v-icon></v-btn>
          <v-btn class="elevation-6" flat v-bind:value="false" v-else><v-icon>brightness_5</v-icon></v-btn>
        </v-btn-toggle>
        <v-spacer></v-spacer>
        <span class="group pa-2" v-if="is_connected"><v-icon>wifi</v-icon>&nbsp;<span class="hidden-xs-only">&nbsp;McLighting </span>connected</span>
        <span class="group pa-2" v-else><v-icon>signal_wifi_off</v-icon>&nbsp;&nbsp;McLighting disconnected</span>
      </v-toolbar>

      <v-content id="content" class="mx-auto">
        <v-card>
          <v-layout row wrap>
            <v-flex xs12 sm5>
              <ColorPicker v-on:selected="onColorSelected"/>
            </v-flex>
            <v-flex xs12 sm7>
              <v-card-text>
                <v-slider v-model="color_red" hint="Red" label="R" :max="255" v-on:input="set_color" color="red"></v-slider>
                <v-slider v-model="color_green" hint="Green" label="G" :max="255" v-on:input="set_color" color="green"></v-slider>
                <v-slider v-model="color_blue" hint="Blue" label="B" :max="255" v-on:input="set_color" color="blue"></v-slider>
                <v-slider v-model="brightness" prepend-icon="wb_incandescent" hint="Brightness" :max="255" step="12" ticks v-on:input="set_brightness"></v-slider>
                <v-slider v-model="speed" prepend-icon="slow_motion_video" hint="Speed" :max="255" step="12" ticks v-on:input="set_speed"></v-slider>
              </v-card-text>
            </v-flex>
          </v-layout>

          <v-container pb-5>
            <v-btn v-if="edit_mode" block color="green" dark @click="saveSettings()"><v-icon>save</v-icon> Save settings</v-btn>
            <v-layout row wrap>
              <v-flex xs12 sm6 lg4 xl3 v-for="(mode, index) in modes" :key="mode.id" v-if="edit_mode || !mode.hidden">
                <span v-if="edit_mode">
                  <v-icon
                    color="grey lighten-1"
                    @click="toggle(mode, index)"
                    v-if="mode.hidden === true"
                    >star_border</v-icon>
                  <v-icon
                    color="yellow darken-2"
                    @click="toggle(mode, index)"
                    v-else>star</v-icon>
                </span>

                <v-btn :id="mode.id" v-on:click="set_mode(mode.id)" class="elevation-6 white--text" v-bind:class="[modeIsActive(mode) ? 'red' : 'blue']">{{ mode.title }} ({{ mode.id }})</v-btn>
              </v-flex>
            </v-layout>
          </v-container>

          <v-speed-dial
            v-model="fab"
            transition="scale-transition"
            bottom
            right
            open-on-hover
            fixed>
            <v-btn
              slot="activator"
              color="pink"
              dark
              fab
              hover
              v-model="fab">
              <v-icon>settings</v-icon>
              <v-icon>close</v-icon>
            </v-btn>
            <v-btn
              fab
              dark
              small
              color="green"
              v-on:click="edit_mode = !edit_mode">
              <v-icon>edit</v-icon>
            </v-btn>
          </v-speed-dial>
        </v-card>

        <v-footer color="primary white--text" class="pa-3">
          <div><a href="https://github.com/toblum/McLighting/" target="_blank" class="white--text">Project home</a> - &copy; {{ new Date().getFullYear() }}</div>
          <v-spacer></v-spacer>
        </v-footer>
      </v-content>
    </v-app>

  </div>
</template>

<script>
/* eslint-disable */
import ColorPicker from "./components/ColorPicker";

var host = "192.168.0.49";

export default {
  name: "McLightingUI",
  
  components: {
    ColorPicker
  },

  data: () => ({
    color_red: 0,
    color_green: 0,
    color_blue: 0,
    brightness: 192,
    speed: 192,
    modes: [{ title: "TV", id: "tv" }],
    connection: null,
    is_connected: false,
    dark_theme: false,
    ws2812fx_mode: null,
    settings: {},
    edit_mode: false,
    fab: false
  }),

  methods: {
    getModes() {
      var that = this;
      this.$http.get("//" + host + "/get_modes").then(data => {
        // console.log("Getting modes list via REST:", data);
        data.body.forEach(item => {
          if (item.name && item.name.length > 0) {
            that.modes.push({ title: item.name, id: item.mode });
          }
        });
        this.readSettings();
      });
    },
    modeIsActive(mode) {
      return mode.id === this.ws2812fx_mode;
    },

    readSettings() {
      this.$http.get("//" + host + "/uistate.json").then(data => {
        console.log("readSettings()", data.body);
        this.settings = data.body || {};
        this.applySettings();
      });
    },
    applySettings() {
      this.modes.forEach((mode, index) => {
        if (this.settings.visibility.includes(mode.id)) {
          mode.hidden = true;
          this.$set(this.modes, index, mode);
        }
      });
    },
    saveSettings() {
      var visibility = this.modes.map(mode => {
        if (mode.hidden) {
          return mode.id;
        }
      }).filter((x) => {return x >= 0});
      this.settings.visibility = visibility;

      var formData = new FormData();
      var blob = new Blob([JSON.stringify(this.settings)], {type: 'application/json'});
      formData.append('settings', blob, '/uistate.json');

      this.$http.post("//" + host + "/edit", formData).then(data => {
        console.log("SUCCESS saveSettings()", data, this.settings);
        this.edit_mode = false;
      }, err => {
        console.error("ERROR saveSettings()", err);
      });
    },

    toggle(mode, index) {
      mode.hidden = !mode.hidden;
      this.$set(this.modes, index, mode);
    },

    ws_connect() {
      var that = this;
      this.connection = new ReconnectingWebSocket("ws://" + host + ":81");
      this.connection.debug = true;
      // connection.timeoutInterval = 5400;

      // When the connection is open, send some data to the server
      this.connection.onopen = function() {
        console.log("WebSocket Open");
        that.is_connected = true;
        that.ws_send("$");
      };

      // When the connection is open, send some data to the server
      this.connection.onclose = function() {
        console.log("WebSocket Closed");
        that.is_connected = false;
      };

      // Log errors
      this.connection.onerror = function(error) {
        that.is_connected = false;
        console.error("WebSocket Error", error);
      };

      // Log messages from the server
      this.connection.onmessage = function(e) {
        console.log("WebSocket from server:", e.data);
        try {
          var res = JSON.parse(e.data);
          // console.log("res", res);
          if (res && res.ws2812fx_mode) {
            that.ws2812fx_mode = res.ws2812fx_mode;
          }
          if (res && res.speed) {
            that.speed = res.speed;
          }
          if (res && res.brightness) {
            that.brightness = res.brightness;
          }
          if (res && res.color) {
            that.color_red = res.color[0];
            that.color_green = res.color[1];
            that.color_blue = res.color[2];
          }
        } catch (e) {}
      };
    },

    ws_send(message) {
      console.log("WS send: ", message);
      this.connection.send(message);
    },

    set_mode(mode_id) {
      this.ws2812fx_mode = mode_id;
      this.ws_send("/" + mode_id);
    },
    set_speed(speed) {
      this.ws_send("?" + speed);
    },
    set_brightness(brightness) {
      this.ws_send("%" + brightness);
    },
    set_color() {
      this.ws_send(
        "#" + this.rgbToHex([this.color_red, this.color_green, this.color_blue])
      );
    },

    componentToHex(c) {
      return ("0" + Number(c).toString(16)).slice(-2).toUpperCase();
    },
    rgbToHex(rgb) {
      return (
        this.componentToHex(rgb[0]) +
        this.componentToHex(rgb[1]) +
        this.componentToHex(rgb[2])
      );
    },

    onColorSelected(color) {
      this.color_red = color.r;
      this.color_green = color.g;
      this.color_blue = color.b;
      this.set_color();
    }
  },

  mounted() {
    this.getModes();
    this.ws_connect();
  }
};
</script>

<style>
#content {
  max-width: 1380px;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
/*
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
*/
</style>