<template>
  <div class="hello">
    <div class="container">
      <h1>Saveify</h1>
      <h3 v-if="access_token">hello <a v-bind:href="userProfile">{{ userID }}</a></h3>
      <h3 v-if="currentSongId">currently playing</h3><h3 v-else>nothing playing right now</h3>
      <div v-if="currentSongId">
        <ul>
          <li>
            {{ currentArtistName }} - {{ currentSongName }}
          </li>
        </ul>
        <div v-if="!savedAlready && !newlySaved">
          <button v-on:click="saveSong()">save this song</button>
        </div>
        <div v-if="saveMessage">
          <span>{{ saveMessage }}</span>
        </div>
        <div v-if="savedAlready">
          <span>already saved</span>
        </div>
      </div>
      <button v-on:click="login()" v-if="!access_token">login to spotify</button>
      <button v-on:click="logout()" v-else>logout</button>
    </div>
  </div>
</template>

<script>
export default {
  name: 'HelloWorld',
  data() {
    return {
      client_id: 'dc766e86ba004ababb36d263372a403d',
      redirect_uri: 'http://localhost:8080/',
      scope: 'user-read-private user-library-read user-read-email user-read-playback-state user-library-modify',
      hashParams: {},
      stateKey: 'spotify_auth_state',
      state: '',
      access_token: '',
      userID: '',
      userProfile: '',
      meLoaded: false,
      currentSongId: '',
      currentArtistName: '',
      currentSongName: '',
      savedAlready: false,
      newlySaved: false,
      saveMessage: ''
    }
  },
  created: function (){
    // var storedState = localStorage.getItem(this.stateKey)
    var e, r = /([^&;=]+)=?([^&;]*)/g,
        q = window.location.hash.substring(1);
    // eslint-disable-next-line
    while ( e = r.exec(q)) {
        this.hashParams[e[1]] = decodeURIComponent(e[2]);
    }

    this.access_token = this.hashParams.access_token;
    this.state = this.hashParams.state;
    return this.hashParams;
  },
  watch: {
    access_token: function () {
      if(this.access_token) {
        return this.about();
      }
    },
    meLoaded: function () {
      return this.currentSong();
    }
  },
  methods: {
    about: function() {
      fetch('https://api.spotify.com/v1/me', {
        method: 'get',
        headers: {
          'Authorization': 'Bearer ' + this.access_token
        }
      }).then(res => res.json()).then(res => {
        this.userID = res.id;
        this.userProfile = res.external_urls.spotify;
        this.meLoaded = true;
      })
    },
    currentSong: function() {
      fetch('https://api.spotify.com/v1/me/player/currently-playing', {
        method: 'get',
        headers: {
          'Authorization': 'Bearer ' + this.access_token
        }
      }).then(res => res.json()).then(res => {
        this.currentSongId = res.item.id;
        this.currentArtistName = res.item.artists[0].name
        this.currentSongName = res.item.name;
        fetch('https://api.spotify.com/v1/me/tracks/contains?ids=' + this.currentSongId, {
          method: 'get',
          headers: {
            'Authorization': 'Bearer ' + this.access_token
          }
        }).then(res => res.json()).then(res => this.savedAlready = res[0])
      })
    },
    saveSong: function() {
      fetch('https://api.spotify.com/v1/me/tracks', {
        method: 'put',
        headers: {
          "Content-Type": "application/json; charset=utf-8",
          'Authorization': 'Bearer ' + this.access_token
        },
        body: JSON.stringify({
          "ids": [this.currentSongId]
        })
      }).then(res => {
        if(res.status === 200) {
          this.newlySaved = true;
          this.saveMessage = 'saved'
        } else {
          this.saveMessage = 'error'
        }
      })
    },
    generateRandomString: function (length) {
      var text = '';
      var possible = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';

      for (var i = 0; i < length; i++) {
        text += possible.charAt(Math.floor(Math.random() * possible.length));
      }
      return text;
    },
    login: function() {
      var state = this.generateRandomString(16);
      localStorage.setItem(this.stateKey, state);

      var url = 'https://accounts.spotify.com/authorize';
      url += '?response_type=token';
      url += '&client_id=' + encodeURIComponent(this.client_id);
      url += '&scope=' + encodeURIComponent(this.scope);
      url += '&redirect_uri=' + encodeURIComponent(this.redirect_uri);
      url += '&state=' + encodeURIComponent(state);
      return window.location = url;
    },
    logout: function() {
      localStorage.removeItem(this.stateKey);
      return window.location = '/';
    }
  }
}
</script>
