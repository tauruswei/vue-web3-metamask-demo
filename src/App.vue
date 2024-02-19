<template>
  <img alt="Vue logo" src="./assets/logo.png">
  <div v-show="!userAddress">
    <button @click="checkLogin">login</button>
  </div>
  <div>
    <span>{{userAddress}}</span>
  </div>
  <div>
    <button @click="sendEth"> 发送 eth </button>
  </div>
  <div>
    <button @click="sendERC20"> 发送 erc20 代币 </button>
  </div>
  <div>
    <button @click="depositERC20"> 使用 erc20 质押 </button>
  </div>
  <div>
    <button @click="depositErc20Permit"> 使用 erc20 permit 质押 </button>
  </div>
  <div>
    <button @click="deposit"> 质押 </button>
  </div>
  <div>
    <button @click="withdraw"> 提现 </button>
  </div>
  <div>
    <button @click="doPostRequest">发送 POST 请求</button>
  </div>
  <div>
    <button @click="getBNBPrice">获取 BNB 主网价格</button>
  </div>
  <div>
    <button @click="estimateGasFeeForContract">估算 erc20 转账的 10倍 Gas 费用</button>
  </div>
  <div>
    <button @click="estimateGasFeeForETHTransfer">估算 eth 转账的 5倍 Gas 费用</button>
  </div>
  <div>
    <button @click="estimateGasPrice">估算 当前网络的 gas 价格</button>
  </div>
  <div>
    <button @click="estimateGasLimitForETHTransfer">估算 eth 转账的 gas limit</button>
  </div>
  <div>
    <img :src="imageSrc" alt="Image" />
    <button @click="fetchImage">Fetch Image</button>
  </div>
</template>

<script>
import { ethers } from 'ethers'
import {
  contractAddress,
  contractABI,
  contractERC20Address,
  contractERC20ABI,
  contractERC20GameAddress,
  contractERC20GameABI,
  MyAppContractErc20PermitABI,MyAppContractErc20PermitAddress,
  ERC20PermitABI, ERC20PermitAddress
} from "@/components/contract";
import axios from 'axios';


export default {
  name: 'App',
  data(){
    return{
      userAddress:"",
      estimatedGasFee: null, // 用于存储估算的Gas费用
      imageSrc: null,
    }
  },
  methods:{
    async fetchImage() {
      try {
        const response = await axios.get('http://localhost:8090/aac/getImage', {
          responseType: 'blob', // important
        });
        const url = URL.createObjectURL(new Blob([response.data]));
        this.imageSrc = url;
      } catch (error) {
        console.error('Error fetching the image', error);
      }
    },
    async checkLogin(){
      const { ethereum } = window
      if(!ethereum){
        alert("请安装 metamask 钱包")
        return
      }
      console.log("您已安装metamask钱包")
      const account = await this.Provider().send("eth_requestAccounts", [])
      console.log(account)
      this.userAddress = account
    },
    Provider(){
      return new ethers.providers.Web3Provider(window.ethereum)
    },
    async sendEth(){
      const AMOUNT_TO_SEND = ethers.utils.parseEther("0.1"); // 发送0.1 ETH
      const RECEIVER_ADDRESS = "0xA574BC2942F0Bb48351872be4D481a37b762c49B"; // 这里替换为接收方的以太坊地址

      if (!this.userAddress) {
        alert("请先登录");
        return;
      }

      const provider = this.Provider();
      const signer = provider.getSigner(); // 获取签名者

      try {
        let tx = {
          to: RECEIVER_ADDRESS,
          value: AMOUNT_TO_SEND
        };
        let transactionResponse = await signer.sendTransaction(tx);

        console.log("Transaction hash:", transactionResponse.hash);

        let receipt = await transactionResponse.wait(); // 等待交易确认

        console.log("Transaction has been confirmed:", receipt);
      } catch (error) {
        console.error("Failed to send transaction:", error);
        alert("交易失败：" + error.message);
      }
    },
    Contract(){
      const provider = this.Provider()
      const signer = provider.getSigner()
      const contract = new ethers.Contract(contractAddress,contractABI,signer)
      return contract
    },

    ContractERC20(){
      const provider = this.Provider()
      const signer = provider.getSigner()
      const contract = new ethers.Contract(contractERC20Address,contractERC20ABI,signer)
      return contract
    },

    ContractERC20Game(){
      const provider = this.Provider()
      const signer = provider.getSigner()
      const contract = new ethers.Contract(contractERC20GameAddress,contractERC20GameABI,signer)
      return contract
    },

    async sendERC20(){
      const AMOUNT_TO_SEND = ethers.utils.parseUnits("10000", 18); // 发送10个代币，假设代币有18个小数位
      const RECEIVER_ADDRESS = "0x03cd3509b49b1a16acf64658ba25cd3f08b340e0"; // 这里替换为接收方的以太坊地址

      if (!this.userAddress) {
        alert("请先登录");
        return;
      }

      const contract = this.ContractERC20();

      try {
        // 调用 transfer 方法发送代币
        let tx = await contract.transfer(RECEIVER_ADDRESS, AMOUNT_TO_SEND);

        console.log("Transfer transaction hash:", tx.hash);

        let receipt = await tx.wait(); // 等待交易确认

        console.log("Transfer has been confirmed:", receipt);
      } catch (error) {
        console.error("Failed to send ERC20 tokens:", error);
        alert("发送代币失败：" + error.message);
      }
    },


    async depositErc20Permit(){
      const AMOUNT_TO_DEPOSIT = ethers.utils.parseUnits("1000", 18); // 发送10000个代币，假设代币有18个小数位

      if (!this.userAddress) {
        alert("请先登录");
        return;
      }

      try {

        const provider = this.Provider();
        const signer = provider.getSigner();
        let ownerAddress = await signer.getAddress(); // 确保使用await

        const MyAppContractErc20Permit = new ethers.Contract(MyAppContractErc20PermitAddress, MyAppContractErc20PermitABI, signer);
        const Erc20Permit = new ethers.Contract(ERC20PermitAddress, ERC20PermitABI, signer);

        const deadline = Math.floor(Date.now() / 1000) + 3600; // Permit有效期为1小时
        const nonce = await Erc20Permit.nonces(ownerAddress);

        const permitData = {
          owner: ownerAddress, // 确保这是一个有效的字符串地址
          spender: MyAppContractErc20Permit.address, // 使用.address获取字符串地址
          value: AMOUNT_TO_DEPOSIT.toString(),
          nonce: nonce.toString(),
          deadline
        };

        const domain = {
          name: await Erc20Permit.name(),
          version: '1',
          chainId: await signer.getChainId(),
          verifyingContract: Erc20Permit.address.toString() // 确保这是一个字符串
        };

        const types = {
          Permit: [
            { name: 'owner', type: 'address' },
            { name: 'spender', type: 'address' },
            { name: 'value', type: 'uint256' },
            { name: 'nonce', type: 'uint256' },
            { name: 'deadline', type: 'uint256' }
          ]
        };

        const signature = await signer._signTypedData(domain, types, permitData);

        const { v, r, s } = ethers.utils.splitSignature(signature);
        console.log("5 :");

        let tx = await MyAppContractErc20Permit.depositTokens(AMOUNT_TO_DEPOSIT, deadline, v, r, s);
        console.log("6 :");


        console.log("Deposit transaction hash:", tx.hash);

        let receipt = await tx.wait(); // 等待交易确认

        console.log("Deposit has been confirmed:", receipt);
      } catch (error) {
        console.error("Failed to deposit:", error);
        alert("存款失败：" + error.message);
      }
    },

    async deposit(){
      const AMOUNT_TO_DEPOSIT = ethers.utils.parseEther("0.01"); // 存款0.1 ETH，你可以根据需要调整这个值

      if (!this.userAddress) {
        alert("请先登录");
        return;
      }

      const contract = this.Contract();

      try {
        // 调用 deposit 方法并发送ETH
        let tx = await contract.deposit({ value: AMOUNT_TO_DEPOSIT });

        console.log("Deposit transaction hash:", tx.hash);

        let receipt = await tx.wait(); // 等待交易确认

        console.log("Deposit has been confirmed:", receipt);
      } catch (error) {
        console.error("Failed to deposit:", error);
        alert("存款失败：" + error.message);
      }
    },


    async depositERC20(){
      const AMOUNT_TO_DEPOSIT = ethers.utils.parseUnits("1000", 18); // 发送10个代币，假设代币有18个小数位
      const POOL_INDEX = 0.0;


      if (!this.userAddress) {
        alert("请先登录");
        return;
      }

      const contract = this.ContractERC20Game();

      try {
        // 调用 deposit 方法并发送ETH
        let tx = await contract.deposit(POOL_INDEX, AMOUNT_TO_DEPOSIT);

        console.log("Deposit transaction hash:", tx.hash);

        let receipt = await tx.wait(); // 等待交易确认

        console.log("Deposit has been confirmed:", receipt);
      } catch (error) {
        console.error("Failed to deposit:", error);
        alert("存款失败：" + error.message);
      }
    },

    async withdraw() {
      if (!this.userAddress) {
        alert("请先登录");
        return;
      }

      const contract = this.Contract();

      try {
        // 调用 withdraw 方法
        let tx = await contract.withdraw();

        console.log("Withdraw transaction hash:", tx.hash);

        let receipt = await tx.wait(); // 等待交易确认

        console.log("Withdraw has been confirmed:", receipt);
      } catch (error) {
        console.error("Failed to withdraw:", error);
        alert("提款失败：" + error.message);
      }
    },
    async doPostRequest() {
      const url = 'http://13.230.210.183:5432/asset/queryUserAssets';
      const data = {
        "userId": "2",
        "assetType":"0"
      };

      try {
        const response = await fetch(url, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify(data)
        });

        if(response.ok) {
          const result = await response.json();
          console.log('请求成功:', result);
          // 在这里处理请求成功后的结果
        } else {
          console.error('请求错误:', response.statusText);
        }
      } catch (error) {
        console.error('请求错误:', error);
        // 在这里处理请求错误
      }
    },
    async getBNBPrice() {
      try {
        const response = await fetch('https://api.coingecko.com/api/v3/simple/price?ids=binancecoin&vs_currencies=usd');
        if (!response.ok) {
          throw new Error('Network response was not ok ' + response.statusText);
        }
        const data = await response.json();
        const bnbPrice = data.binancecoin.usd;
        console.log('BNB Price: $', bnbPrice);
        // 你可以将价格存储在你的组件的 data 属性中，或者在 UI 中显示它
        this.bnbPrice = bnbPrice;  // 假设你有一个名为 bnbPrice 的 data 属性
      } catch (error) {
        console.error('There has been a problem with your fetch operation:', error);
      }
    },
    async estimateGasFeeForContract(){
      const AMOUNT_TO_SEND = ethers.utils.parseUnits("10000", 18);  // 要发送的代币数量
      const RECEIVER_ADDRESS = "0x03cd3509b49b1a16acf64658ba25cd3f08b340e0";  // 接收者的地址

      if (!this.userAddress) {
        alert("请先登录");
        return;
      }

      const provider = this.Provider();
      const contract = new ethers.Contract(contractERC20Address, contractERC20ABI, provider);
      const data = contract.interface.encodeFunctionData("transfer", [RECEIVER_ADDRESS, AMOUNT_TO_SEND]);

      try {
        const gasPriceResult = await provider.getGasPrice();  // 获取当前Gas价格
        const gasPrice = gasPriceResult.mul(10);  // Multiply the gas price by 10

        const gasLimit = await provider.estimateGas({
          to: contractERC20Address,
          from: this.userAddress[0],
          data: data
        });  // 估算Gas限制

        const fee = gasPrice.mul(gasLimit);
        this.estimatedGasFee = ethers.utils.formatEther(fee.toString());  // 将费用转换为以太，并存储在data属性中

        console.log('Estimated Gas Fee: ', this.estimatedGasFee, ' ETH');
      } catch (error) {
        console.error('Error estimating gas fee: ', error);
        alert('估算Gas费用时出错: ' + error.message);
      }
    },
    async estimateGasFeeForETHTransfer(){
      const AMOUNT_TO_SEND = ethers.utils.parseUnits("1", 18);  // 要发送的ETH数量
      const RECEIVER_ADDRESS = "0x03cd3509b49b1a16acf64658ba25cd3f08b340e0";  // 接收者的地址

      if (!this.userAddress) {
        alert("请先登录");
        return;
      }

      const provider = this.Provider();

      try {
        const gasPriceResult = await provider.getGasPrice();  // 获取当前Gas价格
        const gasPrice = gasPriceResult.mul(5);  // Multiply the gas price by 10

        const gasLimit = await provider.estimateGas({
          to: RECEIVER_ADDRESS,
          from: this.userAddress[0],
          value: AMOUNT_TO_SEND
        });  // 估算Gas限制

        const fee = gasPrice.mul(gasLimit);
        this.estimatedGasFee = ethers.utils.formatEther(fee.toString());  // 将费用转换为以太，并存储在data属性中
        alert('Estimated Gas Fee for ETH transfer: '+this.estimatedGasFee+'ETH');
        console.log('Estimated Gas Fee for ETH transfer: ', this.estimatedGasFee, ' ETH');
      } catch (error) {
        console.error('Error estimating gas fee for ETH transfer: ', error);
        alert('估算ETH转账Gas费用时出错: ' + error.message);
      }
    },
    async estimateGasPrice(){

      if (!this.userAddress) {
        alert("请先登录");
        return;
      }

      const provider = this.Provider();

      try {
        const gasPriceResult = await provider.getGasPrice();  // 获取当前Gas价格
        alert('Estimated gas price: '+gasPriceResult+'wei');
      } catch (error) {
        console.error('Error estimating gas price: ', error);
        alert('估算 gas price 时出错: ' + error.message);
      }
    },
    async estimateGasLimitForETHTransfer(){
      const AMOUNT_TO_SEND = ethers.utils.parseUnits("1", 18);  // 要发送的ETH数量
      const RECEIVER_ADDRESS = "0x03cd3509b49b1a16acf64658ba25cd3f08b340e0";  // 接收者的地址

      if (!this.userAddress) {
        alert("请先登录");
        return;
      }

      const provider = this.Provider();

      try {

        const gasLimit = await provider.estimateGas({
          to: RECEIVER_ADDRESS,
          from: this.userAddress[0],
          value: AMOUNT_TO_SEND
        });  // 估算Gas限制

        alert('Estimated Gas limit for ETH transfer: '+gasLimit);
      } catch (error) {
        console.error('Error estimating gas limit for ETH transfer: ', error);
        alert('估算ETH转账Gas limit 时出错: ' + error.message);
      }
    }
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
</style>
