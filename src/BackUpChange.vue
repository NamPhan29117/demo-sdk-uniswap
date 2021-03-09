<template>
  <div id="app">
    <input type="radio" v-model="isMetamask" value="false">PrivateKey
    <input type="radio" v-model="isMetamask" value="true">Metamask
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
  </div>
</template>

<script>
import Web3 from 'web3';
import { ChainId, Fetcher, Trade, Route, TokenAmount, TradeType, Percent } from '@uniswap/sdk'
import IUniswapV2Router02 from './IUniswapV2Router02.json';
import ERC20 from './ERC20.json';
// account.json format
// {
//     "address": "",
//     "private": ""
// }
import account from '../account.json';
const Tx = require('ethereumjs-tx'); // day la thu vien gi ? 

export default {
  name: 'App',
  data: function () {
    return {
      UNI:"0x1f9840a85d5af5bf1d1762f925bdaddc4201f984",
      WEENUS: '0x101848d5c5bbca18e6b4431eedf6b95e9adf82fa',
      WETH: '0xc778417e063141139fce010982780140aa0cd5ab',
      isMetamask: false,
    }
  },
  watch: {
    async isMetamask () {
      if (this.isMetamask) {
        await this.connectToMetaMask()
      } else {
        await this.connectByPrivateKey()
      }
    }
  },
  methods: {

    // Guide 11 - ==================================================
    async approve () {
      let web3 = window.web3
      let contract = await this.getContractInstance(web3); // CÁC INSTANT CỦA SMART CONTRACT ĐÓ 
      console.log("zzzzz", contract._address,)
      const token = new web3.eth.Contract(ERC20.abi, this.WEENUS);

      // 1. Approve
      const decimals = await token.methods.decimals().call(); // LAY RA SÓ decimals (16, 18 .....) 
      const amountIn = (1 * 10 ** decimals).toString(); // nhập số lượng truyền vào nhân vs 18 , 16

      if (this.isMetamask) {
        // Call smart contract via metamask
        token.methods.approve(
          contract._address, // địa chỉ của smart contract hiện tạo m đang thực hiện
          amountIn
        ).send({ from: account.address });
      } else {
        // Call smart contract via private key
        const data = token.methods.approve(
          contract._address,
          amountIn
        );

        const params = {
          nonce: web3.utils.toHex(await web3.eth.getTransactionCount(account.address)), //vị trí của action đó trong list smart contract (number)
          gasLimit: web3.utils.toHex(await data.estimateGas({ from: account.address })), // phí giao dịch tối đa mà mình phải chịu (number)
          gasPrice: web3.utils.toHex(await web3.eth.getGasPrice()), // phí giao dịch (number)
          to: this.WEENUS,  // đến đâu ? . địa chỉ cái đồng mình cần xác nhận
          data: data.encodeABI(), // nội dung m gửi lên (noi dung smart contract được viết file JSON, cụ thể là hàm approve)
          value: '0x00' // 0
        }

        await this.signTransaction(params) // Kí 1 transaction , sau kí xong thì chạy hàm confirm (nằm trong await) là đã kí rồi
      }

    },
    // ============================================================


    // Guide 7 - ==================================================
    async swap () {
      const tokenForSwap = 1; // so luong ban muon swap

      let web3 = window.web3
      let contract = await this.getContractInstance(web3);

      const token = new web3.eth.Contract(ERC20.abi, this.WEENUS);

     

      const decimals = await token.methods.decimals().call();
      const amountIn = (tokenForSwap * 10 ** decimals).toString();

      // 2. Swap
      const path = [this.UNI, this.WETH]; // đia chỉ của 2 token
      const amountOutMin = '0'; // số lượng token ra nhỏ nhất mà b cho phép

      if (this.isMetamask) {
        // Call smart contract via metamask
        contract.methods.swapExactTokensForETH(
          amountIn,
          amountOutMin,
          path,
          account.address, // đia chi vi cua ban
          Date.now() + 1000 //deadline
        ).send({ from: account.address });;
      } else {
        // Call smart contract via private key
        const data = contract.methods.swapExactTokensForETH(
          amountIn,
          amountOutMin,
          path,
          account.address,
          Date.now() + 1000
        );
        const params = {
          nonce: web3.utils.toHex(await web3.eth.getTransactionCount(account.address)),
          gasLimit: web3.utils.toHex(await data.estimateGas({ from: account.address })),
          gasPrice: web3.utils.toHex(await web3.eth.getGasPrice()),
          to: contract._address,
          data: data.encodeABI(),
          value: '0x00' // 0
        }

        await this.signTransaction(params);
      }
    },
    // ============================================================


    // Guide 1 2 4 5 - ==================================================
    async getBalance () {
      const eth = await window.web3.eth.getBalance(account.address)
      console.log(eth); // sỐ DƯ ETH đang có trong ví của mình
      const tokenContract = new window.web3.eth.Contract(ERC20.abi, this.WEENUS); // TƯƠNG TÁC VS SMART CONTRACT
      const token = await tokenContract.methods.balanceOf(account.address).call() // => DAY LÀ CÁI J ?
      console.log(token)
    },
    // ============================================================


    // Guide 3 - ==================================================
    getAddress () {
      console.log(account.address)
    },
    // ============================================================


    // Guide 6 - ==================================================
    async getPrice () {
      const route = await this.getRoute()
      console.log(route.midPrice.toFixed()) // 1 weth = x uni //=> GIÁ CẢ TRUNG BÌNH GIỮA 1 ĐỒNG A VÀ B
      console.log(route.midPrice.invert().toFixed()) // 1 uni = y weth //=> GIÁ CẢ TRUNG BÌNH GIỮA 1 ĐỒNG B VÀ A
    },
    // ============================================================


    // Guide 8 - ==================================================
    async getMinimumReceived () { // lấy giá cả nhỏ nhất mình có thể nhận được
      const trade = await this.getTrade();
      const slippageTolerance = new Percent('50', '10000') // 50 bips, or 0.50%
      console.log(trade.minimumAmountOut(slippageTolerance).toFixed()) 
      // slippageTolerance // => độ trượt giá
    },
    // ============================================================


    // Guide 9 - ==================================================
    async getPriceImpact () { // giá Impact 
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


    async getRoute () {
      const tokenUni = await Fetcher.fetchTokenData(ChainId.RINKEBY, this.UNI)
      const tokenWeth = await Fetcher.fetchTokenData(ChainId.RINKEBY, this.WETH)
      const pair = await Fetcher.fetchPairData(tokenUni, tokenWeth)
      const route = new Route([pair], tokenWeth)
      return route
    },

    async getTrade () { // la gi ? 
      const tokenUni = await Fetcher.fetchTokenData(ChainId.RINKEBY, this.UNI)
      const tokenWeth = await Fetcher.fetchTokenData(ChainId.RINKEBY, this.WETH)
      const pair = await Fetcher.fetchPairData(tokenUni, tokenWeth)
      const route = new Route([pair], tokenWeth)

      const amountIn = '1000000000000000000' // 1 WETH
      const trade = new Trade(route, new TokenAmount(tokenWeth, amountIn), TradeType.EXACT_INPUT)
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

    getContractInstance (web3Instance) { // lay instandce của 1 smart contract
      const router2Addr = '0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D' // ĐỊA CHỈ CỦA SMART CONTRACT
      const contractAbi = IUniswapV2Router02.abi;
      return new web3Instance.eth.Contract(contractAbi, router2Addr);
    },

    getTokenContract (web3Instance, address) { // lấy token của 1 contract
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
button {
  margin-right: 10px;
}
</style>
