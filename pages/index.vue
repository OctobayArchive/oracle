<template>
  <div class="container mt-5 text-center">
    <h1>Octobay Oracle Demo</h1>
    <div class="alert alert-info mx-auto mt-4 mb-5" style="max-width: 360px">
      Make sure you are connected to the Kovan test network in your MetaMask
      wallet and connect to your GitHub account. You can then pay the oracle
      (0.001 ETH) to update the ETH price on the
      <a
        href="https://kovan.etherscan.io/address/0x183BB83438307a04132f48EE9649D8534d786cE0"
        target="_blank"
      >
        example contract </a
      >.
    </div>
    <p v-if="githubUser" class="lead">Hi {{ githubUser.login }}!</p>
    <button class="btn btn-primary" @click="getEthPrice">Get ETH Price</button>
    <button v-if="githubUser" class="btn btn-primary" @click="updateEthPrice">
      Update ETH Price
    </button>
    <a
      v-else
      class="btn btn-primary"
      href="https://github.com/login/oauth/authorize?scope=user:email,public_repo&client_id=376c59cbd5e3b4d3c9a6"
    >
      Connect to GitHub
    </a>
    <h5 class="mt-3">Current ETH price: {{ ethPrice }}</h5>
    <div v-if="updated" class="text-success">Updated!</div>
  </div>
</template>

<script>
import Web3 from 'web3'
import ABI from './../contract.abi.json'

export default {
  data() {
    return {
      ethPrice: 0,
      web3: null,
      contractAddress: '0x183BB83438307a04132f48EE9649D8534d786cE0',
      contract: null,
      oracle: '0xA56d9e73f98212e56A2eFb00c9F47d1da64937ee',
      updated: false,
      accounts: [],
      githubUser: null,
      githubAccessToken: null,
    }
  },
  mounted() {
    if (window.ethereum) {
      this.web3 = new Web3(window.ethereum)
      this.contract = new this.web3.eth.Contract(ABI, this.contractAddress)
      this.getEthPrice()
      this.connect()

      const code = this.$route.query.code
      if (code) {
        this.$axios
          .$post('https://octobay.uber.space/oracle-demo/github/access-token', {
            code,
          })
          .then((response) => {
            this.$axios
              .$get(`https://api.github.com/user`, {
                headers: { Authorization: 'bearer ' + response.accessToken },
              })
              .then((githubUser) => {
                this.githubUser = githubUser
                this.githubAccessToken = response.accessToken
              })
          })
      }
    } else {
      alert('You need an Ethereum compatible browser to try this demo.')
    }
  },
  methods: {
    connect() {
      this.web3.eth
        .requestAccounts()
        .then((accounts) => (this.accounts = accounts))
    },
    getEthPrice() {
      this.contract.methods
        .ethPrice()
        .call()
        .then((ethPrice) => {
          this.ethPrice = ethPrice
          this.updated = true
          setTimeout(() => (this.updated = false), 3000)
        })
    },
    updateEthPrice() {
      this.web3.eth
        .sendTransaction({
          from: this.accounts[0],
          to: this.oracle,
          value: this.web3.utils.toWei('0.001', 'ether'),
        })
        .then((tx) => {
          this.$axios.$post(
            'https://api.github.com/repos/octobay/oracle/issues',
            {
              title: '[ETHUSD]',
              body: `${tx.transactionHash}
              ${this.contractAddress}:setEthPrice(uint256)`,
            },
            {
              headers: { Authorization: 'bearer ' + this.githubAccessToken },
            }
          )
        })
    },
  },
}
</script>
