<template>
  <div>
    <div class="row">
      <div class="col-12 text-center">
        <img :src="cow.image" class="tokenlogo">
        <div class="name">{{cow.name}}</div>
        <div class="desc">{{cow.desc}}</div>
      </div>
    </div>
    <br>
    <div class="row">
      <div class="col-6">
        <div class="card text-center">
          <div class="card-body">
            <h5 class="card-title">{{ rewards }}</h5>
            <p class="card-text">{{$t('cow.earned')}}</p>
            <b-button block @click="onClaim" variant="success">
             {{$t('cow.harvest')}}
            </b-button>
          </div>
        </div>
      </div>
      <div class="col-6">
        <div class="card text-center">
          <div class="card-body">
            <h5 class="card-title">{{ stakingBalance }} </h5>
            <p class="card-text">{{$t('cow.symbol-staked', {symbol: cow.stakeToken.symbol})}}</p>
            <b-button block @click="onApprove" v-if="stakeAllowance.lte(toBigNumber(stakeAmount))" variant="danger">
             {{$t('cow.approve-symbol', {symbol: cow.stakeToken.symbol})}}
            </b-button>
            <b-button block v-else @click="$bvModal.show('stake-modal')" variant="primary">
              {{$t('cow.stake')}}
            </b-button>
          </div>
        </div>
      </div>
    </div>
    <br>
    <div class="row">
      <div class="col-12 text-center">
        <b-button @click="onExit" variant="primary">
          {{$t('cow.harvest-unstake')}}
        </b-button>
      </div>
    </div>


    <b-modal id="stake-modal" hide-footer size="md">
      <template v-slot:modal-title="{ close }">
        <b>{{$t('cow.stake')}}</b>
      </template>
      <b-form>
        <div>{{$t('cow.balance')}}<b class="balance" @click="fillStakeAmount(stakeWalletBalance)">{{ stakeWalletBalance }}</b> {{ cow.stakeToken.symbol}}</div>
        <b-form-group id="input-group-1" label="" label-for="input-1">
          <b-form-input
            id="input-1"
            type="number"
            v-model="stakeAmount"
            autocomplete="off"
            required
          ></b-form-input>
          <b-form-invalid-feedback :state="validationAmount(stakeAmount)">
            {{$t('cow.greater-than')}}
          </b-form-invalid-feedback>
        </b-form-group>

        <b-button block @click="onStake" variant="success" v-if="stakeAllowance.gte(toBigNumber(stakeAmount))" :disabled="!validationAmount(stakeAmount) || txStatus == 'pending' || toBigNumber(stakeWalletBalance).lte(0)" >{{$t('cow.stake')}}</b-button>
      </b-form>
      <br>
      <br>
    </b-modal>

    <b-modal id="unstake-modal" hide-footer size="md">
      <template v-slot:modal-title="{ close }">
        <b>{{$t('cow.unstake')}}</b>
      </template>
      <b-form>
        <div>{{$t('cow.staking-balance', {stakingBalance: stakingBalance, symbol: cow.stakeToken.symbol})}}</div>
        <b-form-group id="input-group-1" label="" label-for="input-1">
          <b-form-input
            id="input-1"
            v-model="unstakeAmount"
            placeholder="Amount to unstake"
            autocomplete="off"
            required
          ></b-form-input>
          <b-form-invalid-feedback :state="validationAmount(unstakeAmount)">
            {{$t('cow.amount-greater-than')}}
          </b-form-invalid-feedback>
        </b-form-group>
        <b-button block @click="onUnstake" variant="success" :disabled="!validationAmount(unstakeAmount) || txStatus == 'pending'">{{$t('cow.unstake')}}</b-button>
      </b-form>
      <br>
      <br>
    </b-modal>

    <b-toast id="tx-toast" solid no-auto-hide>
      <template v-slot:toast-title>
        <div class="tx-status">
          <b-spinner small label="Loading..." v-if="txStatus == 'pending'"></b-spinner>
          <b-icon icon="check-circle" v-if="txStatus == 'mined'"></b-icon>
          <b-icon icon="x-circle" variant="danger" v-if="txStatus == 'error'"></b-icon>
          {{ txStatus }}
        </div>
      </template>

      <div class="tx-hash">
        <div><a :href="etherscanTx(txHash)" target="_blank">{{ txHash }}</a></div>
      </div>
    </b-toast>

  </div>
</template>

<script>
  import utils from '~/mixins/utils'
  import { Cow, Erc20 } from '~/contracts'
  import { toBN, BN, isBN } from 'web3-utils'
  import { BigNumber } from 'bignumber.js'
  import config from '~/config'

  export default {
    asyncData({params}) {
      let cow = config.cows.find((cow)=>{
        return cow.id == 1
      });
      return {
        cow: cow,
        stakeAddress: '',
        txid: '',
        show: true,
        submitDisabled: false,
        titleStatus: '',
        txStatus: 'null',
        stakeWalletBalance: '0',
        stakingBalance: '0',
        rewards: '0',
        isBalanceLoaded: false,
        stakeAmount: '',
        unstakeAmount: '',
        allowed: true,
        startTime: '',
        finishTime: '',
        txHash: '',
        claimDisabled: false,
        exitDisabled: false,
        initReward: '',
        stakeAllowance: BigNumber(0)
      }
    },
    computed: {
      // validationAmount(v) {
      //   return v && parseFloat(v) > 0
      // },
    },
    methods: {
      toBigNumber(v) {
        return v ? BigNumber(v) : BigNumber(0)
      },
      validationAmount(v) {
        return v && parseFloat(v) > 0;
      },

      etherscanAddress(addr) {
        return utils.etherscanAddr(addr)
      },
      etherscanTx(tx) {
        return utils.etherscanTx(tx)
      },

      cowLoaded() {
        return !!this.cowContract
      },

      onApprove() {
        if(!this.cowLoaded()) return;
        this.stakeToken.approveTest(this.$store.state.connectedAccount, this.cow.address, (err, txHash) => {
          console.log("tttt====",err, txHash);

          if(txHash) {
            alert(txHash)
            this.txHash = txHash;
            this.txStatus = 'pending';
            this.$bvToast.show('tx-toast')
          } else{
            console.log(err)
            alert(err)
          }
        }).then((receipt, hash) => {
          console.log("receipt", receipt)
          console.log("hash", hash)
          this.txStatus = 'mined';
          this.update()
          this.$bvToast.hide('tx-toast')
        }).catch(err => {
          console.log("err", err)
          this.$bvModal.hide('stake-modal')
        })
      },
      onStake() {
        if(!this.cowLoaded()) return;
        let amount = parseFloat(this.stakeAmount);
        if(amount <= 0) return;
        this.txStatus = 'pending';
        this.cowContract.stake(this.$store.state.connectedAccount, amount, (err, txHash)=>{
          if(txHash) {
            this.txHash = txHash;
            this.txStatus = 'pending';
            this.$bvToast.show('tx-toast')
          }
          this.$bvModal.hide('stake-modal')
        }).then((receipt, b) => {
          this.txStatus = 'mined';
          this.update()
          this.$bvToast.hide('tx-toast')
        }).catch(err => {
          this.txStatus = 'error';
        });
      },
      onUnstake() {
        if(!this.cowLoaded()) return;
        let amount = parseFloat(this.unstakeAmount);
        if(amount <= 0) return;
        this.txStatus = 'pending';
        this.cowContract.withdraw(this.$store.state.connectedAccount, amount, (err, txHash) => {
          if(txHash) {
            this.txHash = txHash;
            this.txStatus = 'pending';
            this.$bvToast.show('tx-toast')
          }
          this.$bvModal.hide('unstake-modal')
        }).then(receipt => {
          this.txStatus = 'mined';
          this.update()
          this.$bvToast.hide('tx-toast')
        }).catch(err => {
          this.$bvModal.hide('unstake-modal')
          this.txStatus = 'error';
        })
      },

      onExit() {
        if(!this.cowLoaded()) return;
        if(!confirm("Are you sure to exit?")) return;
        this.exitDisabled = true;
        this.cowContract.exit(this.$store.state.connectedAccount, (err, txHash) => {
          if(txHash) {
            this.txHash = txHash;
            this.txStatus = 'pending';
            this.$bvToast.show('tx-toast')
          }
          this.exitDisabled = false;
        }).then(receipt => {
          this.txStatus = 'mined';
          this.exitDisabled = false;
        }).catch(err => {
          this.update()
          this.txStatus = 'error';
          this.exitDisabled = false;
        })
      },
      onClaim() {
        if(!this.cowLoaded()) return;
        this.claimDisabled = true;
        this.cowContract.getReward(this.$store.state.connectedAccount, (err, txHash) => {
          if(txHash) {
            this.txHash = txHash;
            this.txStatus = 'pending';
            this.$bvToast.show('tx-toast')
            this.claimDisabled = false;
          }
        }).then(receipt => {
          this.txStatus = 'mined';
          this.claimDisabled = false;
          this.update()
          this.$bvToast.hide('tx-toast')
        }).catch(err => {
          this.txStatus = 'error';
          this.claimDisabled = false;
        })
      },

      fillStakeAmount(v) {
        this.stakeAmount = v.toString();
      },
      async update() {
        this.stakeWalletBalance = await this.stakeToken.balanceOf(this.$store.state.connectedAccount);
        this.stakingBalance = await this.cowContract.balanceOf(this.$store.state.connectedAccount);
        this.rewards = await this.cowContract.earned(this.$store.state.connectedAccount);
        this.stakeAllowance = await this.stakeToken.allowance(this.$store.state.connectedAccount, this.cow.address);
      }
    },
    async mounted() {
      await this.$onConnect();
      let cow = new Cow(this.cow.address, this.cow.stakeToken, this.cow.yieldToken);
      let stakeToken = new Erc20(this.cow.stakeToken.address);
      let yieldToken = new Erc20(this.cow.yieldToken.address);

      if(this.cow.initialized) {
        this.stakeWalletBalance = await stakeToken.balanceOf(this.$store.state.connectedAccount);
        this.stakingBalance =  await cow.balanceOf(this.$store.state.connectedAccount)
        this.rewards = await cow.earned(this.$store.state.connectedAccount);
        let stakeAllowance =  await stakeToken.allowance(this.$store.state.connectedAccount, this.cow.address);
        this.cowContract = cow;
        this.stakeToken = stakeToken;
        this.yieldToken = yieldToken;
        this.stakeAllowance = stakeAllowance;
        setInterval(this.update, 10 * 1000)
      }
    }
  }
</script>

<style scoped>
  .cow {
    width: 8rem;
  }
  .avatar {
    font-size: 5rem;
  }
  .name {
    font-size: 1.2rem;
  }
  .desc {
    color: #666;
  }
  .info {
    background-color: #eee;
    padding: 1rem;
  }
  .write-form {
    max-width: 30rem;
  }
  .connect {
    background-color: #e9ecef;
    padding: 1rem;
    margin-bottom: 1rem;
  }
  .address {
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .balance {
    cursor: pointer;
  }
  .toast-body {
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .tx-hash {
    margin-bottom: 0.5rem;
    margin-top: 0rem;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .tx-status {
    text-transform: capitalize;
  }
  .tokenlogo {
    width: 150px;
  }
</style>
