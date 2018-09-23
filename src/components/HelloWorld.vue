<template>
  <div class="hello">
    <nav class="navbar navbar-light bg-light">
      <a class="navbar-brand">saveify</a>
      <button class="btn btn-outline-primary my-2 my-sm-0" type="button" v-on:click="login()" v-if="!access_token">login to spotify</button><button class="btn btn-outline-primary my-2 my-sm-0" v-on:click="logout()" v-else>logout</button>
    </nav>

    <div class="container">
      <br>
      <p class="h2" v-if="access_token">hello, {{ userID }}</p>
      <br>
      <p class="h4" v-if="currentSongId">currently playing:</p><p class="h4" v-else>nothing playing right now</p>

      <div v-if="currentSongId">
        <ul>
          <li>
            <a :href="currentArtistNameUri">{{ currentArtistName }}</a> - <a :href="currentSongIdUri">{{ currentSongName }}</a>
          </li>
        </ul>

        <div class="text-center">
          <div class="btn-group" role="group" aria-label="Button group">
            <button class="btn btn-outline-primary btn-lg" v-on:click="previousSong()">previous song</button>
            <button v-if="!savedAlready" class="btn btn-outline-primary btn-lg" v-on:click="saveSong()">save this song</button>
            <button v-else class="btn btn-outline-primary btn-lg" v-on:click="deleteSong()">remove this song</button>
            <button class="btn btn-outline-primary btn-lg" v-on:click="nextSong()">next song</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'HelloWorld',
  data() {
    return {
      client_id: 'dc766e86ba004ababb36d263372a403d',
      redirect_uri: window.location.origin + '/',
      scope: 'user-read-private user-library-read user-read-email user-read-playback-state user-library-modify user-modify-playback-state',
      hashParams: {},
      stateKey: 'spotify_auth_state',
      state: '',
      access_token: '',
      userID: '',
      userProfile: '',
      meLoaded: false,
      currentSongId: '',
      currentSongIdUri: '',
      currentArtistName: '',
      currentArtistNameUri: '',
      currentSongName: '',
      savedAlready: false,
      repeatInterval: 10000
    }
  },
  created: function (){
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
      setTimeout(this.currentSong, this.repeatInterval)
      fetch('https://api.spotify.com/v1/me/player/currently-playing', {
        method: 'get',
        headers: {
          'Authorization': 'Bearer ' + this.access_token
        }
      }).then(res => res.json()).then(res => {
        if(this.currentSongId === res.item.id) {
          return
        }
        this.currentSongId = res.item.id;
        this.currentSongIdUri = res.item.uri;
        this.currentArtistName = res.item.artists[0].name;
        this.currentArtistNameUri = res.item.artists[0].uri;
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
          this.savedAlready = true;
        } else {
          alert('error')
        }
      })
    },
    deleteSong: function() {
      fetch('https://api.spotify.com/v1/me/tracks?ids=' + this.currentSongId, {
        method: 'delete',
        headers: {
          "Content-Type": "application/json; charset=utf-8",
          'Authorization': 'Bearer ' + this.access_token
        }
      }).then(res => {
        if(res.status === 200) {
          this.savedAlready = false;
        } else {
          alert('error')
        }
      })
    },
    previousSong: function() {
      fetch('https://api.spotify.com/v1/me/player/previous', {
        method: 'post',
        headers: {
          'Authorization': 'Bearer ' + this.access_token
        }
      }).then(res => {
        if(res.status === 204) {
          this.currentSong();
        } else {
          alert('error')
        }
      })
    },
    nextSong: function() {
      fetch('https://api.spotify.com/v1/me/player/next', {
        method: 'post',
        headers: {
          'Authorization': 'Bearer ' + this.access_token
        }
      }).then(res => {
        if(res.status === 204) {
          this.currentSong();
        } else {
          alert('error')
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
