<template>
  <div id="app">
    <input type="radio" v-model="isMetamask" value="false">PrivateKey
    <input type="radio" v-model="isMetamask" value="true">Metamask
    <br>
    <br>
    <br>
    <select v-model="from">
      <option v-for="token in listToken">
        {{ token.symbol }} {{ token.address }}
      </option>
    </select>
    <select v-model="to">
      <option v-for="token in listToken">
        {{ token.symbol }} {{ token.address }}
      </option>
    </select>
    <br>
    <br>
    <br>
    <input type="text" v-model="numberFrom">
    <input disabled type="text" v-model="numberTo">
    <br>
    <br>
    <br>
    <button @click="getBalance()">
      Show Balance
    </button>
    <button @click="approve()">
      Approve
    </button>
    <button @click="swap()">
      Swap
    </button>
    <button @click="getPrice()">
      Get Price
    </button>
    <button @click="getMinimumReceived()">
      Get Minimum Received
    </button>
    <button @click="getPriceImpact()">
      Get Price Impact
    </button>
    <button @click="getFee()">
      Get Fee
    </button>
    <TradingRoute />
  </div>
</template>

<script>
import Web3 from 'web3';
import { ChainId, Fetcher, Trade, Route, TokenAmount, TradeType, Percent, WETH as weth } from '@uniswap/sdk'
import DEFAULT_TOKEN_LIST from './rinkebyToken.json'
import IUniswapV2Router02 from './IUniswapV2Router02.json';
import TradingRoute from './TradingRoute'
import ERC20 from './ERC20.json';
import account from '../account.json';
const Tx = require('ethereumjs-tx');

export default {
  name: 'App',
  data: function () {
    return {
      UNI: '0x1f9840a85d5af5bf1d1762f925bdaddc4201f984',
      WETH: weth[ChainId.RINKEBY].address,
      isMetamask: false,
      from: '',
      to: '',
      listToken: DEFAULT_TOKEN_LIST,
      addressFrom: '', // là address của cái j ? 
      addressTo: '',
      numberFrom: '',
      numberTo: '',
    }
  },
  components: {
    TradingRoute
  },
  watch: {
    async isMetamask () {
      if (this.isMetamask) {
        await this.connectToMetaMask()
      } else {
        await this.connectByPrivateKey()
      }
    },
    from (value) {
      this.addressFrom = value.split(' ')[1]
    },
    to (value) {
      this.addressTo = value.split(' ')[1]
    },
    async numberFrom (value) {
      const route = await this.getRoute(this.addressFrom, this.addressTo)
      this.numberTo = route.midPrice.invert().toFixed() * value
    },
  },
  methods: {

    // Guide 11 - ==================================================
    async approve () {
      let web3 = window.web3
      let contract = await this.getContractInstance(web3);

      const token = new web3.eth.Contract(ERC20.abi, this.addressFrom);

      const decimals = await token.methods.decimals().call();
      const amountIn = (1 * 10 ** decimals).toString();

      if (this.isMetamask) {
        // Call smart contract via metamask
        token.methods.approve(
          contract._address,
          amountIn
        ).send({ from: account.address });
      } else {
        // Call smart contract via private key
        const data = token.methods.approve(
          contract._address,
          amountIn
        );

        const params = {
          nonce: web3.utils.toHex(await web3.eth.getTransactionCount(account.address)),
          gasLimit: web3.utils.toHex(await data.estimateGas({ from: account.address })),
          gasPrice: web3.utils.toHex(await web3.eth.getGasPrice()),
          to: this.UNI,
          data: data.encodeABI(),
          value: '0x00' // 0
        }

        await this.signTransaction(params)
      }

    },
    // ============================================================


    // Guide 7 - ==================================================
    async swap () {
      const tokenForSwap = this.numberFrom; // input from ? 

      let web3 = window.web3
      let contract = await this.getContractInstance(web3);

      const token = new web3.eth.Contract(ERC20.abi, this.addressFrom); // token của 1 đồng from , kết nối vs smart contract

 

      const decimals = await token.methods.decimals().call(); // lay demical
      const amountIn = (tokenForSwap * 10 ** decimals).toString(); // so luong nhap vao

      const path = [this.addressFrom, this.addressTo]; // 2 cap token cần swap
      const amountOutMin = '0'; // so luong nho nhat co the ra

      let methodName = '' // các case để swap ? 
      if (this.WETH == this.addressFrom) {
        methodName = '' //4
      } else if (this.WETH == this.addressTo) {
        methodName = 'swapExactTokensForETH' //5
      } else {
        methodName = 'swapExactTokensForTokens' //5
      }

      const input1 = [amountIn] // DAY LÀ SỐ LUONGJ NHẬP VÀO :  1
      const input2 = [amountOutMin, path, account.address, Date.now() + 1000] // ĐÂY LÀ CÁI J ? 4
      const input = this.WETH == this.addressFrom ? input2 : input1.concat(input2) // ? 

      if (this.isMetamask) {
        // Call smart contract via metamask
        contract.methods[methodName](...input)
          .send(this.WETH == this.addressFrom
            ? { from: account.address, value: amountIn }
            : {from: account.address}
          )
      } else {
        // Call smart contract via private key
        const data = contract.methods[methodName](...input);
        const params = {
          nonce: web3.utils.toHex(await web3.eth.getTransactionCount(account.address)),
          gasLimit: web3.utils.toHex(await data.estimateGas(
            { from: account.address, value: this.WETH == this.addressFrom ? web3.utils.toHex(amountIn) : '0x00' }
          )),
          gasPrice: web3.utils.toHex(await web3.eth.getGasPrice()),
          to: contract._address,
          data: data.encodeABI(),
          value: this.WETH == this.addressFrom ? web3.utils.toHex(amountIn) : '0x00'
        }

        await this.signTransaction(params);
      }
    },
    // ============================================================


    // Guide 1 2 4 5 - ==================================================
    async getBalance () {
      const eth = await window.web3.eth.getBalance(account.address)
      console.log(eth);
      const tokenContract = new window.web3.eth.Contract(ERC20.abi, this.UNI);
      const token = await tokenContract.methods.balanceOf(account.address).call()
      console.log(token)
    },
    // ============================================================


    // Guide 3 - ==================================================
    getAddress () {
      console.log(account.address) }, // ============================================================


    // Guide 6 - ==================================================
    async getPrice () {
      const route = await this.getRoute()
      console.log(route.midPrice.toFixed()) // 1 weth = x uni
      console.log(route.midPrice.invert().toFixed()) // 1 uni = y weth
    },
    // ============================================================


    // Guide 8 - ==================================================
    async getMinimumReceived () {
      const trade = await this.getTrade();
      const slippageTolerance = new Percent('50', '10000') // 50 bips, or 0.50%
      console.log(trade.minimumAmountOut(slippageTolerance).toFixed())
    },
    // ============================================================


    // Guide 9 - ==================================================
    async getPriceImpact () {
      const trade = await this.getTrade();
      console.log(trade.priceImpact.toFixed())
    },
    // ============================================================


    // Guide 10 - ==================================================
    getFee () {
      const amount = 1;
      console.log(amount * 0.003)
    },
    // ============================================================


    async getRoute (addressFrom = this.UNI, addressTo = this.WETH) {
      const tokenFrom = await Fetcher.fetchTokenData(ChainId.RINKEBY, addressFrom)
      const tokenTo = await Fetcher.fetchTokenData(ChainId.RINKEBY, addressTo)
      const pair = await Fetcher.fetchPairData(tokenFrom, tokenTo)
      const route = new Route([pair], tokenTo)
      return route
    },

    async getTrade (addressFrom = this.UNI, addressTo = this.WETH) { // ? ?
      const tokenFrom = await Fetcher.fetchTokenData(ChainId.RINKEBY, addressFrom)
      const tokenTo = await Fetcher.fetchTokenData(ChainId.RINKEBY, addressTo)
      const pair = await Fetcher.fetchPairData(tokenFrom, tokenTo)
      const route = new Route([pair], tokenTo)

      const amountIn = '1000000000000000000' // 1 WETH
      const trade = new Trade(route, new TokenAmount(tokenTo, amountIn), TradeType.EXACT_INPUT)
      return trade
    },

    async signTransaction (params) {
      const tx = new Tx(params, { chain: 'rinkeby' });
      tx.sign(Buffer.from(account.private, 'hex'))
      const serializeTx = tx.serialize();
      await window.web3.eth.sendSignedTransaction('0x' + serializeTx.toString('hex'), function(err, hash) {
        if (!err) {
          console.log(hash);
        } else {
          console.log(err)
        }
      })
    },

    getContractInstance (web3Instance) {
      const router2Addr = '0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D'
      const contractAbi = IUniswapV2Router02.abi;
      return new web3Instance.eth.Contract(contractAbi, router2Addr);
    },

    getTokenContract (web3Instance, address) {
      const tokenAbi = ERC20.abi;
      return new web3Instance.eth.Contract(tokenAbi, address);
    },

    async connectToMetaMask() {
      if (window.ethereum) {
        window.web3 = new Web3(window.ethereum);
        await window.ethereum.enable();
      }
    },

    async connectByPrivateKey() {
      try {
        const provider = new Web3.providers.HttpProvider(
          'https://rinkeby.infura.io/v3/2806c626047f4fb590c7e20593b7dd73'
        );
        window.web3 = new Web3(provider);
      } catch (err) {

      }
    }
  },

  async mounted () {
    if (this.isMetamask) {
      await this.connectToMetaMask()
    } else {
      await this.connectByPrivateKey()
    }
  },
  created () {

  }
}
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
button, select, input[type=text] {
  margin-right: 10px;
}
select, input[type=text] {
  width: 200px;
}
</style>
