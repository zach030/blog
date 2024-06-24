---
slug: the-default-framework
title: 智能合约架构--The Default Framework
date: 2024-01-09
authors: [Zach]
math: true
categories:
  - system design
---

>在阅读OlympusV3的合约时，架构的设计给我留下很深的印象，架构清晰，在智能合约领域实现了高内聚低耦合的设计思想，从而可以专注的深入独立的模块阅读，降低了心智负担。

OlympusV3的架构图如下所示：
![image](https://pic.imgdb.cn/item/659c9468871b83018a59c3fd.jpg)

后来了解到这种框架也并非Olympus独创，在Olympus介绍合约架构的[文章](https://docs.olympusdao.finance/main/technical/overview)中提到了[Default框架](https://github.com/fullyallocated/Default)

## 智能合约架构的复杂性

在介绍Default框架之前，作者首先对比了传统的协议如AAVE MakerDAO的架构图
![MakerDAO架构图](https://pic.imgdb.cn/item/6678b7ecd9c307b7e9fc1387.png)
![AAVE架构图](https://2799188404-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-M3C77KySce4HXyLqkEq%2F-MNbklkK7vNPshPbrCZ-%2F-MNbxHseq3eFT7u8gEo3%2Fimage.png?alt=media&token=61a006eb-8d2b-4de6-8498-05fc889feed3)

光看架构图，开发者是很难厘清其中的调用链路的，作者将这种设计解释为以过程为中心，按照逻辑链的思路来组织
> Today, most projects take a process-centric approach towards protocol development: each smart contract is developed to manage a process along a chain of logic that is triggered when external interactions occur.

那么随着业务的升级和更替，逻辑链路的复杂度会指数级增长，原因在于整个协议呈现网状架构，或许每两个点之间都有逻辑链，那么每增添一个点，带来的复杂度提升是难以估量和覆盖的。

正如软件工程领域的设计模式演进过程一样，智能合约的工程化架构也在不断改进，Default Framework正是着手解决这个问题

## 什么是The Default Framework

光从名字看不出来设计的思路，不像[Diamond框架](https://eips.ethereum.org/EIPS/eip-2535)能够让人产生联想。The Default Framework也没有做特别的创新，而是借鉴了传统领域开发的设计思想，对整个合约进行了分层
- Modules：面向内部，模块可以理解为微服务，每个模块管理一部分数据，彼此之间不存在依赖关系，高度独立，对于模块的访问都到权限管理约束
- Policy：面向外部调用，在内部负责组织不同的模块以实现业务逻辑

核心的管理逻辑都收口在[Kernal.sol](https://github.com/fullyallocated/Default/blob/master/src/Kernel.sol)文件中，负责管控全局的Policy和Modules

### Modules

每个Modules包括独一无二的KEYCODE及VERSION，在定义Modules时需要实现此方法，在Kernal中会维护所有的keycode及其与Module的映射关系

在Kernal中包括以下方法
- _installModule：安装module
- _upgradeModule：升级module

示例：
```solidity
    /// @inheritdoc Module
    function KEYCODE() public pure override returns (Keycode) {
        return toKeycode("MINTR");
    }

    /// @inheritdoc Module
    function VERSION() external pure override returns (uint8 major, uint8 minor) {
        major = 1;
        minor = 0;
    }
```

### Policy

Policy可以理解为传统软件开发的service层，用来组织一系列Modules，包括以下方法：

- configureDependencies：返回支持哪些Modules
- requestPermissions：返回支持的Modules的具体函数selector

在kernal中包括以下方法：

- _activatePolicy
- _deactivatePolicy

示例：
```solidity
    /// @inheritdoc Policy
    function configureDependencies() external override returns (Keycode[] memory dependencies) {
        dependencies = new Keycode[](3);
        dependencies[0] = toKeycode("MINTR");
        dependencies[1] = toKeycode("TRSRY");
        dependencies[2] = toKeycode("ROLES");

        MINTR = MINTRv1(getModuleAddress(dependencies[0]));
        TRSRY = TRSRYv1(getModuleAddress(dependencies[1]));
        ROLES = ROLESv1(getModuleAddress(dependencies[2]));

        (uint8 TRSRY_MAJOR, ) = TRSRY.VERSION();
        (uint8 MINTR_MAJOR, ) = MINTR.VERSION();
        (uint8 ROLES_MAJOR, ) = ROLES.VERSION();

        // Ensure Modules are using the expected major version.
        // Modules should be sorted in alphabetical order.
        bytes memory expected = abi.encode([1, 1, 1]);
        if (MINTR_MAJOR != 1 || ROLES_MAJOR != 1 || TRSRY_MAJOR != 1)
            revert Policy_WrongModuleVersion(expected);
    }

    /// @inheritdoc Policy
    function requestPermissions() external view override returns (Permissions[] memory requests) {
        Keycode MINTR_KEYCODE = MINTR.KEYCODE();

        requests = new Permissions[](4);
        requests[0] = Permissions(MINTR_KEYCODE, MINTR.mintOhm.selector);
        requests[1] = Permissions(MINTR_KEYCODE, MINTR.burnOhm.selector);
        requests[2] = Permissions(MINTR_KEYCODE, MINTR.increaseMintApproval.selector);
        requests[3] = Permissions(MINTR_KEYCODE, MINTR.decreaseMintApproval.selector);
    }
```