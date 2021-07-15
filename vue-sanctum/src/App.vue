<template>
  <div id="app">
    <img alt="Vue logo" src="./assets/logo.png" />
  </div>
</template>

<script>
import axios from "axios";
import Pusher from "pusher-js";
import Echo from "laravel-echo";

console.log(Pusher);

export default {
  name: "App",

  data() {
    return {
      data: {
        email: "johndoe@example.org",
        password: "secret",
      },
    };
  },

  async mounted() {
    axios.defaults.withCredentials = true;

    await axios.get("http://localhost:8000/sanctum/csrf-cookie");

    await axios.post("http://localhost:8000/login", this.data);

    const { data } = await axios.get("http://localhost:8000/api/user");

    let echo = new Echo({
      broadcaster: "pusher",
      key: "s3cr3t",
      wsHost: "localhost",
      wsPort: 6001,
      forceTLS: false,
      cluster: "mt1",
      disableStats: true,
      authorizer: (channel, options) => {
        console.log(options);
        return {
          authorize: (socketId, callback) => {
            axios({
              method: "POST",
              url: "http://localhost:8000/api/broadcasting/auth",
              data: {
                socket_id: socketId,
                channel_name: channel.name,
              },
            })
              .then((response) => {
                callback(false, response.data);
              })
              .catch((error) => {
                callback(true, error);
              });
          },
        };
      },
    });

    echo
      .private(`App.Models.User.${data.id}`)
      .listen(".new-message-event", (message) => {
        console.log(message);
      });
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
