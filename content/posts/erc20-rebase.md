---
slug: erc20-rebase
title: ERC20 Rebase的设计与实现
date: 2024-04-09
authors: [Zach]
math: true
categories:
  - system design
---

## 1 基础框架

### 背景

ERC20 Rebase机制是在ERC20协议基础之上衍生的，用来对代币持有者做激励分红，这里将以[ethereum-credit-guild](https://github.com/volt-protocol/ethereum-credit-guild)项目中设计的`ERC20RebaseDistributor`合约为基础，讲解Rebase机制的设计与实现思路。


在ecg中，`ERC20RebaseDistributor`合约是作为底层的creditToken，类比到其他借贷协议，creditToken等同于compound中的cToken，是lender的质押凭证，也是生息资产，针对不同的抵押物都有一个绑定的creditToken，质押者可以通过持有creditToken来累积收益。

### 功能分析

在`ERC20RebaseDistributor`合约中并不是默认所有holder都持有生息资产，而是将参与rebase和非rebase的分开，显然分红只针对参与rebase的holder展开，如图所示，ERC20Rebase由rebasingSupply和nonRebasingSupply两部分构成
![erc20 rebase supply关系](https://pic.imgdb.cn/item/6678b74fd9c307b7e9fb787d.png)

这里我们总结出下面的恒等式：
- `totalSupply() == nonRebasingSupply() + rebasingSupply()`
- `sum of balanceOf(x) == totalSupply()`

接下来，再分析分红机制，在ecg协议中，存在一个合约来汇总单个creditToken的收益，并按照比例把一部分token转为对参与rebase的holder分红，这里的分红逻辑也很直接，每个参与rebase的用户根据当前余额来瓜分分红额度即可。

至此，我们的ERC20Rebase基础设计已经明确，需要提供enter/exit rebase方法和分红distribute方法，具体的代码如下所示（注：这里只实现关键逻辑，仍有缺漏）：

- 定义 `rebasingAccounts(array)`  `rebasingAccount(mapping)` 来跟踪参与rebase的地址
- 定义 `rebasingSupply` 来记录所有参与rebase的供应量
- `enterRebase` 函数：将该地址标记为参与rebase，累加rebase供应量
- `exitRebase` 函数：取消参与rebase的标记，扣减rebase供应量
- `distribute` 函数：先将分红数额销毁，再按照比例mint给所有参与rebase的地址

```solidity
    function enterRebase() external {
        require(!rebasingAccount[msg.sender], "SimpleERC20Rebase: already rebasing");
        uint256 balance = balanceOf(msg.sender);
        rebasingAccount[msg.sender] = true;
        rebasingSupply += balance;
        rebasingAccounts.push(msg.sender);
    }

    function exitRebase()  external {
        require(rebasingAccount[msg.sender], "SimpleERC20Rebase: not rebasing");
        uint256 balance = balanceOf(msg.sender);
        rebasingAccount[msg.sender] = false;
        rebasingSupply -= balance;
        for (uint256 i = 0; i < rebasingAccounts.length; i++) {
            if (rebasingAccounts[i] == msg.sender) {
                rebasingAccounts[i] = rebasingAccounts[rebasingAccounts.length - 1];
                rebasingAccounts.pop();
                break;
            }
        }
    }

    function distribute(uint256 amount) external {
        require(balanceOf(msg.sender)>=amount, "SimpleERC20Rebase: not enough");
        _burn(msg.sender, amount);
        for (uint256 i = 0; i < rebasingAccounts.length; i++) {
            uint256 delta = amount * balanceOf(rebasingAccounts[i]) / rebasingSupply;
            _mint(rebasingAccounts[i], delta);
        }
        rebasingSupply += amount;
    }

    function mint(address user, uint256 amount) external {
        return _mint(user, amount);
    }
```

最基础的rebase分红机制已经实现了，回顾上面的代码，存在非常关键的问题：如果参与rebase的地址很多，每次分红都会有大量的mint操作，成本过高，系统无法拓展

### 完整代码

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.13;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract SimpleERC20Rebase is ERC20 {

    mapping(address => bool) internal rebasingAccount;
    address[] internal rebasingAccounts;
    uint256 public rebasingSupply;
    
    constructor(
        string memory _name,
        string memory _symbol
    ) ERC20(_name, _symbol) {}

    function nonRebasingSupply() public view returns (uint256) {
        return totalSupply() - rebasingSupply;
    }

    function enterRebase() external {
        require(!rebasingAccount[msg.sender], "SimpleERC20Rebase: already rebasing");
        uint256 balance = balanceOf(msg.sender);
        rebasingAccount[msg.sender] = true;
        rebasingSupply += balance;
        rebasingAccounts.push(msg.sender);
    }

    function exitRebase()  external {
        require(rebasingAccount[msg.sender], "SimpleERC20Rebase: not rebasing");
        uint256 balance = balanceOf(msg.sender);
        rebasingAccount[msg.sender] = false;
        rebasingSupply -= balance;
        for (uint256 i = 0; i < rebasingAccounts.length; i++) {
            if (rebasingAccounts[i] == msg.sender) {
                rebasingAccounts[i] = rebasingAccounts[rebasingAccounts.length - 1];
                rebasingAccounts.pop();
                break;
            }
        }
    }

    function distribute(uint256 amount) external {
        require(balanceOf(msg.sender)>=amount, "SimpleERC20Rebase: not enough");
        _burn(msg.sender, amount);
        for (uint256 i = 0; i < rebasingAccounts.length; i++) {
            uint256 delta = amount * balanceOf(rebasingAccounts[i]) / rebasingSupply;
            _mint(rebasingAccounts[i], delta);
        }
        rebasingSupply += amount;
    }

    function mint(address user, uint256 amount) external {
        return _mint(user, amount);
    }
}
```

## 2 Share与Balance

### 引出Share

我们需要针对上面提出的问题进行优化，通过观察分红机制可以看出，所有account对应的balance都是按比例增加，那么这里的循环更新是否有存在的必要呢？是否可以全局记录一个系数，每次调用distribute时只更新这个系数，要查询balance时，乘上这个系数即可

> 那么这里就拆分出两个概念，balance和share，并且衍生出sharePrice，对于所有进入rebase的用户都只记录share，并且有一个全局的sharePrice，查询用户balance时需要根据share和sharePrice计算出balance，sharePrice在分红时增加
> 

针对第一个版本，我们梳理一下需要修改的部分和积攒的问题：

- 需要记录每个参与rebase的地址的share
- share与balance如何映射
- sharePrice在每次分红时如何累加

具体的改动如下：

- 将mapping的value拓展为结构体，记录share
- 将rebasingSupply改为totalShares
- 定义初始sharePrice=1e30

```solidity
    struct RebasingState {
        bool isRebasing;
        uint256 nShares;
    }
    mapping(address => RebasingState) internal rebasingAccount;
    uint256 public totalShares;
    uint256 public sharePrice = 1e30;
```

用下面的函数处理share和balance的映射关系：
```solidity
    function rebasingSupply() public view returns (uint256) {
        return share2Balance(totalShares);
    }

    function nonRebasingSupply() public view returns (uint256) {
        return totalSupply() - rebasingSupply();
    }

    function share2Balance(uint256 shares) view public returns (uint256) {
        return shares * sharePrice / 1e30;
    }

    function balance2Share(uint256 balance) view public returns (uint256) {
        return balance * 1e30 / sharePrice ;
    }
```

继续改写enter/exit和distribute部分：

- enter时先把balance转换为share，更新rebase记录
- exit时把share转换为balance，移除rebase记录
- distribute时，先销毁token，再更新sharePrice

```solidity
    function _enterRebase(address user) internal {
        uint256 balance = balanceOf(user);
        uint256 shares = balance2Share(balance);
        rebasingAccount[user].isRebasing = true;
        rebasingAccount[user].nShares = shares;
        totalShares += shares;
        emit RebaseEnter(user, shares, block.timestamp);
    }

    function _exitRebase(address user)  internal {
        uint256 shares = rebasingAccount[user].nShares;
        rebasingAccount[user].isRebasing = false;
        rebasingAccount[user].nShares = 0;
        totalShares -= shares;
        emit RebaseExit(user, shares, block.timestamp);
    }

    function distribute(uint256 amount) external {
        require(balanceOf(msg.sender)>=amount, "SimpleERC20Rebase: not enough");
        _burn(msg.sender, amount);
        sharePrice += amount*1e30 / totalShares;
    }
```

### ERC20方法重写

同时，我们需要对ERC20的几个方法进行重写：

- balanceOf：对于参与rebase的用户需要用share转换到balance
- mint burn transfer transferFrom：对于相关用户，如果之前参与rebase的需要先exit，操作后再enter

```solidity
    function balanceOf(address account) public view override returns (uint256) {
        uint256 rawBalance = ERC20.balanceOf(account);
        if (rebasingAccount[account].isRebasing) {
            return share2Balance(rebasingAccount[account].nShares);
        } else {
            return rawBalance;
        }
    }

    function mint(address user, uint256 amount) external {
        bool isRebasing = rebasingAccount[user].isRebasing;
        if (isRebasing) {
            _exitRebase(user);
        }
        ERC20._mint(user, amount);
        if (isRebasing) {
            _enterRebase(user);
        }
    }
    
    function burn(address user, uint256 amount) external {
        bool isRebasing = rebasingAccount[user].isRebasing;
        if (isRebasing) {
            _exitRebase(user);
        }
        ERC20._burn(user, amount);
        if (isRebasing) {
            _enterRebase(user);
        }
    }
    
    function transfer(address to, uint256 amount)  public virtual override returns (bool) {
        bool isFromRebasing = rebasingAccount[msg.sender].isRebasing;
        bool isToRebasing = rebasingAccount[to].isRebasing;
        if (isFromRebasing) {
            _exitRebase(msg.sender);
        }
        if (isToRebasing && to != msg.sender) {
            _exitRebase(to);
        }
        bool result = ERC20.transfer(to, amount);
        if (isFromRebasing) {
            _enterRebase(msg.sender);
        }
        if (isToRebasing && to != msg.sender) {
            _enterRebase(to);
        }
        return result;
    }

    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public virtual override returns (bool) {
        bool isFromRebasing = rebasingAccount[from].isRebasing;
        bool isToRebasing = rebasingAccount[to].isRebasing;
        if (isFromRebasing) {
            _exitRebase(from);
        }
        if (isToRebasing && to != from) {
            _exitRebase(to);
        }
        bool result = ERC20.transfer(to, amount);
        if (isFromRebasing) {
            _enterRebase(from);
        }
        if (isToRebasing && to != from) {
            _enterRebase(to);
        }
        return result;
    }
```

### 遗留的问题

大致逻辑已经实现了，用户的balance也能通过share和sharePrice来正确计算，我们再来看是否满足了前面的恒等式：

- `totalSupply() == nonRebasingSupply() + rebasingSupply()`
    - rebasingSupply可以用当前的share和sharePrice计算出，但是totalSupply我们没有重写，totalSupply=nonRebasingSupply，以distribute为例，nonRebasingSupply减少了，但是share不变，sharePrice还增加了，那么显然totalSupply小于rebasingSupply
- `sum of balanceOf(x) == totalSupply()`
    - 同理，用户的share不变但是sharePrice增加了，显然totalSupply也偏小

那么缺少的这部分是什么？

### Code

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.13;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract SimpleERC20Rebase is ERC20 {
    event RebaseEnter(address indexed account, uint256 indexed shares, uint256 indexed timestamp);
    event RebaseExit(address indexed account, uint256 indexed shares, uint256 indexed timestamp);
    
    struct RebasingState {
        bool isRebasing;
        uint256 nShares;
    }
    mapping(address => RebasingState) internal rebasingAccount;
    uint256 public totalShares;
    uint256 public sharePrice = 1e30;

    constructor(
        string memory _name,
        string memory _symbol
    ) ERC20(_name, _symbol) {}

    function rebasingSupply() public view returns (uint256) {
        return share2Balance(totalShares);
    }

    function nonRebasingSupply() public view returns (uint256) {
        return totalSupply() - rebasingSupply();
    }

    function share2Balance(uint256 shares) view public returns (uint256) {
        return shares * sharePrice / 1e30;
    }

    function balance2Share(uint256 balance) view public returns (uint256) {
        return balance * 1e30 / sharePrice ;
    }

    function enterRebase() external {
        require(!rebasingAccount[msg.sender].isRebasing, "SimpleERC20Rebase: already rebasing");
        _enterRebase(msg.sender);
    }

    function _enterRebase(address user) internal {
        uint256 balance = balanceOf(user);
        uint256 shares = balance2Share(balance);
        rebasingAccount[user].isRebasing = true;
        rebasingAccount[user].nShares = shares;
        totalShares += shares;
        emit RebaseEnter(user, shares, block.timestamp);
    }

    function exitRebase()  external {
        require(rebasingAccount[msg.sender].isRebasing, "SimpleERC20Rebase: not rebasing");
        _exitRebase(msg.sender);
    }

    function _exitRebase(address user)  internal {
        uint256 shares = rebasingAccount[user].nShares;
        rebasingAccount[user].isRebasing = false;
        rebasingAccount[user].nShares = 0;
        totalShares -= shares;
        emit RebaseExit(user, shares, block.timestamp);
    }

    function distribute(uint256 amount) external {
        require(balanceOf(msg.sender)>=amount, "SimpleERC20Rebase: not enough");
        _burn(msg.sender, amount);
        sharePrice += amount*1e30 / totalShares;
    }

    function balanceOf(address account) public view override returns (uint256) {
        uint256 rawBalance = ERC20.balanceOf(account);
        if (rebasingAccount[account].isRebasing) {
            return share2Balance(rebasingAccount[account].nShares);
        } else {
            return rawBalance;
        }
    }

    function mint(address user, uint256 amount) external {
        bool isRebasing = rebasingAccount[user].isRebasing;
        if (isRebasing) {
            _exitRebase(user);
        }
        ERC20._mint(user, amount);
        if (isRebasing) {
            _enterRebase(user);
        }
    }
    
    function burn(address user, uint256 amount) external {
        bool isRebasing = rebasingAccount[user].isRebasing;
        if (isRebasing) {
            _exitRebase(user);
        }
        ERC20._burn(user, amount);
        if (isRebasing) {
            _enterRebase(user);
        }
    }
    
    function transfer(address to, uint256 amount)  public virtual override returns (bool) {
        bool isFromRebasing = rebasingAccount[msg.sender].isRebasing;
        bool isToRebasing = rebasingAccount[to].isRebasing;
        if (isFromRebasing) {
            _exitRebase(msg.sender);
        }
        if (isToRebasing && to != msg.sender) {
            _exitRebase(to);
        }
        bool result = ERC20.transfer(to, amount);
        if (isFromRebasing) {
            _enterRebase(msg.sender);
        }
        if (isToRebasing && to != msg.sender) {
            _enterRebase(to);
        }
        return result;
    }

    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public virtual override returns (bool) {
        bool isFromRebasing = rebasingAccount[from].isRebasing;
        bool isToRebasing = rebasingAccount[to].isRebasing;
        if (isFromRebasing) {
            _exitRebase(from);
        }
        if (isToRebasing && to != from) {
            _exitRebase(to);
        }
        bool result = ERC20.transfer(to, amount);
        if (isFromRebasing) {
            _enterRebase(from);
        }
        if (isToRebasing && to != from) {
            _enterRebase(to);
        }
        return result;
    }
}
```

## 3 补偿铸造

### 实际例子

我们以实际例子展开，假设A、B均开启rebase，且初始balance和share均为100，C未开启rebase，balance为200

此时的状态如下，可见是满足等式条件的：

- totalSupply = 100 + 100 + 200 = 400
- sharePrice = 1
- rebasingSupply = (100 + 100) * 1 = 200
- nonRebasingSupply = 200

那么C调用distribute来贡献自己的200 token，状态变化为：

- totalSupply = 100 + 100 - 200 = 0
- sharePrice = 1 + (200/200) = 2
- rebasingSupply = (100 + 100) * 2 = 400
- nonRebasingSupply = 0

此时显然等式不再成立，为了保证等式成立，totalSupply应该是400而不是0，回到第一个版本，我们每次调用distribute时都会通过_mint来修改每个参与rebase用户的balance，实际上系统是增发了，因为前面进行了销毁操作，因此保持了平衡

那么在第二版中，前面也进行了销毁，但是却没有增发，只是更新sharePrice，那么就算用户账面上的balance增加了，但是实际来领取时，系统总量是不够的。因此，这里差的400就是需要增发的量，同理我们不需要按比例增发给每个用户，而是记录在一个全局变量中，当用户退出rebase时再mint出相应的数额

### 补偿铸造

首先定义全局的unminted变量：
```solidity
uint256 public unminted;
```

unminted需要在distribute时增加，在exit时减少：

```solidity
    function _exitRebase(address user)  internal {
        uint256 shares = rebasingAccount[user].nShares;
        rebasingAccount[user].isRebasing = false;
        rebasingAccount[user].nShares = 0;
        totalShares -= shares;
        uint256 balance = share2Balance(shares);
        uint256 rawBalance = ERC20.balanceOf(user);
        if (balance > rawBalance) {
            uint256 delta = balance - rawBalance;
            ERC20._mint(user, delta);
            unminted -= delta;
        }
        emit RebaseExit(user, shares, block.timestamp);
    }

    function distribute(uint256 amount) external {
        require(balanceOf(msg.sender)>=amount, "SimpleERC20Rebase: not enough");
        _burn(msg.sender, amount);
        sharePrice += amount*1e30 / totalShares;
        unminted += amount;
    }
```

修改后的代码如下，我们新增了unminted变量，并在distribute时累加，在exit时增发相应数目token给用户

至此，ERC20Rebase的核心框架已经实现了，不过代码仅供参考，只关注了核心逻辑的实现，对比ecg中的ERC20RebaseDistributor合约，我们仍欠缺很关键的部分：线性释放，分红的token不是一次性反馈给holder，而是在一定的周期内线性增加。

由于添加了时间的维度，相应的也会衍生出许多问题：如果分红周期内share总量发生变化，如何保证公平分发？

### Code

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.13;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract SimpleERC20Rebase is ERC20 {
    event RebaseEnter(address indexed account, uint256 indexed shares, uint256 indexed timestamp);
    event RebaseExit(address indexed account, uint256 indexed shares, uint256 indexed timestamp);
    
    struct RebasingState {
        bool isRebasing;
        uint256 nShares;
    }
    mapping(address => RebasingState) internal rebasingAccount;
    uint256 public totalShares;
    uint256 public unminted;
    uint256 public sharePrice = 1e30;

    constructor(
        string memory _name,
        string memory _symbol
    ) ERC20(_name, _symbol) {}

    function rebasingSupply() public view returns (uint256) {
        return share2Balance(totalShares);
    }

    function nonRebasingSupply() public view returns (uint256) {
        return totalSupply() - rebasingSupply();
    }

    function share2Balance(uint256 shares) view public returns (uint256) {
        return shares * sharePrice / 1e30;
    }

    function balance2Share(uint256 balance) view public returns (uint256) {
        return balance * 1e30 / sharePrice ;
    }

    function enterRebase() external {
        require(!rebasingAccount[msg.sender].isRebasing, "SimpleERC20Rebase: already rebasing");
        _enterRebase(msg.sender);
    }

    function _enterRebase(address user) internal {
        uint256 balance = balanceOf(user);
        uint256 shares = balance2Share(balance);
        rebasingAccount[user].isRebasing = true;
        rebasingAccount[user].nShares = shares;
        totalShares += shares;
        emit RebaseEnter(user, shares, block.timestamp);
    }

    function exitRebase()  external {
        require(rebasingAccount[msg.sender].isRebasing, "SimpleERC20Rebase: not rebasing");
        _exitRebase(msg.sender);
    }

    function _exitRebase(address user)  internal {
        uint256 shares = rebasingAccount[user].nShares;
        rebasingAccount[user].isRebasing = false;
        rebasingAccount[user].nShares = 0;
        totalShares -= shares;
        uint256 balance = share2Balance(shares);
        uint256 rawBalance = ERC20.balanceOf(user);
        if (balance > rawBalance) {
            uint256 delta = balance - rawBalance;
            ERC20._mint(user, delta);
            unminted -= delta;
        }
        emit RebaseExit(user, shares, block.timestamp);
    }

    function distribute(uint256 amount) external {
        require(balanceOf(msg.sender)>=amount, "SimpleERC20Rebase: not enough");
        _burn(msg.sender, amount);
        sharePrice += amount*1e30 / totalShares;
        unminted += amount;
    }

    function totalSupply() public view override returns (uint256) {
        return super.totalSupply() + unminted;
    }

    function balanceOf(address account) public view override returns (uint256) {
        uint256 rawBalance = ERC20.balanceOf(account);
        if (rebasingAccount[account].isRebasing) {
            return share2Balance(rebasingAccount[account].nShares);
        } else {
            return rawBalance;
        }
    }

    function mint(address user, uint256 amount) external {
        bool isRebasing = rebasingAccount[user].isRebasing;
        if (isRebasing) {
            _exitRebase(user);
        }
        ERC20._mint(user, amount);
        if (isRebasing) {
            _enterRebase(user);
        }
    }

    function transfer(address to, uint256 amount)  public virtual override returns (bool) {
        bool isFromRebasing = rebasingAccount[msg.sender].isRebasing;
        bool isToRebasing = rebasingAccount[to].isRebasing;
        if (isFromRebasing) {
            _exitRebase(msg.sender);
        }
        if (isToRebasing && to != msg.sender) {
            _exitRebase(to);
        }
        bool result = ERC20.transfer(to, amount);
        if (isFromRebasing) {
            _enterRebase(msg.sender);
        }
        if (isToRebasing && to != msg.sender) {
            _enterRebase(to);
        }
        return result;
    }

    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public virtual override returns (bool) {
        bool isFromRebasing = rebasingAccount[from].isRebasing;
        bool isToRebasing = rebasingAccount[to].isRebasing;
        if (isFromRebasing) {
            _exitRebase(from);
        }
        if (isToRebasing && to != from) {
            _exitRebase(to);
        }
        bool result = ERC20.transfer(to, amount);
        if (isFromRebasing) {
            _enterRebase(from);
        }
        if (isToRebasing && to != from) {
            _enterRebase(to);
        }
        return result;
    }
}
```