---
slug: eigenlayer-analysis
title: EigenLayer 合约解析
date: 2024-02-05
authors: [Zach]
math: true
categories:
  - Defi
---

## Strategy

![整体架构图](https://pic.imgdb.cn/item/6678b863d9c307b7e9fc9370.png)

### StrategyBase

由`StrategyBase`和`StrategyBaseTVLLimits`构成，`StrategyBase`是基础逻辑，`StrategyBaseTVLLimits`继承自`StrategyBase`且在此基础之上拓展。

`StrategyBase`的核心方法就是`deposit`和`withdraw`，分别提供质押和提取功能，用户质押入token，会获得相应份额的shares，提取时再根据shares计算出应得的token数量

- `deposit`方法

```solidity
function deposit(
        IERC20 token,
        uint256 amount
    ) external virtual override onlyWhenNotPaused(PAUSED_DEPOSITS) onlyStrategyManager returns (uint256 newShares) {
        // 要求deposit未暂停，且只有strategyManager可以调用
        // call hook to allow for any pre-deposit logic
        _beforeDeposit(token, amount); // deposit前的hook，在本合约内是虚函数，需要继承的合约去实现
        // 每个strategy只支持一个底层token
        require(token == underlyingToken, "StrategyBase.deposit: Can only deposit underlyingToken");

        // copy `totalShares` value to memory, prior to any change
        uint256 priorTotalShares = totalShares;

        /**
         * @notice calculation of newShares *mirrors* `underlyingToShares(amount)`, but is different since the balance of `underlyingToken`
         * has already been increased due to the `strategyManager` transferring tokens to this strategy prior to calling this function
         */
        // account for virtual shares and balance
        // 计算virtual的share和token总额度，这里都加上了OFFSET，用来避免shares膨胀攻击，后面会介绍
        uint256 virtualShareAmount = priorTotalShares + SHARES_OFFSET;
        uint256 virtualTokenBalance = _tokenBalance() + BALANCE_OFFSET;
        // calculate the prior virtual balance to account for the tokens that were already transferred to this contract
        uint256 virtualPriorTokenBalance = virtualTokenBalance - amount;
        // 按照比例计算本次质押的amount可以得到多少新的shares
        newShares = (amount * virtualShareAmount) / virtualPriorTokenBalance;

        // extra check for correctness / against edge case where share rate can be massively inflated as a 'griefing' sort of attack
        require(newShares != 0, "StrategyBase.deposit: newShares cannot be zero");
        // 更新累加的总shares
        // update total share amount to account for deposit
        totalShares = (priorTotalShares + newShares);
        return newShares;
    }
```

- `withdraw`方法

```solidity
function withdraw(
        address recipient,
        IERC20 token,
        uint256 amountShares
    ) external virtual override onlyWhenNotPaused(PAUSED_WITHDRAWALS) onlyStrategyManager {
        // 要求withdraw未暂停，且只有strategyManager可以调用
        // call hook to allow for any pre-withdrawal logic
        _beforeWithdrawal(recipient, token, amountShares);

        require(token == underlyingToken, "StrategyBase.withdraw: Can only withdraw the strategy token");

        // copy `totalShares` value to memory, prior to any change
        uint256 priorTotalShares = totalShares;

        require(
            // 不得提取超出上限的shares
            amountShares <= priorTotalShares,
            "StrategyBase.withdraw: amountShares must be less than or equal to totalShares"
        );

        /**
         * @notice calculation of amountToSend *mirrors* `sharesToUnderlying(amountShares)`, but is different since the `totalShares` has already
         * been decremented. Specifically, notice how we use `priorTotalShares` here instead of `totalShares`.
         */
        // account for virtual shares and balance
        // 同样加上OFFSET统一处理
        uint256 virtualPriorTotalShares = priorTotalShares + SHARES_OFFSET;
        uint256 virtualTokenBalance = _tokenBalance() + BALANCE_OFFSET;
        // calculate ratio based on virtual shares and balance, being careful to multiply before dividing
        // 计算提取出amountShares，可以得到多少token
        uint256 amountToSend = (virtualTokenBalance * amountShares) / virtualPriorTotalShares;

        // Decrease the `totalShares` value to reflect the withdrawal
        // 更新最新的总份额数
        totalShares = priorTotalShares - amountShares;
        // 把相应数额的token转移给接收方
        underlyingToken.safeTransfer(recipient, amountToSend);
    }
```

`StrategyBaseTVLLimits`继承自`StrategyBase`，当前eigenlayer主网的所有底层策略都是基于本合约实现的。

本合约内加了两个变量：`maxPerDeposit`和`maxTotalDeposits`

- `maxPerDeposit`：限制单次deposit的上限
- `maxTotalDeposits`：限制累计的deposit上限

### StrategyManager

Strategy合约中的`deposit`和`withdraw`方法都限制仅`strategyManager`调用，`strategyManager`主要包括两个合约代码：

- `StrategyManagerStorage`：抽象合约，用于定义基础变量
- `StrategyManager`：继承自`StrategyManagerStorage`，实现核心逻辑

`StrategyManager`对外方法较多，按照调用方划分，分为无权限校验，仅delegationManager调用，仅owner调用，仅strategyWhitelister调用

**任何人可以直接调用的**

- `depositIntoStrategy`：转移token到strategy，累增shares

```solidity
function _depositIntoStrategy(
        address staker,
        IStrategy strategy,
        IERC20 token,
        uint256 amount
    ) internal onlyStrategiesWhitelistedForDeposit(strategy) returns (uint256 shares) {
        // 仅限白名单内的策略
        // transfer tokens from the sender to the strategy
        // 直接转移token到strategy合约
        token.safeTransferFrom(msg.sender, address(strategy), amount);
        // 调用strategy的deposit方法
        // deposit the assets into the specified strategy and get the equivalent amount of shares in that strategy
        shares = strategy.deposit(token, amount);

        // add the returned shares to the staker's existing shares for this strategy
        // 累加用户在指定strategy下的shares
        _addShares(staker, strategy, shares);

        // Increase shares delegated to operator, if needed
        // 调用delegation，尝试累加deposit shares
        delegation.increaseDelegatedShares(staker, strategy, shares);

        emit Deposit(staker, token, strategy, shares);
        return shares;
    }
```

- `depositIntoStrategyWithSignature`：校验链下签名，再调用到`_depositIntoStrategy`

```solidity
function depositIntoStrategyWithSignature(
        IStrategy strategy,
        IERC20 token,
        uint256 amount,
        address staker,
        uint256 expiry,
        bytes memory signature
    ) external onlyWhenNotPaused(PAUSED_DEPOSITS) nonReentrant returns (uint256 shares) {
        // 校验签名未过期
        require(expiry >= block.timestamp, "StrategyManager.depositIntoStrategyWithSignature: signature expired");
        // calculate struct hash, then increment `staker`'s nonce
        // 累增nonce，按照规则计算hash
        uint256 nonce = nonces[staker];
        bytes32 structHash = keccak256(abi.encode(DEPOSIT_TYPEHASH, strategy, token, amount, nonce, expiry));
        unchecked {
            nonces[staker] = nonce + 1;
        }

        // calculate the digest hash
        bytes32 digestHash = keccak256(abi.encodePacked("\x19\x01", domainSeparator(), structHash));

        /**
         * check validity of signature:
         * 1) if `staker` is an EOA, then `signature` must be a valid ECDSA signature from `staker`,
         * indicating their intention for this action
         * 2) if `staker` is a contract, then `signature` will be checked according to EIP-1271
         */
        // 校验签名是否来自于staker
        EIP1271SignatureUtils.checkSignature_EIP1271(staker, digestHash, signature);

        // deposit the tokens (from the `msg.sender`) and credit the new shares to the `staker`
        // 调用到底层deposit方法
        shares = _depositIntoStrategy(staker, strategy, token, amount);
    }
```

**仅delegationManager调用**

- `removeShares`：扣除用户在指定strategy下的shares
- `addShares`：增加用户在指定strategy下的shares
- `withdrawSharesAsTokens`：调用strategy的withdraw方法
- `migrateQueuedWithdrawal`：deprecated

**仅owner调用**

- `setStrategyWhitelister`：设置strategyWhitelister

**仅strategyWhitelister调用**

- `addStrategiesToDepositWhitelist`：新增strategy到whitelist
- `removeStrategiesFromDepositWhitelist`：从whitelist中移除strategy

可以看到`StrategyManager`核心的功能是

- 对外提供deposit和withdraw方法：deposit无限制，withdraw只有delegationManager调用
- 记录用户的shares
- 用白名单管理strategy

## Core

### DelegationManager

StrategyManager的几个方法被限制仅由delegationManager调用，delegationManager的合约代码由两部分组成：

- `DelegationManagerStorage`：抽象合约，定义变量
- `DelegationManager`：继承自`DelegationManagerStorage`，实现核心逻辑

核心的对外方法如下，根据调用者分为下面几组：代理商、用户、AVS、strategyManager或eigenPodManager

**代理商**

- `registerAsOperator`：注册成为operator，任何人都可以注册
- `modifyOperatorDetails`：修改operator的属性
- `updateOperatorMetadataURI`：更新metadata，发出event

**用户**

- `delegateTo`：把资产代理给代理商

```solidity
function _delegate(
        address staker,
        address operator,
        SignatureWithExpiry memory approverSignatureAndExpiry,
        bytes32 approverSalt
    ) internal onlyWhenNotPaused(PAUSED_NEW_DELEGATION) {
        // 只能代理给一个operator
        require(!isDelegated(staker), "DelegationManager._delegate: staker is already actively delegated");
        // staker自己不能是operator
        require(isOperator(operator), "DelegationManager._delegate: operator is not registered in EigenLayer");

        // fetch the operator's `delegationApprover` address and store it in memory in case we need to use it multiple times
        // 获取代理商的approver地址，这个地址用于颁发签名给用户，在下面用来做校验
        address _delegationApprover = _operatorDetails[operator].delegationApprover;
        /**
         * Check the `_delegationApprover`'s signature, if applicable.
         * If the `_delegationApprover` is the zero address, then the operator allows all stakers to delegate to them and this verification is skipped.
         * If the `_delegationApprover` or the `operator` themselves is the caller, then approval is assumed and signature verification is skipped as well.
         */
        if (_delegationApprover != address(0) && msg.sender != _delegationApprover && msg.sender != operator) {
            // check the signature expiry
            require(
                // 校验签名是否过期
                approverSignatureAndExpiry.expiry >= block.timestamp,
                "DelegationManager._delegate: approver signature expired"
            );
            // check that the salt hasn't been used previously, then mark the salt as spent
            require(
                // 校验salt是否被使用过
                !delegationApproverSaltIsSpent[_delegationApprover][approverSalt],
                "DelegationManager._delegate: approverSalt already spent"
            );
            // 标记此approver的salt为已使用
            delegationApproverSaltIsSpent[_delegationApprover][approverSalt] = true;

            // calculate the digest hash
            bytes32 approverDigestHash = calculateDelegationApprovalDigestHash(
                staker,
                operator,
                _delegationApprover,
                approverSalt,
                approverSignatureAndExpiry.expiry
            );
            // 校验签名是否有效
            // actually check that the signature is valid
            EIP1271SignatureUtils.checkSignature_EIP1271(
                _delegationApprover,
                approverDigestHash,
                approverSignatureAndExpiry.signature
            );
        }

        // record the delegation relation between the staker and operator, and emit an event
        // 记录代理关系
        delegatedTo[staker] = operator;
        emit StakerDelegated(staker, operator);
        // 获取用户当前已质押的strategy和对应的数目
        (IStrategy[] memory strategies, uint256[] memory shares)
            = getDelegatableShares(staker);

        for (uint256 i = 0; i < strategies.length;) {
            // 把之前的质押累加到代理商的记录上
            _increaseOperatorShares({
                operator: operator,
                staker: staker,
                strategy: strategies[i],
                shares: shares[i]
            });

            unchecked { ++i; }
        }
    }
```

- `delegateToBySignature`：同上，先校验签名再delegate
- `undelegate`：取消代理

```solidity
function undelegate(address staker) external onlyWhenNotPaused(PAUSED_ENTER_WITHDRAWAL_QUEUE) returns (bytes32[] memory withdrawalRoots) {
        // 校验之前存在代理记录
        require(isDelegated(staker), "DelegationManager.undelegate: staker must be delegated to undelegate");
        // 不能是代理商
        require(!isOperator(staker), "DelegationManager.undelegate: operators cannot be undelegated");
        require(staker != address(0), "DelegationManager.undelegate: cannot undelegate zero address");
        address operator = delegatedTo[staker];
        require(
            // 允许以下条件调用：质押人自己 or 代理商 or 代理商的授权地址
            msg.sender == staker ||
                msg.sender == operator ||
                msg.sender == _operatorDetails[operator].delegationApprover,
            "DelegationManager.undelegate: caller cannot undelegate staker"
        );

        // Gather strategies and shares to remove from staker/operator during undelegation
        // Undelegation removes ALL currently-active strategies and shares
        // 获取当前代理的策略及shares信息
        (IStrategy[] memory strategies, uint256[] memory shares) = getDelegatableShares(staker);

        // emit an event if this action was not initiated by the staker themselves
        if (msg.sender != staker) {
            // 如果不是质押人本人，发出事件
            emit StakerForceUndelegated(staker, operator);
        }

        // undelegate the staker
        emit StakerUndelegated(staker, operator);
        // 清理代理关系记录
        delegatedTo[staker] = address(0);

        // if no delegatable shares, return an empty array, and don't queue a withdrawal
        if (strategies.length == 0) {
            withdrawalRoots = new bytes32[](0);
        } else {
            // 遍历现存的质押的策略
            withdrawalRoots = new bytes32[](strategies.length);
            for (uint256 i = 0; i < strategies.length; i++) {
                IStrategy[] memory singleStrategy = new IStrategy[](1);
                uint256[] memory singleShare = new uint256[](1);
                singleStrategy[0] = strategies[i];
                singleShare[0] = shares[i];
                // 移除全部的份额，并返回root记录
                withdrawalRoots[i] = _removeSharesAndQueueWithdrawal({
                    staker: staker,
                    operator: operator,
                    withdrawer: staker,
                    strategies: singleStrategy,
                    shares: singleShare
                });
            }
        }

        return withdrawalRoots;
    }
```

```solidity
function _removeSharesAndQueueWithdrawal(
        address staker, 
        address operator,
        address withdrawer,
        IStrategy[] memory strategies, 
        uint256[] memory shares
    ) internal returns (bytes32) {
        require(staker != address(0), "DelegationManager._removeSharesAndQueueWithdrawal: staker cannot be zero address");
        require(strategies.length != 0, "DelegationManager._removeSharesAndQueueWithdrawal: strategies cannot be empty");
    
        // Remove shares from staker and operator
        // Each of these operations fail if we attempt to remove more shares than exist
        for (uint256 i = 0; i < strategies.length;) {
            // Similar to `isDelegated` logic
            if (operator != address(0)) {
                // 减少代理商的代理shares数目
                _decreaseOperatorShares({
                    operator: operator,
                    staker: staker,
                    strategy: strategies[i],
                    shares: shares[i]
                });
            }
            // 区分普通策略还是pod，分别调用removeShares
            // Remove active shares from EigenPodManager/StrategyManager
            if (strategies[i] == beaconChainETHStrategy) {
                /**
                 * This call will revert if it would reduce the Staker's virtual beacon chain ETH shares below zero.
                 * This behavior prevents a Staker from queuing a withdrawal which improperly removes excessive
                 * shares from the operator to whom the staker is delegated.
                 * It will also revert if the share amount being withdrawn is not a whole Gwei amount.
                 */
                eigenPodManager.removeShares(staker, shares[i]);
            } else {
                // this call will revert if `shares[i]` exceeds the Staker's current shares in `strategies[i]`
                strategyManager.removeShares(staker, strategies[i], shares[i]);
            }

            unchecked { ++i; }
        }

        // Create queue entry and increment withdrawal nonce
        uint256 nonce = cumulativeWithdrawalsQueued[staker];
        cumulativeWithdrawalsQueued[staker]++;

        Withdrawal memory withdrawal = Withdrawal({
            staker: staker,
            delegatedTo: operator,
            withdrawer: withdrawer,
            nonce: nonce,
            startBlock: uint32(block.number),
            strategies: strategies,
            shares: shares
        });
        // 将用户的每次提款请求记录hash并存到pending中
        bytes32 withdrawalRoot = calculateWithdrawalRoot(withdrawal);

        // Place withdrawal in queue
        pendingWithdrawals[withdrawalRoot] = true;

        emit WithdrawalQueued(withdrawalRoot, withdrawal);
        return withdrawalRoot;
    }
```

- `queueWithdrawals`：提出提款申请，返回凭证
- `completeQueuedWithdrawal`：完成提款

```solidity
function _completeQueuedWithdrawal(
        Withdrawal calldata withdrawal,
        IERC20[] calldata tokens,
        uint256 /*middlewareTimesIndex*/,
        bool receiveAsTokens
    ) internal {
        // 根据用户的输入，计算提款hash
        bytes32 withdrawalRoot = calculateWithdrawalRoot(withdrawal);

        require(
            // 确保提款记录有效
            pendingWithdrawals[withdrawalRoot], 
            "DelegationManager.completeQueuedAction: action is not in queue"
        );

        require(
            // 确保距离申请已经超出等待时间
            withdrawal.startBlock + withdrawalDelayBlocks <= block.number, 
            "DelegationManager.completeQueuedAction: withdrawalDelayBlocks period has not yet passed"
        );

        require(
            // 必须是申请本人调用
            msg.sender == withdrawal.withdrawer, 
            "DelegationManager.completeQueuedAction: only withdrawer can complete action"
        );

        if (receiveAsTokens) {
            require(
                tokens.length == withdrawal.strategies.length, 
                "DelegationManager.completeQueuedAction: input length mismatch"
            );
        }

        // Remove `withdrawalRoot` from pending roots
        // 移除待完成的提款记录
        delete pendingWithdrawals[withdrawalRoot];

        // Finalize action by converting shares to tokens for each strategy, or
        // by re-awarding shares in each strategy.
        if (receiveAsTokens) {
            for (uint256 i = 0; i < withdrawal.strategies.length; ) {
                // 调用strategyManager的withdrawSharesAsTokens，根据用户的shares接收到相应数目的token
                _withdrawSharesAsTokens({
                    staker: withdrawal.staker,
                    withdrawer: msg.sender,
                    strategy: withdrawal.strategies[i],
                    shares: withdrawal.shares[i],
                    token: tokens[i]
                });
                unchecked { ++i; }
            }
        // Award shares back in StrategyManager/EigenPodManager. If withdrawer is delegated, increase the shares delegated to the operator
        } else {
            // 如果不选择接收token，这里就会默认把提款申请的shares重新投入质押
            address currentOperator = delegatedTo[msg.sender];
            for (uint256 i = 0; i < withdrawal.strategies.length; ) {
                /** When awarding podOwnerShares in EigenPodManager, we need to be sure to only give them back to the original podOwner.
                 * Other strategy shares can + will be awarded to the withdrawer.
                 */
                if (withdrawal.strategies[i] == beaconChainETHStrategy) {
                    address staker = withdrawal.staker;
                    /**
                    * Update shares amount depending upon the returned value.
                    * The return value will be lower than the input value in the case where the staker has an existing share deficit
                    */
                    uint256 increaseInDelegateableShares = eigenPodManager.addShares({
                        podOwner: staker,
                        shares: withdrawal.shares[i]
                    });
                    address podOwnerOperator = delegatedTo[staker];
                    // Similar to `isDelegated` logic
                    if (podOwnerOperator != address(0)) {
                        _increaseOperatorShares({
                            operator: podOwnerOperator,
                            // the 'staker' here is the address receiving new shares
                            staker: staker,
                            strategy: withdrawal.strategies[i],
                            shares: increaseInDelegateableShares
                        });
                    }
                } else {
                    strategyManager.addShares(msg.sender, withdrawal.strategies[i], withdrawal.shares[i]);
                    // Similar to `isDelegated` logic
                    // 如果用户存在代理记录，累加到代理商的shares上
                    if (currentOperator != address(0)) {
                        _increaseOperatorShares({
                            operator: currentOperator,
                            // the 'staker' here is the address receiving new shares
                            staker: msg.sender,
                            strategy: withdrawal.strategies[i],
                            shares: withdrawal.shares[i]
                        });
                    }
                }
                unchecked { ++i; }
            }
        }

        emit WithdrawalCompleted(withdrawalRoot);
    }
```

- `completeQueuedWithdrawals`：批量操作，完成取款

**由strategyManager或eigenPodManager调用**

- `increaseDelegatedShares`：增加质押人所在的代理商的shares数
- `decreaseDelegatedShares`：减少质押人所在的代理商的shares数

**avs调用**

- `registerOperatorToAVS`：注册代理商到avs

```solidity
function registerOperatorToAVS(
        address operator,
        ISignatureUtils.SignatureWithSaltAndExpiry memory operatorSignature
    ) external onlyWhenNotPaused(PAUSED_OPERATOR_REGISTER_DEREGISTER_TO_AVS) {

        require(
            operatorSignature.expiry >= block.timestamp,
            "DelegationManager.registerOperatorToAVS: operator signature expired"
        );
        require(
            // 校验此代理商没有被此avs注册过
            avsOperatorStatus[msg.sender][operator] != OperatorAVSRegistrationStatus.REGISTERED,
            "DelegationManager.registerOperatorToAVS: operator already registered"
        );
        require(
            // 校验此次注册请求salt是否被使用过
            !operatorSaltIsSpent[operator][operatorSignature.salt],
            "DelegationManager.registerOperatorToAVS: salt already spent"
        );
        require(
            // 校验是合法的代理商
            isOperator(operator),
            "DelegationManager.registerOperatorToAVS: operator not registered to EigenLayer yet");

        // Calculate the digest hash
        // 校验签名有效性
        bytes32 operatorRegistrationDigestHash = calculateOperatorAVSRegistrationDigestHash({
            operator: operator,
            avs: msg.sender,
            salt: operatorSignature.salt,
            expiry: operatorSignature.expiry
        });

        // Check that the signature is valid
        EIP1271SignatureUtils.checkSignature_EIP1271(
            operator,
            operatorRegistrationDigestHash,
            operatorSignature.signature
        );

        // Set the operator as registered
        // 标记完成代理商到avs注册
        avsOperatorStatus[msg.sender][operator] = OperatorAVSRegistrationStatus.REGISTERED;

        // Mark the salt as spent
        // 标记此salt被使用过
        operatorSaltIsSpent[operator][operatorSignature.salt] = true;

        emit OperatorAVSRegistrationStatusUpdated(operator, msg.sender, OperatorAVSRegistrationStatus.REGISTERED);
    }
```

- `deregisterOperatorFromAVS`：avs移除代理商的注册关系

## Pod

pod与strategy是同级的，区别是strategy进行的是流动性代币质押，而pod是进行原生质押，直接调用质押合约

> 主网及goerli的pos质押合约
> 
> 
> [Ethereum DepositContract](https://etherscan.io/address/0x00000000219ab540356cbb839cbe05303d7705fa)
> 
> [Goerli DepositContract](https://goerli.etherscan.io/address/0xff50ed3d0ec03aC01D4C79aAd74928BFF48a7b2b)
> 

### EigenPod & EigenPodManager

- `createPod`：使用create2部署pod合约，记录ownerToPod
- `stake`：调用自己为owner的pod的stake方法
- `recordBeaconChainETHBalanceUpdate`
- `removeShares`：减少shares
- `addShares`：增加shares
- `withdrawSharesAsTokens`：取回指定数目的eth