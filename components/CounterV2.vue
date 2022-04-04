<template>
  <div>
    <button class="DisBtn" v-if="!readyToClaim" :disabled="!checkManual()" @click="manualClaim">
      {{ remainingTime }}
    </button>
    <button class="EnBtn" v-else @click="handleClaim">{{ msg }}</button>
    <!--<button @click="test">test</button>-->
    <b-spinner label="Spinning" v-if="wait" class="spinner-login"></b-spinner>
  </div>
</template>

<script>
const utcNow = require('utc-now');
export default {
  name: "Counter",
  props: ["timestamp", "autoclaim", "item", "claiminfo"],
  data() {
    return {
      msg: "Ready to claim",
      wait: false,
      loaded: false,
      readyToClaim: false,
      last: undefined,
      queued: false,
      displayHours: 0,
      displayMinutes: 0,
      displaySeconds: 0,
      manual: false
    };
  },
  computed: {
    ISreadyToClaim() {
      return this.readyToClaim;
    },
    _seconds() {
      return 1000;
    },
    _minutes() {
      return this._seconds * 60;
    },
    _hours() {
      return this._minutes * 60;
    },
    remainingTime() {
      if (this.loaded) {
        return (
          this.displayHours +
          ":" +
          this.displayMinutes +
          ":" +
          this.displaySeconds
        );
      } else {
        return "Getting remaining time...";
      }
    },
  },
  watch: {
    // eslint-disable-next-line object-shorthand
    timestamp: function () {
      this.loaded = false;
      // this.readyToClaim = false;
      this.showRemaining();
    },
    // eslint-disable-next-line object-shorthand
    autoclaim: function () {
      if (this.autoclaim) {
        this.loaded = false;
        // this.readyToClaim = false;
        this.showRemaining();
      }
    },
  },
  mounted() {
    // console.log(this.timestamp)
    this.showRemaining();
    this.$on(`${this.item.asset_id}.claiming`, () => {
      this.last = Date.now()
      console.log("received claiming!");
      this.msg = "Claiming...";
    });
    this.$on("test", (id) => {
      console.log("counter receive test", id);
    });
    this.$on(`${this.item.asset_id}.success`, () => {
      this.last = Date.now()
      console.log("received success!");
      this.msg = "Ready to claim";
      this.readyToClaim = false;
      this.queued = false;
      setTimeout(() => {
        this.last = undefined;
      }, 10000);
      this.showRemaining();
    });
    this.$on(`${this.item.asset_id}.retry`, (nb) => {
      console.log("received retry!");
      this.msg = `Retry to claim/repair... (${nb})`;
    });
  },
  methods: {
    /*test() {
      console.log("Integrity_usage :");
      console.log(this.item.current_durability);
      console.log(this.item.durability_usage);
      console.log(this.item.current_durability >= this.item.durability_usage);
    },*/
    checkManual() {
      const real = this.timestamp  + (this.$store.state.user.members[this.claiminfo.type] * this.item.charged_time)
       const now = new Date();
       const end = new Date(real * 1000);
       return parseInt(this.$store.state.user.members[this.claiminfo.type] - ((end.getTime() - now.getTime()) / 1000 / this.item.charged_time) + 1).toFixed(0) > 0
    },
    manualClaim() {
      if (this.checkManual()){
        this.manual = true;
        this.handleClaim();
        setTimeout(() => {
          this.manual = false
        }, 5000)
      }
    },
    showRemaining() {
      const timer = setInterval(() => {

        const end = new Date(this.timestamp);
        const stake = this.timestamp
        const distance = end.getTime() - utcNow() ;
        if (
          this.$store.state.user.autorepair[this.claiminfo.type][
            this.item.asset_id
          ] === true &&
          !(
            this.$store.state.user.actions.find((x) => x.id === this.item.id) !=
            undefined
          )
        ) {
          if (
            this.claiminfo.type != "Members" && this.claiminfo.type != "Crops" &&
            this.item.current_durability <= this.item.durability_usage * this.$store.state.user.members[this.claiminfo.type] &&
            this.$store.state.user.autorepair[this.claiminfo.type][
              this.item.asset_id
            ]
          ) {
            let cost =
              (this.item.durability - this.item.current_durability) * 0.3;
              const r_action = {
                actions: [
                  {
                    account: "cryptofarmsg",
                    name: this.claiminfo.r_action,
                    authorization: [
                      {
                        actor: this.$store.state.user.name,
                        permission: "active",
                      },
                    ],
                    data: {
                      owner : this.$store.state.user.name,
                      asset_id: this.item.asset_id,
                    },
                  },
                ],
              };
              const r_block = {
                blocksBehind: 3,
                expireSeconds: 30,
              };
              const r_transac = {
                id: this.item.asset_id,
                action: r_action,
                block: r_block,
              };
              console.log("add to repair");
              this.$store.commit("user/addRAction", r_transac);
            
          }
        }
        if (distance <= 0) {
          clearInterval(timer);
          if (this.last == undefined) {
          this.readyToClaim = true;
          }
          this.loaded = true;
          if (
            this.$store.state.user.autoclaim[this.claiminfo.type][
              this.item.asset_id
            ] === true
          ) {
            if (this.claiminfo.type == "Crops" || this.claiminfo.type == "Members" || this.item.current_durability >= this.item.durability_usage * this.$store.state.user.members[this.claiminfo.type]) {
              this.handleClaim();
            }
          }
          return null;
        }
        else {
          this.readyToClaim = false;
        }

        const hours = Math.floor(distance / this._hours);
        const minutes = Math.floor((distance % this._hours) / this._minutes);
        const seconds = Math.floor((distance % this._minutes) / this._seconds);
        this.displayHours = hours < 10 ? "0" + hours : hours;
        this.displayMinutes = minutes < 10 ? "0" + minutes : minutes;
        this.displaySeconds = seconds < 10 ? "0" + seconds : seconds;

        this.loaded = true;
      }, 1000);
    },
    async handleClaim() {
      if (
        this.$store.state.user.actions.find((x) => x.id === this.item.id) !=
        undefined
      )
        return;
      if (!this.queue) {
        this.queued = true;
        try {

          const data = this.claiminfo.type != "Crops" ? {
                  owner: this.$store.state.user.name,
                  asset_id: this.item.asset_id,
                } : {
                  owner: this.$store.state.user.name,
                  crop_id: this.item.asset_id
                };


          const action = {
            actions: [
              {
                account: "cryptofarmsg",
                name: this.claiminfo.action,
                authorization: [
                  {
                    actor: this.$store.state.user.name,
                    permission: "active",
                  },
                ],
                data: data,
              },
            ],
          };
          const block = {
            blocksBehind: 3,
            expireSeconds: 30,
          };
          const transac = {
            id: this.item.asset_id,
            action: action,
            block: block,
          };
          if (this.last == undefined) {
            this.readyToClaim = true;
            console.log("add to queue");
            this.$store.commit("user/addAction", transac);
            this.msg = "Claim in queue...";
            if (
              this.$store.state.user.r_actions.find(
                (x) => x.id === this.item.id
              ) != undefined
            )
              return;
            if ( this.claiminfo.type != "Members" && this.claiminfo.type != "Crops" &&
              this.item.current_durability <= this.item.durability_usage * this.$store.state.user.members[this.claiminfo.type] &&
              this.$store.state.user.autorepair[this.claiminfo.type][
                this.item.asset_id
              ]
            ) {
              let cost =
                (this.item.durability - this.item.current_durability) * 0.2;
                const r_action = {
                  actions: [
                    {
                      account: "cryptofarmsg",
                      name: this.claiminfo.r_action,
                      authorization: [
                        {
                          actor: this.$store.state.user.name,
                          permission: "active",
                        },
                      ],
                      data: {
                        owner: this.$store.state.user.name,
                        asset_id: this.item.asset_id,
                      },
                    },
                  ],
                };
                const r_block = {
                  blocksBehind: 3,
                  expireSeconds: 30,
                };
                const r_transac = {
                  id: this.item.asset_id,
                  action: r_action,
                  block: r_block,
                };
                this.$store.commit("user/addRAction", r_transac);
              
            }
          }
          //alert("claiming !");
        } catch (e) {
          console.log(e);
        }
      }
    },
  },
};
</script>

<style scoped>
button {
  border-radius: 5px;
  border: none;
  font-weight: bold;
  box-shadow: 1px 1px 0px 1px rgba(0, 0, 0, 0.2);
}
.DisBtn {
  background: rgb(211, 140, 9);
  color: rgb(15, 12, 5);
  font-size: small;
}
.EnBtn {
  background: #28a745;
  color: rgb(15, 27, 14);
  font-size: small;
}
</style>
