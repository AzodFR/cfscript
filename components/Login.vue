<template>
  <div class="login-form">
    <select v-model="selected" class="api-selector">
      <option v-for="opt in options" :key="opt">
        {{ opt }}
      </option>
    </select>
    <b-button variant="success" @click="login">Login on WAX</b-button>
    <b-spinner label="Spinning" v-if="wait" class="spinner-login"></b-spinner>
  </div>
</template>

<script>
export default {
  name: "Login",
  computed: {
    user() {
      return this.$store.state.user;
    },
  },
  data() {
    return {
      wait: false,
      options: [
        "https://wax.cryptolions.io",
        "https://query.3dkrender.com",
        "https://api.wax.alohaeos.com",
        "https://wax.eu.eosamsterdam.net",
        "https://wax.blokcrafters.io",
        "https://api-wax-mainnet.wecan.dev",
        "https://wax.dapplica.io",
        "https://wax.eosdac.io",
        "https://wax.eoseoul.io",
        "https://api.wax.liquidstudios.io",
        "https://wax.api.eosnation.io",
        "https://api.waxsweden.org"
      ],
      selected: "https://wax.cryptolions.io",
    };
  },
  methods: {
    login: async function () {
      this.wait = true;
      localStorage.setItem("rpc", this.selected)
      await this.$store.commit("user/login", this.selected);
      const wax = this.$store.state.user.wax;
      //console.log(wax);
      const waitLoging = setInterval(() => {
        if (wax != null && wax.userAccount != undefined) {
          clearInterval(waitLoging);
          this.$store.commit("user/setUser");
          this.$emit("logged");
        }
      }, 1000);
    },
  },
  mounted() {
      if (localStorage.getItem("rpc")) {
      if (localStorage.getItem("rpc") == "random" &&  (!localStorage.getItem("blockedRPC") || localStorage.getItem("blockedRPC") == "false")) {
        const rn = parseInt(Math.random() * this.options.length)
        this.selected = this.options[rn]
      }
      else {
       this.selected = localStorage.getItem("rpc");
      }
     }
    if (localStorage.getItem("autoLogin") && localStorage.getItem("autoLogin") == "true")
    {
      this.login();
    }
    if (localStorage.getItem("autoLogin") && localStorage.getItem("autoLogin") == "rpc")
    {
      localStorage.setItem("autoLogin", "false")
      this.login();
    }
  },
};
</script>

<style>
.login-form {
  display: flex;
  justify-content: center;
  margin-top: 2%;
}
.api-selector {
  margin-right: 1%;
}
.spinner-login {
  margin-left: 1%;
  color: grey;
}
</style>
