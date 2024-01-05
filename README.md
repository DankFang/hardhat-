# hardhat-entrap-reason
### 记录了我使用hardhat框架以来踩过的坑，从11月份开始记录，之前的没有记录



1. 合约实例.address已不能获取到合约地址，得改用合约实例.target
2. 在测试代码中，若部署的合约涉及到引用库合约，则先部署库合约，然后导入到部署主合约上
   例子如下：
   ```
   const CSHARE = await hre.ethers.getContractFactory("Contract", {
       libraries: {
           Libname: lib实例.target
       }
   });
   ```
3. 另外，在没有配置hardhat运行环境，即hre时，调用如ethers.getContractFactory时则要加上hre. 若不加上，则会导致找不到
4. loadFixture的使用：
   第一次调用 loadFixture 时，将执行fixture。但第二次时，loadFixture 不会再次执行fixture，而是将网络状态重置为执行fixture后的状态。
   可代替掉beforeEach，在每一次测试时不用再次部署合约。
   另外如果想重新要一个deployment，则可以不用loadFixture，可以手动部署，例子如下：
   ```
   it("Should fail if the unlockTime is not in the future", async function() {
       // We don't use the fixture here because we want a different deployment
       const latestTime = await time.latest();
       const Lock = await ethers.getContractFactory("Lock");
       await expect(Lock.deploy(latestTime, { value:1 })).to.be.revertedWith(
           "Unlock time should be in the future"
       );
   });
   ```
5. 在hardhat test脚本文件里，若调用合约的函数，函数有返回值且是view修饰符，则可以直接定义一个变量来接收返回值，例子如下：
   ```
   合约：function owner() public view returns (address) {
       return _owner;
   }

   脚本：const owner = await myNFT.owner();
   ```
   如果不是view修饰，则不能直接定义一个变量接收返回值。
   函数： function mintNFT(address recipient, string memory tokenURI)
        public returns (uint256)
    {
        require(msg.sender == owner(), "Only owner is allowed to mint");
        uint newItemId = ++_counter;
        ERC721._mint(recipient, newItemId);
        ERC721URIStorage._setTokenURI(newItemId, tokenURI);

        return newItemId;
      }
      这样的函数则按上述方法调用会报错，网上解释：You can not directly receive a return value coming from a function that you are sending a transaction to off-chain. You can only do so on-chain - i.e. when one SC function invokes another SC function.

      解决办法：再写个合约，自定义变量来接收该函数返回值，即在本合约里导如函数所在合约，创建实例并且调用函数
... (continues with more items)

6. 如果函数名相同，但参数不同，调用时，需声明完整的函数签名。例子如下：
   ```
   如果遇到两个safeTransferFrom函数，但参数类型或个数不同
   错误方式contract.safeTransferFrom(addr1, addr2, 1);
   正确方式：contract["safeTransferFrom(address,address,uint256)"](addr1, addr2, 1);
   ```
   ```
   await contract['functionName(uint256,uint256)'](arg1, arg2);
   ```
7. 在Hardhat中的test部署可升级的合约，需要使用 @openzeppelin/hardhat-upgrades 插件。
   部署方式如下：
   ```
      const { AddressZero } = ethers.constants;
         let deployResult = await deploy('', {
         contract: 'ParamcontractName',
         from: 从哪个地址部署,
         gasLimit: 30000000,
         args: [],
         proxy: {
            proxyContract: 'OpenZeppelinTransparentProxy',
            execute: {
               init: {
                  methodName: 'initialize',
                  args: [1, AddressZero]
               }
            }
         },
            log: true,
});
      const contract = await ethers.getContract('合约名字')
   ```
   ```
   部署到区块链上
   const { deployProxy } = require('@openzeppelin/hardhat-upgrades');

   async function main() {
       const Contract = await ethers.getContractFactory("MyContract");
       const instance = await deployProxy(Contract, [arg1, arg2], { initializer: 'initialize' });
       console.log("Deployed to:", instance.address);
   }
   ```
8. 处理测试超时问题：
   默认的测试超时时间是20000ms。如果需要更长时间，可以在 it 函数中设置 this.timeout()。
   例如，设置超时为60000ms：
   ```
   it("test case", async function() {
       this.timeout(60000);
       // test code
   });

   it("", async function () {

   }).timeout(1000000)

   在hardhat-config.js文件里加上限定
   mocha: {
   timeout: 10000000
   },
   ```
9. hardhat测试时：打包交易的calldata：
   ```
   以下是打包erc20转账的演示
   //先把打包函数搞出来
   const abiCoder = new ethers.AbiCoder();
   // 将function转换为buffer
   const functionBytes = toUtf8Bytes('transfer(address,uint256)');
   // 打包参数
   const data = abiCoder.encode(['address', 'uint256'], [user4.address, ethers.parseEther("10")]);
   // 获取函数选择器 
   const functionSelector = dataSlice(keccak256(functionBytes), 0, 4);
   // 拼接calldata
   const finalData = functionSelector + data.substring(2);


   最终传入finaldata即可调通
   ```

10. Git指令：指定克隆某分支git clone --branch <branchname> <remote-repo-url>或者git clone -b <branchname> <remote-repo-url>
11. Git rebase 命令：合并分支或者合并之前提交的记录
