---
slug: uniswapv2
title: 从零开始UniswapV2
date: 2023-12-04
authors: [Zach]
math: true
categories:
  - Defi
---

## Part-1 流动性
> UniswapV2相较于V1有了较大变动，也是目前最流行的dex版本，很多协议都是从UniswapV2 fork而来，在本系列的文章中，将使用Foundry作为合约的测试框架，使用solmate而非OpenZeppelin作为底层协议如ERC20的实现，将uniswap的核心框架代码从零到一进行复现。


UniswapV2的代码库分为core和periphery两部分，core包括：

- UniswapV2ERC20：用于lp代币的ERC20拓展
- UniswapV2Factory：工厂合约，用于管理交易对
- UniswapV2Pair：核心的交易对合约

periphery主要包括的合约是UniswapV2Router和UniswapV2Library，是重要的辅助交易合约

### 添加流动性

先从核心的添加流动性入手，uniswap在设计上与v1类似，交易者，lp，用户仍然是协议的核心参与者。与v1相比，v2的代码设计有所差异，lp注入流动性分为两部分，底层的实现是UniswapV2Pair合约，上层的入口在UniswapV2Router合约，在此我们先关注底层的实现。

添加流动性：即LP向交易对合约内按照一定比例转入两个底层资产，交易对为其按照投入的资产铸造出相应数量的流动性代币的过程。

作为UniswapV2Pair底层，这里需要实现的功能就是：计算用户投入多少底层资产，计算出相应额度的lp代币再mint给用户
```solidity
function mint() public {
   uint256 balance0 = IERC20(token0).balanceOf(address(this));
   uint256 balance1 = IERC20(token1).balanceOf(address(this));
   uint256 amount0 = balance0 - reserve0;
   uint256 amount1 = balance1 - reserve1;
	 // 计算出本次用户投入的资产数量amount0和amount1
   uint256 liquidity;
	 // 区分是否是首次mint，流动性代币的计算方式不同
   if (totalSupply == 0) {
      liquidity = ???
      _mint(address(0), MINIMUM_LIQUIDITY);
   } else {
      liquidity = ???
   }

   if (liquidity <= 0) revert InsufficientLiquidityMinted();
	 // mint流动性代币
   _mint(msg.sender, liquidity);
	 // 更新交易对内储备量缓存
   _update(balance0, balance1);

   emit Mint(msg.sender, amount0, amount1);
}
```
从代码中可以看出，这里的交易对会通过reserve0和reserve1两个变量来缓存当前交易对中两个token的数量，而不是直接使用balanceOf来计数（注：这里也是为了合约安全起见，避免被外部操控）

在调用mint 方法前，用户应该按照预期向当前合约转入token0和token1，这里再计算当前合约内的token余额balance0 和balance1 ，减去之前的缓存，得到的amount0和amount1 就是本次用户转入的token数量

在计算应该铸造出多少个lp代币时会区分totalSupply是否为0，即当前是否是初次提供流动性，假设当前交易对内的token情况如下所示：
|  token0   | token1  | liquidity  |
|  ----  | ----  |----  |
| reserve0  | reserve1 |totalSupply |
| amount0  | amount1 |totalSupply+lp |

按照固定的比例，本次待铸造的lp代币数目来源有两个，用户投入的两个token都可以作为基准来计算本次铸造的流动性代币

- amount0/totalSupply*reserve0
- amount1/totalSupply*reserve1

在实际开发中，UniswapV2的规则是选择两者中较小的那个，按照规定，用户提供的流动性是严格按照比例来的，两个值应该相等，但是若用户提供不平衡的流动性，这两个值就存在差异，如果协议按照大的来计算lp，那么相当于是对这种方式的鼓励，因此选择较小的流动性代币数目作为对用户的惩罚

回到totalSupply=0的条件分支，无法按照统一的比例计算lp代币数量，uniswapV2选择的是计算amount0*amount1的根号值，并且会统一减去MINIMUM_LIQUIDITY（1000）。

- 假设某lp初次投入token0和token1各1 wei，如果不减去MINIMUM_LIQUIDITY，则会mint出1 枚lp代币，然后再直接转入1000枚token0和token1，则此时交易对内有1000*10^18+1个token0和token1，但是只有1 wei的lp，那么对于后来的lp来说，即使只想提供最小单位的 1 wei 流动性，也要付出 2000 ether 的 token，[解释参考](https://learnblockchain.cn/article/3004#首次铸币的漏洞)
- 若统一减去MINIMUM_LIQUIDITY，则存在1000的流动性下限，用户可以不通过mint直接转入token，如果重新执行攻击流程，流动性单价最大值为(1001+2000×10^18)1001≈2×10^18，相较于前面已经降低很多，但是这里损失了首次流动性提供者的利益

整理后的代码如下：
```solidity
if (totalSupply == 0) {
   liquidity = Math.sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
   _mint(address(0), MINIMUM_LIQUIDITY);
} else {
   liquidity = Math.min(
      (amount0 * totalSupply) / _reserve0,
      (amount1 * totalSupply) / _reserve1
   );
}
```

配套的测试代码：
```solidity
	 function testInitialMint() public {
        vm.startPrank(lp);
        token0.transfer(address(pair),1 ether);
        token1.transfer(address(pair),1 ether);
        
        pair.mint();
        uint256 lpToken = pair.balanceOf(lp);
        assertEq(lpToken, 1e18-1000);
    }

    function testExistLiquidity() public {
        testInitialMint();
        vm.startPrank(lp);
        token0.transfer(address(pair),1 ether);
        token1.transfer(address(pair),1 ether);
        
        pair.mint();
        uint256 lpToken = pair.balanceOf(lp);
        assertEq(lpToken, 2e18-1000);
    }

    function testUnbalancedLiquidity() public {
        testInitialMint();
        vm.startPrank(lp);
        token0.transfer(address(pair),2 ether);
        token1.transfer(address(pair),1 ether);
        
        pair.mint();
        uint256 lpToken = pair.balanceOf(lp);
        assertEq(lpToken, 2e18-1000);
    }
```

### 移除流动性
从添加流动性的流程可以看出整体的流程是：用户转入底层资产token0和token1，mint出对应数目的lp代币

那么移除流动性就是逆向的过程，移除的前提是用户拥有lp代币，这里的lp代币就是用户提供流动性的凭证，具体的代码如下：

- 首先根据用户持有的lp数目，重新计算出他应得的amount0和amount1
- 将用户的全部lp代币销毁（可以看到这里暂不支持移除部分流动性）
- 将计算出的相应数目的token0和token1转移回用户
- 更新交易对内的资金储备量

```solidity
function burn() external{
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        uint256 liquidity = balanceOf[msg.sender];
        // 计算用户的流动性占比的token数量
        uint256 amount0 = liquidity * balance0 / totalSupply;
        uint256 amount1 = liquidity * balance1 / totalSupply;
        if (amount0 <=0 || amount1 <=0) revert InsufficientLiquidityBurned();
        // 流动性代币burn
        _burn(msg.sender, liquidity);
        // 转移token回给用户
        _safeTransfer(token0, msg.sender, amount0);
        _safeTransfer(token1, msg.sender, amount1);
        // 更新当前储备金
        balance0 = IERC20(token0).balanceOf(address(this));
        balance1 = IERC20(token1).balanceOf(address(this)); 
        _update(balance0, balance1);
        emit Burn(msg.sender, amount0, amount1);
    }
```

测试代码如下：
```solidity
    function testBurn() public{
        testInitialMint();
        vm.startPrank(lp);
        pair.burn();
        assertEq(pair.balanceOf(lp), 0);
        assertEq(token0.balanceOf(lp), 10 ether-1000);
        assertEq(token1.balanceOf(lp), 10 ether-1000);
    }

    function testUnbalancedBurn() public {
        testInitialMint();
        vm.startPrank(lp);
        token0.transfer(address(pair),2 ether);
        token1.transfer(address(pair),1 ether);
        
        pair.mint();
        uint256 lpToken = pair.balanceOf(lp);
        assertEq(lpToken, 2e18-1000);

        pair.burn();
        assertEq(pair.balanceOf(lp), 0);
        assertEq(token0.balanceOf(lp), 10 ether-1500);
        assertEq(token1.balanceOf(lp), 10 ether-1000);
    }
```

## Part-2 预言机

### 什么是价格

Uniswap作为链上的去中心化交易所，承载着价值发现的功能，即用户或其他链上合约可以通过Uniswap来获取代币的价格，Uniswap在这其中承担链上预言机的功能。

假设当前交易池中有1 Ether和2000 USDC，那么以太币的价格就是2000/1=2000 USDC，反之就是USDC的价格，因此价格就是一个比率，由于智能合约尚不支持小数，所以在Uniswap的代码中拓展了新的数据类型来存储价格，关于新的数据类型后面会专门做介绍。

### TWAP价格机制

然而仅仅使用瞬时的代币数目之比作为价格是不安全的，存在人为操纵价格预言机的风险，由于Uniswap提供了闪电贷功能，因此在某个闪电贷交易的瞬间，交易对内的代币余额会产生剧烈波动。在UniswapV2中为了解决这个问题，采用了TWAP（Time Weighted Average Price）即时间加权的价格预言机机制。

具体工作原理如下：

- 假设过去一天，资产在前20个小时的价格为20$，最近4小时的价格为10$，那么TWAP=($20*20+10$*4)/24 = 18.33$
- 假设过去一天，资产在第一个小时的价格为10$，最近23小时的价格为15$，那么TWAP=($10*1+15$*23)/24 = 14.79$

总结下来，TWAP的公式如下，这里的T是时间段，P是对应时间段的价格


![image](https://cloudflare-ipfs.com/ipfs/QmTshmD8bGz9yHHMXmuEu2F3qfnNhiFusi5iAGCMcM5CRd)


在UniswapV2的合约中，只会记录分子部分，即记录每个时间段乘以单价的求和，而分母部分则需要使用方自行维护，交易对内有两个代币，所有有两个值来记录

通常我们只需关心某一时间区间内的代币价格，这是TWAP公式的历史价格公式：


![image](https://cloudflare-ipfs.com/ipfs/QmRe6ePXNpo8ZXwq86Q2inW71VSviEV8tfAotHEg4K3hXd)


假设我们的计价从T4开始，那么实际的计价公式应该如下：


![image](https://cloudflare-ipfs.com/ipfs/QmXiFQm61ivq75DWk3cy1ne3BjLF2Sb7rTWwXB3pjQ3yjN)


前面已经提到，在合约内有一个变量会追踪分子的求和值，以token0的追踪计价变量price0Cumulativelast为例：


![image](https://cloudflare-ipfs.com/ipfs/QmYXEUiiw8WnmMETJbDZQKk8GMWRcRNRz7EtWL8b9CmHwY)


这个变量是记录了历史以来所有时间段的求和，那么我们只需要从T4开始的部分即可，计算方式也很简单，在T3时间点我们获取一个price0Cumulativelast变量的快照，在最新即T6时间点再获取一次，两次的差值即是T4-最新时间段内token0的计价和


![image](https://cloudflare-ipfs.com/ipfs/Qmc1xJsvoNSEfiGL3ddqe6dFWjkDCkm2KWB2rv53gpQSBB)


我们自己也维护了最近的窗口持续时间和，即：T4+T5+T6

那么这段时间内的TWAP价格即可计算得出：(price0Cumulativelast-UpToTime3)/(T4+T5+T6)

### UniswapV2的实现
具体到Uniswap的实现中，对于每个交易对都维护了两个变量price0Cumulativelast和price1Cumulativelast，在之前提到的`_update` 方法中进行求和，具体的代码如下：

- 首先获取当前的区块时间戳`blockTimestamp`
- 通过与blockTimestampLast相减，计算出距离上次更新过去了多少时间`timeElapsed`
- 只有在`timeElapsed` >0时，即进入下一个区块时，才会累加价格*时间段
- 注意这里的价格计算用到了`UQ112x112` 的特殊格式，了解它是为了记录小数即可，后面会专门讲解这里的优化
- 在处理完累加后，才会更新`blockTimestampLast` 到最新的区块时间戳

```solidity
function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
    ...
    uint32 blockTimestamp = uint32(block.timestamp % 2**32);
    uint32 timeElapsed = blockTimestamp - blockTimestampLast;
    if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
      // * never overflows, and + overflow is desired
      price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
      price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
    }
    blockTimestampLast = blockTimestamp;
    ...
  }
```

### TWAP的潜在问题

- 基于时间加权的价格计算，会提高攻击者的操纵成本，因为需要连续控制多个区块，这样的攻击成本是很高的。
当价格产生剧烈波动时，由于有时间作为加权因素，预言机的价格无法在较短时间内反映出价格的波动，反而提供出过时的价格，尤其是在市场发生剧烈动荡时，这样的情况会导致Uniswap中的价格与外部市场产生较大的差异。

- 使用TWAP预言机仍然依赖链下的定时触发，存在维护成本与中心化问题。