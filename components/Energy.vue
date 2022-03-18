<template>
   <div>
   <b-button size="sm" v-b-tooltip.hover title="Activate this to refill your energy when you are lower than 25% of your maximum." :variant="refill ? 'success': 'danger'" @click="switchRef">Auto Energy: {{refill ? "ON" : "OFF"}}</b-button>
  </div>
</template>
<script>
export default {
  data() {
    return {
      refill: false,
      refresh: null,
      pushed: false,
      user: this.$store.state.user
    }
  },
  mounted() {
    if (localStorage.getItem("energy") && localStorage.getItem("energy") == "true")
    {
      this.refill = true;
      this.launch();
    }
  },
  methods: {
    launch() {
      this.refresh = setInterval(() => {
        this.user = this.$store.state.user

        if (this.user.energy <= this.user.max_energy / 4 && !this.pushed)
        {
          if (this.user.ressources["PWR"] >= 1)
          {

            const needed = parseInt((this.user.max_energy - this.user.energy) / 5)
            const have = parseInt(this.user.ressources["PWR"])
            console.log("need to refill and got food");
            console.log("needed", needed)
            console.log("have", have)
            console.log(parseInt(have < needed))
            let n = this.user.max_energy - this.user.energy
            if (have < needed){
              n = have * 5
            }
            n--;
            console.log("asked for", n)
            const r_action = {
                  actions: [
                    {
                      account: "cryptofarmsg",
                      name: "recoverpower",
                      authorization: [
                        {
                          actor: this.$store.state.user.name,
                          permission: "active",
                        },
                      ],
                      data: {
                        owner: this.$store.state.user.name,
                        energy: n,
                      },
                    },
                  ],
                };
                const r_block = {
                  blocksBehind: 3,
                  expireSeconds: 30,
                };
                const r_transac = {
                  id: "refill",
                  action: r_action,
                  block: r_block,
                };
                this.pushed = true
                this.$store.commit("user/addRAction", r_transac);
                setTimeout(() => {
                  this.pushed = false
                   console.log("energy available")
                }, 60000)
                console.log("asked for energy")
          }
          else {
            console.log("need to refill but no food");
          }
        }
        else {
          console.log("no need")
        }
      }, 15000);
    },
    stop() {
      clearInterval(this.refresh);
    },
    switchRef() {
      this.refill = !this.refill;
      if (this.refill) {
        this.launch();
        localStorage.setItem("energy", "true")
      }
      else {
        this.stop();
        localStorage.setItem("energy", "false")
      }
    }
  }
}
</script>
