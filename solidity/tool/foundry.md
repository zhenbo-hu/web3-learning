# Foundry

## 什么是 Foundry？

- 使用 Rust 语言编写
- 用于以太坊等 EVM 智能合约开发
- 相比于 Hardhat 等通过 Node.js 完成编译测试的工程项目，Foundry 工程项目的构建、测试执行速度更快
- Foundry 工程支持与其他类型工程集成，如 Hardhat
- 通过 git submodule 并构建目录映射，更加模块化，可以更便捷的引入依赖
- 开发、测试都只需要 solidity 语言
- 通过 Cheatcodes 作弊码，可以操纵区块链的状态，进而更全面的测试
- 测试中包含 gas 使用信息

## 安装

```shell
curl -L https://foundry.paradigm.xyz | bash

foundryup
```

## 初始化一个 Foundry 项目

```shell
forge init
```

初始化的项目目录结构

```shell
$ tree -L 2
.
├── foundry.toml        # Foundry 的 package 配置文件
├── lib                 # Foundry 的依赖库
│   └── forge-std       # 工具 forge 的基础依赖
├── script              # Foundry 的脚本
│   └── Counter.s.sol   # 示例合约 Counter 的脚本
├── src                 # 智能合约的业务逻辑、源代码将会放在这里
│   └── Counter.sol     # 示例合约
└── test                # 测试用例目录
    └── Counter.t.sol   # 示例合约的测试用例
```

### src 目录

项目的核心逻辑

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract Counter {          // 一个很简单的 Counter 合约
    uint256 public number;  // 维护一个 public 的 uint256 数字

    // 设置 number 变量的内容
    function setNumber(uint256 newNumber) public {
        number = newNumber;
    }

    // 让 number 变量的内容自增
    function increment() public {
        number++;
    }
}
```

### script 目录

等同于 hardhat 中的 scripts，用于合约部署

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13; // 许可 和 Solidity版本标识

import "forge-std/Script.sol"; // 引入foundry forge中的Script库
import "../src/Counter.sol"; // 引入要部署的Counter合约

// 部署脚本继承了Script合约
contract CounterScript is Script {
    // 可选函数，在每个函数运行之前被调用
    function setUp() public {}

    // 部署合约时会调用run()函数
    function run() public {
        vm.startBroadcast(); // 开始记录脚本中合约的调用和创建
        new Counter(); // 创建合约
        vm.stopBroadcast(); // 结束记录
    }
}
```

部署命令

```shell
forge script script/Counter.s.sol:CounterScript
```

### test 目录

合约的测试用例

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";        // 引入 forge-std 中用于测试的依赖
import "../src/Counter.sol";        // 引入用于测试的业务合约

// 基于 forge-std 的 test 合约依赖实现测试用例
contract CounterTest is Test {
    Counter public counter;

    // 初始化测试用例
    function setUp() public {
       counter = new Counter();
       counter.setNumber(0);
    }

    // 基于初始化测试用例
    // 断言测试自增后的 counter 的 number 返回值 同等于 1
    function testIncrement() public {
        counter.increment();
        assertEq(counter.number(), 1);
    }

    // 基于初始化测试用例
    // 执行差异测试测试
    // forge 测试的过程中
    // 为 testSetNumber 函数参数传递不同的 unit256 类型的 x
    // 达到测试 counter 的 setNumber 函数 为不同的 x 设置不同的数
    // 断言 number() 的返回值等同于差异测试的 x 参数
    function testSetNumber(uint256 x) public {
        counter.setNumber(x);
        assertEq(counter.number(), x);
    }

    // 差异测试：参考 https://book.getfoundry.sh/forge/differential-ffi-testing
}
```

### 项目构建 & 测试

项目构建

```shell
$ forge build
Compiling 10 files with 0.8.16
Solc 0.8.16 finished in 3.97s
Compiler run successful
```

运行测试

```shell
$ forge test
No files changed, compilation skipped

Running 2 tests for test/Counter.t.sol:CounterTest
[PASS] testIncrement() (gas: 28312)
[PASS] testSetNumber(uint256) (runs: 256, μ: 27376, ~: 28387)
Test result: ok. 2 passed; 0 failed; finished in 24.43ms
```

### 依赖

Forge 使用 [git submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules) 管理依赖项，这意味着它可以与任何包含智能合约的 GitHub 代码库一起使用。

#### 安装依赖

```shell
$ forge install transmissions11/solmate
Installing solmate in "/private/var/folders/p_/xbvs4ns92wj3b9xmkc1zkw2w0000gn/T/tmp.FRH0gNvz/deps/lib/solmate" (url: Some("https://github.com/transmissions11/solmate"), tag: None)
    Installed solmate
```

查看 lib 文件夹

```shell
$ tree lib -L 1
lib
├── forge-std
├── solmate
└── weird-erc20

3 directories, 0 files
```

#### 重新映射依赖项

```shell
$ forge remappings
ds-test/=lib/forge-std/lib/ds-test/src/
forge-std/=lib/forge-std/src/
solmate/=lib/solmate/src/
weird-erc20/=lib/weird-erc20/src/
```

这些重新映射意味着：

- 要从 `forge-std` 导入，我们会这样写：`import "forge-std/Contract.sol";`
- 要从 `ds-test` 导入，我们会这样写：`import "ds-test/Contract.sol";`
- 要从 `solmate` 导入，我们会这样写：`import "solmate/Contract.sol";`
- 要从 `weird-erc20` 导入，我们会这样写：`import "weird-erc20/Contract.sol";`

您可以通过在项目的根目录中创建一个 `remappings.txt` 文件来自定义这些重新映射。

#### 更新依赖

```shell
# 更新solmate依赖项
forge update lib/solmate

# 一次对所有依赖项执行更新
forge update
```

#### 删除依赖

```shell
$ forge remove solmate
# ... 等同于 ...
$ forge remove lib/solmate
```

#### Hardhat 兼容

Forge 还支持基于 Hardhat 的项目，其中依赖项是 npm 包（存储在 `node_modules` 中）并且其合约存储在 `contracts` 中而不是 `src` 中。

要启用 Hardhat 兼容模式，请传递 `--hh` 标志。

## 相关资料

- [官方文档](https://book.getfoundry.sh/)
- [中文文档](https://learnblockchain.cn/docs/foundry/i18n/zh/index.html)
- [Foundry 其他资源](https://github.com/crisgarner/awesome-foundry?tab=readme-ov-file)
