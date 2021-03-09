<template>
  <div id="app">
    
    <br>
    <br>
    
    <button @click="approve()">
      Approve
    </button>

    <button @click="addLiquidity()">
        addLiquidity
    </button>
    
  </div>
</template>

<script>
import Web3 from 'web3';
import { ChainId, Fetcher, Trade, Route, TokenAmount, TradeType, Percent } from '@uniswap/sdk'
import IUniswapV2Router02 from './IUniswapV2Router02.json';
import ERC20 from './ERC20.json';

import account from '../account.json';
const Tx = require('ethereumjs-tx'); // day la thu vien gi ? 

export default {
  name: 'App',
  data: function () {
    return {
      WEENUS: '0x101848d5c5bbca18e6b4431eedf6b95e9adf82fa',
      WETH: '0xc778417e063141139fce010982780140aa0cd5ab',
      isMetamask: false,
    }
  },
 
  methods: {

    // Guide 11 - ==================================================
    async approve () {
      let web3 = window.web3
      let contract = await this.getContractInstance(web3); // CÁC INSTANT CỦA SMART CONTRACT ĐÓ 
   
      const token = new web3.eth.Contract(ERC20.abi, this.WEENUS);

      // 1. Approve
      const decimals = await token.methods.decimals().call(); // LAY RA SÓ decimals ( 18 .....) 
      const amountIn = (100 * 10 ** decimals).toString(); // nhập số lượng truyền vào nhân vs 18 , 16

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

    }, // =====> đã approve xong , bây giờ có thể thực hiện giao dịch vs đồng WEENUS

    // ===== addEqiulityETH

    async addLiquidity() {


      let web3 = window.web3
      let contract = await this.getContractInstance(web3);
       const token = new web3.eth.Contract(ERC20.abi, this.WEENUS); 
       const decimals = await token.methods.decimals().call();


        const amountTokenDesired =   (2 * 10 ** decimals).toString();
        const amountTokenMin = 1
        const amountETHMin = 0
        const to = account.address
        const deadline =  Date.now() + 1000 * 100 //deadline

        if (this.isMetamask) {
        // Call smart contract via metamask
        contract.methods.addLiquidityETH
        console.log("login meta")
      } else {
          console.log("chay")
        // Call smart contract via private key
        const data = contract.methods.addLiquidityETH(
          this.WEENUS,
          '10000000000000000000',
          '0',
          '0',
          to,
          deadline
        );

        let test_value = 0.000156816 & 1000000000000000000;
        const params = {
          nonce: web3.utils.toHex(await web3.eth.getTransactionCount(account.address)),
          gasLimit: 3000000,
          gasPrice: web3.utils.toHex(await web3.eth.getGasPrice()),
          to: contract._address,
          data: data.encodeABI(),
          value: web3.utils.toHex(test_value) // 0
        }

        await this.signTransaction(params);
      }
    },


 






    async signTransaction (params) {
      const tx = new Tx(params, { chain: 'ropsten' });
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
          'https://ropsten.infura.io/v3/8cc494f07a9344fab0e47d6b11260efc'
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
