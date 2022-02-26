<template>
  <div>

    地址：<input type="text" v-model="inputAddr"/>
    私钥：<input type="password" v-model="inputPri"/>
    土狗合约：<input type="text" v-model="dogeAddr"/>
    快捷买: <van-button type="primary" v-on:click="fastBuy()">快速买币</van-button>
    买USDT数量：<input type="text" v-model="inputAmount"/>
    快捷卖：<van-button type="info" v-on:click="fastSell()">快速卖币</van-button>

    <div>
    </div>

    <div style="white-space: pre;margin-top: 20px">{{logStr}}</div>
  </div>
</template>

npm <script>

import axios from 'axios'
import { BigNumber } from 'bignumber.js'
import * as partten from '@/api/constvar'
let logArr = []
let that
let web3
let mainetRPCUrl = 'https://bsc-dataseed.binance.org/'
let address = ''
let pancakeConstract = '0x10ED43C718714eb63d5aA57B78B54704E256024E'
let bnbConstract = '0xbb4cdb9cbd36b01bd1cbaebf2de08d9173bc095c'
let usdtConstract = '0x55d398326f99059ff775485246999027b3197955'
let ABIMap = {}

let bnbContractInstance
let pancakeContractInstance
let BN

function getRndInteger(min, max) {
  return Math.floor(Math.random() * (max - min) ) + min;
}

const delay = ms => new Promise((resolve, reject) => setTimeout(resolve, ms))
function formatter (thistime, fmt) {
  let $this = new Date(thistime)
  let o = {
    'M+': $this.getMonth() + 1,
    'd+': $this.getDate(),
    'h+': $this.getHours(),
    'm+': $this.getMinutes(),
    's+': $this.getSeconds(),
    'q+': Math.floor(($this.getMonth() + 3) / 3),
    'S': $this.getMilliseconds()
  }
  if (/(y+)/.test(fmt)) {
    fmt = fmt.replace(RegExp.$1, ($this.getFullYear() + '').substr(4 - RegExp.$1.length))
  }
  for (var k in o) {
    if (new RegExp('(' + k + ')').test(fmt)) {
      fmt = fmt.replace(RegExp.$1, (RegExp.$1.length === 1) ? (o[k]) : (('00' + o[k]).substr(('' + o[k]).length)))
    }
  }
  return fmt
}

async function startGame(walletAddress) {

    let startPlay = that.accounts[walletAddress].startPlay
    if(!startPlay){
      return
    }

    that.addLog("开始游戏")

}

async function getABI(dogAddr, bscscanApiKey) {
  if(ABIMap[dogAddr] || ABIMap[dogAddr] == '') {
    return ABIMap[dogAddr]
  }

  let res = await axios({
    method: 'get',
    url: `https://api.bscscan.com/api?module=contract&action=getabi&address=${dogAddr}&apikey=${bscscanApiKey}`,
  })

  ABIMap[dogAddr] = res.data.result
  return res.data.result
}

async function getTokenInfo(dogAddr, bscscanApiKey) {
  let res = await axios({
    method: 'get',
    url: `https://api.bscscan.com/api?module=token&action=tokeninfo&contractaddress=${dogAddr}&apikey=${bscscanApiKey}`,
  })

  return res.data.result
}

async function approveToken(tokenContractInstance, tokenConstract , walletAddress, privateKey) {
  const amount = await tokenContractInstance.methods.allowance(
    walletAddress,
    pancakeConstract
  ).call({from:walletAddress});
  let approveMax = 10000000000000000000000000000000
  //已经授权过
  if (amount >= approveMax) {
    console.log("已授权")
    return true
  }
  // *** BigNumber(approveMax) 是为了让数足够大，最大10000000000000000000000000000000，直接用web3 BN数太小，直接用BigNumber pancak方法在生产环境会报错，所以现在只能先转BigNumber再给BN
  let approveMaxStr = BigNumber(approveMax)
  let maxBN = web3.utils.toBN(approveMaxStr)
  // 重新调用授权
  const tx = await tokenContractInstance.methods.approve(
    pancakeConstract,
    maxBN
  ).encodeABI();

  var sign
  sign = await web3.eth.accounts.signTransaction({
    gas: 500000,
    to: tokenConstract,
    data: tx,
    value: 0
  }, privateKey);

  try{
    let result = await web3.eth.sendSignedTransaction(sign.rawTransaction);
    console.log("approveToken result = ", result)
  }catch (e) {
    console.error("error message = ", e)
    return false
  }

  console.log("授权成功")
  return true
}

async function pancakeSwapeExchange(walletAddress, privateKey, tokenConstract, dir, amount) {
  let examount = 0
  let now = new Date()
  let deadLine = now.getTime() + 30 * 1000

  let addresses =[tokenConstract, bnbConstract, usdtConstract]

  if(dir == "buy") {
    examount = amount * Math.pow(10, 18)
    addresses =[usdtConstract, bnbConstract, tokenConstract]
  } else {
    examount = amount
  }

  let functionEncode = await pancakeContractInstance.methods.swapExactTokensForTokens(web3.utils.numberToHex(examount), 1, addresses, walletAddress, deadLine).encodeABI()
  that.addLog("functionEncode完成")
  var sign

  try{

    sign = await web3.eth.accounts.signTransaction({
      gas: 360000,
      to: pancakeConstract,
      data: functionEncode,
      value: 0
    }, privateKey);

  }catch (e) {
    that.addLog("可能私钥有误请检查")
    console.error(e)
    return false
  }

  try{
    let result = await web3.eth.sendSignedTransaction(sign.rawTransaction);
    console.log(result)
    that.addLog("自动交易成功")
    return true
  }catch (e) {
    console.error("自动交易报错 error message = ", e)
    that.addLog("自动交易报错 可能没有足够手续费或者私钥错误请检查 自动交易强制停止")
    return false
  }


}

function formattingTime(count) {
  var time = '';
  var second = parseInt(count % 60);
  var minute = parseInt(parseInt(count) / 60) % 60;

  time = second + "秒";
  if (minute >= 1 ) {
    time = minute + "分" +second + "秒";
  }
  var hour = parseInt( parseInt(count / 60) / 60 ) % 24;
  if ( hour >= 1 ) {
    time = hour + "小时" + minute + "分" + second + "秒";
  }
  var day = parseInt( parseInt( parseInt(count / 60) / 60 ) / 24 );
  if ( day >= 1 ) {
    time = day + "天" + hour + "小时" + minute + "分" + second + "秒";
  }
  return time;
}


export default {
  data() {
    return {
      logStr: '',
      inputAddr: '',
      inputPri: '',
      dogeAddr: '',
      inputAmount: 0,
    }
  },

  computed: {
  },


  async mounted() {

    that = this
    this.accounts = {}
    const Web3 = require('web3');

    web3 = new Web3(mainetRPCUrl);
    BN = web3.utils.BN;

    if(!web3) {
      console.error("初始化web3出错 web3 为空 主网RPC地址 = " + mainetRPCUrl )
      that.addLog("初始化web3出错 web3 为空 主网RPC地址 = " + mainetRPCUrl)
      return
    }

    const pancakeAbi = [{"inputs":[{"internalType":"address","name":"_factory","type":"address"},{"internalType":"address","name":"_WETH","type":"address"}],"stateMutability":"nonpayable","type":"constructor"},{"inputs":[],"name":"WETH","outputs":[{"internalType":"address","name":"","type":"address"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"address","name":"tokenA","type":"address"},{"internalType":"address","name":"tokenB","type":"address"},{"internalType":"uint256","name":"amountADesired","type":"uint256"},{"internalType":"uint256","name":"amountBDesired","type":"uint256"},{"internalType":"uint256","name":"amountAMin","type":"uint256"},{"internalType":"uint256","name":"amountBMin","type":"uint256"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"addLiquidity","outputs":[{"internalType":"uint256","name":"amountA","type":"uint256"},{"internalType":"uint256","name":"amountB","type":"uint256"},{"internalType":"uint256","name":"liquidity","type":"uint256"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"address","name":"token","type":"address"},{"internalType":"uint256","name":"amountTokenDesired","type":"uint256"},{"internalType":"uint256","name":"amountTokenMin","type":"uint256"},{"internalType":"uint256","name":"amountETHMin","type":"uint256"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"addLiquidityETH","outputs":[{"internalType":"uint256","name":"amountToken","type":"uint256"},{"internalType":"uint256","name":"amountETH","type":"uint256"},{"internalType":"uint256","name":"liquidity","type":"uint256"}],"stateMutability":"payable","type":"function"},{"inputs":[],"name":"factory","outputs":[{"internalType":"address","name":"","type":"address"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountOut","type":"uint256"},{"internalType":"uint256","name":"reserveIn","type":"uint256"},{"internalType":"uint256","name":"reserveOut","type":"uint256"}],"name":"getAmountIn","outputs":[{"internalType":"uint256","name":"amountIn","type":"uint256"}],"stateMutability":"pure","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountIn","type":"uint256"},{"internalType":"uint256","name":"reserveIn","type":"uint256"},{"internalType":"uint256","name":"reserveOut","type":"uint256"}],"name":"getAmountOut","outputs":[{"internalType":"uint256","name":"amountOut","type":"uint256"}],"stateMutability":"pure","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountOut","type":"uint256"},{"internalType":"address[]","name":"path","type":"address[]"}],"name":"getAmountsIn","outputs":[{"internalType":"uint256[]","name":"amounts","type":"uint256[]"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountIn","type":"uint256"},{"internalType":"address[]","name":"path","type":"address[]"}],"name":"getAmountsOut","outputs":[{"internalType":"uint256[]","name":"amounts","type":"uint256[]"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountA","type":"uint256"},{"internalType":"uint256","name":"reserveA","type":"uint256"},{"internalType":"uint256","name":"reserveB","type":"uint256"}],"name":"quote","outputs":[{"internalType":"uint256","name":"amountB","type":"uint256"}],"stateMutability":"pure","type":"function"},{"inputs":[{"internalType":"address","name":"tokenA","type":"address"},{"internalType":"address","name":"tokenB","type":"address"},{"internalType":"uint256","name":"liquidity","type":"uint256"},{"internalType":"uint256","name":"amountAMin","type":"uint256"},{"internalType":"uint256","name":"amountBMin","type":"uint256"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"removeLiquidity","outputs":[{"internalType":"uint256","name":"amountA","type":"uint256"},{"internalType":"uint256","name":"amountB","type":"uint256"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"address","name":"token","type":"address"},{"internalType":"uint256","name":"liquidity","type":"uint256"},{"internalType":"uint256","name":"amountTokenMin","type":"uint256"},{"internalType":"uint256","name":"amountETHMin","type":"uint256"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"removeLiquidityETH","outputs":[{"internalType":"uint256","name":"amountToken","type":"uint256"},{"internalType":"uint256","name":"amountETH","type":"uint256"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"address","name":"token","type":"address"},{"internalType":"uint256","name":"liquidity","type":"uint256"},{"internalType":"uint256","name":"amountTokenMin","type":"uint256"},{"internalType":"uint256","name":"amountETHMin","type":"uint256"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"removeLiquidityETHSupportingFeeOnTransferTokens","outputs":[{"internalType":"uint256","name":"amountETH","type":"uint256"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"address","name":"token","type":"address"},{"internalType":"uint256","name":"liquidity","type":"uint256"},{"internalType":"uint256","name":"amountTokenMin","type":"uint256"},{"internalType":"uint256","name":"amountETHMin","type":"uint256"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"},{"internalType":"bool","name":"approveMax","type":"bool"},{"internalType":"uint8","name":"v","type":"uint8"},{"internalType":"bytes32","name":"r","type":"bytes32"},{"internalType":"bytes32","name":"s","type":"bytes32"}],"name":"removeLiquidityETHWithPermit","outputs":[{"internalType":"uint256","name":"amountToken","type":"uint256"},{"internalType":"uint256","name":"amountETH","type":"uint256"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"address","name":"token","type":"address"},{"internalType":"uint256","name":"liquidity","type":"uint256"},{"internalType":"uint256","name":"amountTokenMin","type":"uint256"},{"internalType":"uint256","name":"amountETHMin","type":"uint256"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"},{"internalType":"bool","name":"approveMax","type":"bool"},{"internalType":"uint8","name":"v","type":"uint8"},{"internalType":"bytes32","name":"r","type":"bytes32"},{"internalType":"bytes32","name":"s","type":"bytes32"}],"name":"removeLiquidityETHWithPermitSupportingFeeOnTransferTokens","outputs":[{"internalType":"uint256","name":"amountETH","type":"uint256"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"address","name":"tokenA","type":"address"},{"internalType":"address","name":"tokenB","type":"address"},{"internalType":"uint256","name":"liquidity","type":"uint256"},{"internalType":"uint256","name":"amountAMin","type":"uint256"},{"internalType":"uint256","name":"amountBMin","type":"uint256"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"},{"internalType":"bool","name":"approveMax","type":"bool"},{"internalType":"uint8","name":"v","type":"uint8"},{"internalType":"bytes32","name":"r","type":"bytes32"},{"internalType":"bytes32","name":"s","type":"bytes32"}],"name":"removeLiquidityWithPermit","outputs":[{"internalType":"uint256","name":"amountA","type":"uint256"},{"internalType":"uint256","name":"amountB","type":"uint256"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountOut","type":"uint256"},{"internalType":"address[]","name":"path","type":"address[]"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"swapETHForExactTokens","outputs":[{"internalType":"uint256[]","name":"amounts","type":"uint256[]"}],"stateMutability":"payable","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountOutMin","type":"uint256"},{"internalType":"address[]","name":"path","type":"address[]"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"swapExactETHForTokens","outputs":[{"internalType":"uint256[]","name":"amounts","type":"uint256[]"}],"stateMutability":"payable","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountOutMin","type":"uint256"},{"internalType":"address[]","name":"path","type":"address[]"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"swapExactETHForTokensSupportingFeeOnTransferTokens","outputs":[],"stateMutability":"payable","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountIn","type":"uint256"},{"internalType":"uint256","name":"amountOutMin","type":"uint256"},{"internalType":"address[]","name":"path","type":"address[]"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"swapExactTokensForETH","outputs":[{"internalType":"uint256[]","name":"amounts","type":"uint256[]"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountIn","type":"uint256"},{"internalType":"uint256","name":"amountOutMin","type":"uint256"},{"internalType":"address[]","name":"path","type":"address[]"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"swapExactTokensForETHSupportingFeeOnTransferTokens","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountIn","type":"uint256"},{"internalType":"uint256","name":"amountOutMin","type":"uint256"},{"internalType":"address[]","name":"path","type":"address[]"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"swapExactTokensForTokens","outputs":[{"internalType":"uint256[]","name":"amounts","type":"uint256[]"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountIn","type":"uint256"},{"internalType":"uint256","name":"amountOutMin","type":"uint256"},{"internalType":"address[]","name":"path","type":"address[]"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"swapExactTokensForTokensSupportingFeeOnTransferTokens","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountOut","type":"uint256"},{"internalType":"uint256","name":"amountInMax","type":"uint256"},{"internalType":"address[]","name":"path","type":"address[]"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"swapTokensForExactETH","outputs":[{"internalType":"uint256[]","name":"amounts","type":"uint256[]"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"uint256","name":"amountOut","type":"uint256"},{"internalType":"uint256","name":"amountInMax","type":"uint256"},{"internalType":"address[]","name":"path","type":"address[]"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"deadline","type":"uint256"}],"name":"swapTokensForExactTokens","outputs":[{"internalType":"uint256[]","name":"amounts","type":"uint256[]"}],"stateMutability":"nonpayable","type":"function"},{"stateMutability":"payable","type":"receive"}]
    const bnbAbi = [{"constant":true,"inputs":[],"name":"name","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"guy","type":"address"},{"name":"wad","type":"uint256"}],"name":"approve","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"totalSupply","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"src","type":"address"},{"name":"dst","type":"address"},{"name":"wad","type":"uint256"}],"name":"transferFrom","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"wad","type":"uint256"}],"name":"withdraw","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"decimals","outputs":[{"name":"","type":"uint8"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[{"name":"","type":"address"}],"name":"balanceOf","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"symbol","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"dst","type":"address"},{"name":"wad","type":"uint256"}],"name":"transfer","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[],"name":"deposit","outputs":[],"payable":true,"stateMutability":"payable","type":"function"},{"constant":true,"inputs":[{"name":"","type":"address"},{"name":"","type":"address"}],"name":"allowance","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"payable":true,"stateMutability":"payable","type":"fallback"},{"anonymous":false,"inputs":[{"indexed":true,"name":"src","type":"address"},{"indexed":true,"name":"guy","type":"address"},{"indexed":false,"name":"wad","type":"uint256"}],"name":"Approval","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"src","type":"address"},{"indexed":true,"name":"dst","type":"address"},{"indexed":false,"name":"wad","type":"uint256"}],"name":"Transfer","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"dst","type":"address"},{"indexed":false,"name":"wad","type":"uint256"}],"name":"Deposit","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"src","type":"address"},{"indexed":false,"name":"wad","type":"uint256"}],"name":"Withdrawal","type":"event"}]

    pancakeContractInstance = new web3.eth.Contract(pancakeAbi, pancakeConstract);
    if(!pancakeContractInstance) {
      console.error("初始化合约出错 pancakeContractInstance 为空")
      that.addLog("初始化合约出错 pancakeContractInstance 为空")
      return
    }

    bnbContractInstance = new web3.eth.Contract(bnbAbi, bnbConstract);
    if(!bnbContractInstance) {
      console.error("初始化合约出错 bnbContractInstance 为空")
      that.addLog("初始化合约出错 bnbContractInstance 为空")
      return
    }

    // 获取代币信息
    // let a = await getTokenInfo('0x90ac02497d8d6a5ee741f26946c68cdfa54eae01', partten.BSCSCAN_API_KEY)
    // console.log(a)
  },

  methods: {
    addLog: function(content) {
      let nowTime = formatter(new Date(), 'yyyy-MM-dd hh:mm:ss')
      logArr.push(nowTime +" "+ content)
      if(logArr.length > 50) {
        logArr.splice(0, 1)
      }
      this.logStr = logArr.join("\r\n")
    },
    playGame :function (walletAddress){
      let privateKey = this.accounts[walletAddress].privateKey
      if(!privateKey) {
        that.addLog("不能自动战斗，私钥不存在")
        return
      }

      that.accounts[walletAddress].startPlay = !that.accounts[walletAddress].startPlay
      this.$forceUpdate()
    },
    fastBuy: async function () {
      console.log(that.dogeAddr)
      let tokenABI = await getABI(that.dogeAddr, partten.BSCSCAN_API_KEY)
      console.log(tokenABI)
      if (!tokenABI) {
        return
      }

      let tokenInstanse = new web3.eth.Contract(JSON.parse(tokenABI), that.dogeAddr);
      if (!tokenInstanse) {
        return
      }

      let approved = await approveToken(tokenInstanse, that.dogeAddr, that.inputAddr, that.inputPri)
      if(!approved) {
        return
      }

      await pancakeSwapeExchange(that.inputAddr, that.inputPri, that.dogeAddr, 'buy', parseFloat(that.inputAmount))

    },
    fastSell: async function () {
      console.log(that.dogeAddr)
      let tokenABI = await getABI(that.dogeAddr, partten.BSCSCAN_API_KEY)
      console.log(tokenABI)
      if (!tokenABI) {
        return
      }

      let tokenInstanse = new web3.eth.Contract(JSON.parse(tokenABI), that.dogeAddr);
      if (!tokenInstanse) {
        return
      }

      let approved = await approveToken(tokenInstanse, that.dogeAddr, that.inputAddr, that.inputPri)
      if(!approved) {
        return
      }


      let balance = await tokenInstanse.methods.balanceOf(that.inputAddr).call();
      await pancakeSwapeExchange(that.inputAddr, that.inputPri, that.dogeAddr, 'sell', balance)

    }
  }
}
</script>
<style lang="scss" scoped>
</style>
