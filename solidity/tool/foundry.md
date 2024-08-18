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

## Foundry 测试

Forge 可以使用 `forge test` 命令运行测试（用例）。 所有测试都是用 Solidity 编写的。

Forge 将在您的源目录中的任何位置查找测试。 任何具有以`test`开头的函数的合约都被认为是一个测试。 通常，测试将按照约定放在 `test` 中，并以 `.t.sol` 结尾。

下面是在新创建的项目中运行 `forge test` 的示例，该项目只有默认测试：

```shell
$ forge test
No files changed, compilation skipped

Running 2 tests for test/Counter.t.sol:CounterTest
[PASS] testIncrement() (gas: 28312)
[PASS] testSetNumber(uint256) (runs: 256, μ: 27376, ~: 28387)
Test result: ok. 2 passed; 0 failed; finished in 24.43ms
```

可以通过传递过滤器来运行特定测试：

```shell
$ forge test --match-contract ComplicatedContractTest --match-test testDeposit
Compiling 7 files with 0.8.10
Solc 0.8.10 finished in 4.20s
Compiler run successful

Running 2 tests for test/ComplicatedContract.t.sol:ComplicatedContractTest
[PASS] testDepositERC20() (gas: 102237)
[PASS] testDepositETH() (gas: 61458)
Test result: ok. 2 passed; 0 failed; finished in 1.05ms
```

这将在名称中带有 `testDeposit` 的 `ComplicatedContractTest` 测试合约中运行测试。 这些标志的反向版本也存在（`--no-match-contract` 和 `--no-match-test`）。

您可以使用 `--match-path` 与 `glob` 模式匹配的文件名中运行测试。

```shell
$ forge test --match-path test/ContractB.t.sol
No files changed, compilation skipped

Running 1 test for test/ContractB.t.sol:ContractBTest
[PASS] testExample() (gas: 257)
Test result: ok. 1 passed; 0 failed; finished in 492.35µs
```

`--match-path` 标志的反面是 `--no-match-path`。

### 日志和跟踪

`forge test` 的默认行为是只显示通过和失败测试的摘要。 您可以通过增加详细程度（使用`-v`标志）来控制此行为。 每个详细级别都会添加更多信息：

级别 2 (`-vv`)：还会显示测试期间发出的日志。 这包括来自测试的断言错误，显示诸如预期与实际结果等之类的信息。
级别 3 (`-vvv`)：还显示失败测试的堆栈跟踪。
级别 4 (`-vvvv`)：显示所有测试的堆栈跟踪，并显示失败测试的设置（setup）跟踪。
级别 5 (`-vvvvv`)：始终显示堆栈跟踪和设置（setup）跟踪。

### Watch 模式

当您使用`forge test --watch`对文件进行更改时，Forge 可以重新运行您的测试。

默认情况下，仅重新运行更改的测试文件。 如果你想重新运行更改的所有测试，你可以使用 `forge test --watch --run-all`。

### 编写测试

测试是用 Solidity 编写的。 如果测试功能 `revert`，则测试失败，否则通过。

让我们回顾一下最常见的编写测试的方式，使用 Forge 标准库(forge-std) 的 `Test` 合约，这是使用 Forge 编写测试的首选方式。

在本节中，我们将使用 Forge Std 的 `Test` 合约中的函数复习基础知识，该合约本身是 DSTest 的超集。 很快您将学习如何使用 Forge 标准库中的更多高级内容 。

DSTest 提供基本的日志记录和断言功能。 要访问这些函数，请导入 `forge-std/Test.sol` 并继承自测试合约 `Test` ：

```solidity
import "forge-std/Test.sol";
```

一个基本测试

```solidity
pragma solidity 0.8.10;

import "forge-std/Test.sol";

contract ContractBTest is Test {
    uint256 testNumber;

    function setUp() public {
        testNumber = 42;
    }

    function testNumberIs42() public {
        assertEq(testNumber, 42);
    }

    function testFailSubtract43() public {
        testNumber -= 43;
    }
}
```

Forge 在测试中使用以下关键字：

- `setUp`：在每个测试用例运行之前调用的可选函数

```solidity
    function setUp() public {
        testNumber = 42;
    }
```

- `test`：以 `test` 为前缀的函数作为测试用例运行

```solidity
    function testNumberIs42() public {
        assertEq(testNumber, 42);
    }
```

- `testFail`: `test` 前缀的测试的反面 - 如果函数没有 `revert`，则测试失败

```solidity
    function testFailSubtract43() public {
        testNumber -= 43;
    }
```

一个好的实践是结合 `expectRevert` cheatcode 来使用 `test_Revert[If|When]\_Condition` 模式。现在，不再使用 `testFail`，您可以确切地知道发生了什么并且出现了哪些错误：

```solidity
    function testCannotSubtract43() public {
        vm.expectRevert(stdError.arithmeticError);
        testNumber -= 43;
    }
```

测试合约部署到 `0xb4c79daB8f259C7Aee6E5b2Aa729821864227e84`。 如果您在测试中部署合约，则 `0xb4c...7e84` 将是它的部署者。 如果在测试中部署的合约向其部署者授予特殊权限， 例如 `Ownable.sol` 的 `onlyOwner` 修饰符，那么测试合约 `0xb4c...7e84` 将具有这些权限。

**注意**: 测试函数必须具有`external`或`public`可见性。 声明为`internal`或 `private` 不会被 Forge 选中，即使它们以 `test` 为前缀。

### 共享设置

可以通过创建辅助抽象合约并在测试合约中继承它们来使用共享设置：

```solidity
abstract contract HelperContract {
    address constant IMPORTANT_ADDRESS = 0x543d...;
    SomeContract someContract;
    constructor() {...}
}

contract MyContractTest is Test, HelperContract {
    function setUp() public {
        someContract = new SomeContract(0, IMPORTANT_ADDRESS);
        ...
    }
}

contract MyOtherContractTest is Test, HelperContract {
    function setUp() public {
        someContract = new SomeContract(1000, IMPORTANT_ADDRESS);
        ...
    }
}
```

**提示**: 可使用 `getCode` 作弊码部署具有不兼容 Solidity 版本的合约。

### 作弊码（Cheatcodes）

大多数时候，仅仅测试您的智能合约输出是不够的。 为了操纵区块链的状态，以及测试特定的 `reverts` 和事件 `Events`，Foundry 附带了一组作弊码（Cheatcodes)。

作弊码允许您更改区块号、您的身份（地址）等。 它们是通过在特别指定的地址上 （`0x7109709ECfa91a80626fF3989D68f67F5b1DD12D`）调用特定函数来调用的。

您可以通过 Forge 标准库的 `Test` 合约中提供的 `vm` 实例轻松访问作弊码。

让我们为验证只能由所有者调用的智能合约编写一个测试。

```solidity
pragma solidity 0.8.10;

import "forge-std/Test.sol";

error Unauthorized();

contract OwnerUpOnly {
    address public immutable owner;
    uint256 public count;

    constructor() {
        owner = msg.sender;
    }

    function increment() external {
        if (msg.sender != owner) {
            revert Unauthorized();
        }
        count++;
    }
}

contract OwnerUpOnlyTest is Test {
    OwnerUpOnly upOnly;

    function setUp() public {
        upOnly = new OwnerUpOnly();
    }

    function testIncrementAsOwner() public {
        assertEq(upOnly.count(), 0);
        upOnly.increment();
        assertEq(upOnly.count(), 1);
    }
}
```

如果我们现在运行 `forge test`，我们将看到测试通过，因为 `OwnerUpOnlyTest` 是 `OwnerUpOnly` 的所有者（owner）。

```shell
$ forge test
Compiling 7 files with 0.8.10
Solc 0.8.10 finished in 4.25s
Compiler run successful

Running 1 test for test/OwnerUpOnly.t.sol:OwnerUpOnlyTest
[PASS] testIncrementAsOwner() (gas: 29162)
Test result: ok. 1 passed; 0 failed; finished in 928.64µs
```

让我们确保绝对不是所有者的人不能增加计数：

```solidity
contract OwnerUpOnlyTest is Test {
    OwnerUpOnly upOnly;

        // ...

    function testFailIncrementAsNotOwner() public {
        vm.prank(address(0));
        upOnly.increment();
    }
}
```

如果我们现在运行 `forge test`，我们将看到所有测试都通过了。

```shell
$ forge test
No files changed, compilation skipped

Running 2 tests for test/OwnerUpOnly.t.sol:OwnerUpOnlyTest
[PASS] testFailIncrementAsNotOwner() (gas: 8413)
[PASS] testIncrementAsOwner() (gas: 29162)
Test result: ok. 2 passed; 0 failed; finished in 1.03ms
```

测试通过是因为 `prank` 作弊码将我们的身份更改为零地址再进行下一次调用 (`upOnly.increment()`)。 由于我们使用了 `testFail` 前缀，测试用例通过了，但是，使用 `testFail` 被认为是一种反模式(anti-pattern)，因为它没有告诉我们任何关于为什么 `upOnly.increment()` 被 `revert` 的信息。

如果我们在启用跟踪的情况下再次运行测试，我们可以看到 revert 了正确的错误消息。

```shell
$ forge test -vvvv --match-test testFail_IncrementAsNotOwner
No files changed, compilation skipped

Running 1 test for test/OwnerUpOnly.t.sol:OwnerUpOnlyTest
[PASS] testFailIncrementAsNotOwner() (gas: 8413)
Traces:
  [8413] OwnerUpOnlyTest::testFailIncrementAsNotOwner()
    ├─ [0] VM::prank(0x0000000000000000000000000000000000000000)
    │   └─ ← ()
    ├─ [247] 0xce71…c246::increment()
    │   └─ ← 0x82b42900
    └─ ← 0x82b42900

Test result: ok. 1 passed; 0 failed; finished in 2.01ms
```

为了将来的确定性，让我们确保我们 `revert` 了，可以使用 `expectRevert` 作弊码来验证我们不是所有者的情况：

```solidity
contract OwnerUpOnlyTest is Test {
    OwnerUpOnly upOnly;

     // ...

    // Notice that we replaced `testFail` with `test`
    function testIncrementAsNotOwner() public {
        vm.expectRevert(Unauthorized.selector);
        vm.prank(address(0));
        upOnly.increment();
    }
}
```

如果我们最后再一次运行 `forge test`，我们会看到测试仍然通过，但这次我们确信如果因为任何其他原因 `revert` 时，它将总是会失败。

```shell
$ forge test
No files changed, compilation skipped

Running 2 tests for test/OwnerUpOnly.t.sol:OwnerUpOnlyTest
[PASS] testIncrementAsNotOwner() (gas: 8739)
[PASS] testIncrementAsOwner() (gas: 29162)
Test result: ok. 2 passed; 0 failed; finished in 1.15ms
```

另一个可能不那么直观的作弊码是 `expectEmit` 函数。 在查看 `expectEmit` 之前，我们需要了解什么是事件（`Event`）。

事件是合约的可继承成员。 当触发出事件时，参数存储在区块链上。 `indexed` 属性最多可以添加到事件三个参数中，以形成称为 “主题（topic）” 的数据结构。 主题允许用户搜索（过滤）区块链上的事件。

```solidity
pragma solidity 0.8.10;

import "forge-std/Test.sol";

contract EmitContractTest is Test {
    event Transfer(address indexed from, address indexed to, uint256 amount);

    function testExpectEmit() public {
        ExpectEmit emitter = new ExpectEmit();
        // Check that topic 1, topic 2, and data are the same as the following emitted event.
        // Checking topic 3 here doesn't matter, because `Transfer` only has 2 indexed topics.
        vm.expectEmit(true, true, false, true);
        // The event we expect
        emit Transfer(address(this), address(1337), 1337);
        // The event we get
        emitter.t();
    }

    function testExpectEmitDoNotCheckData() public {
        ExpectEmit emitter = new ExpectEmit();
        // Check topic 1 and topic 2, but do not check data
        vm.expectEmit(true, true, false, false);
        // The event we expect
        emit Transfer(address(this), address(1337), 1338);
        // The event we get
        emitter.t();
    }
}

contract ExpectEmit {
    event Transfer(address indexed from, address indexed to, uint256 amount);

    function t() public {
        emit Transfer(msg.sender, address(1337), 1337);
    }
}
```

当我们调用 `vm.expectEmit(true, true, false, true);` 时，我们想要检查下一个事件的第一个和第二个 `indexed` 主题。

`test_ExpectEmit()` 中预期的 `Transfer` 事件意味着我们期望 `from` 是 `address(this)`，而 `to` 是 `address(1337)`。 这与从 `emitter.t()` 发出的事件进行比较。

换句话说，我们正在检查来自 `emitter.t()` 的第一个主题是否等于 `address(this)` 。 `expectEmit` 中的第三个参数设置为 `false`，因为不需要检查 `Transfer` 事件中的第三个主题，因为只有两个主题。 即使我们设置为 `true` 也没关系。

`expectEmit` 中的第 4 个参数设置为 `true`，这意味着我们要检查 "non-indexed topics（非索引主题）"，也称为数据。

例如，我们希望来自 `test_ExpectEmit` 中预期事件的数据（即 `amount`）等于实际发出的事件中的数据。 换句话说，我们断言 `emitter.t()` 发出的 `amount` 等于 `1337`。 如果 `expectEmit` 中的第四个参数设置为 `false`，我们将不会检查 `amount`。

换句话说，`test_ExpectEmit_DoNotCheckData` 是一个有效的测试用例，即使数量不同，因为我们不检查数据。

## 相关资料

- [官方文档](https://book.getfoundry.sh/)
- [中文文档](https://learnblockchain.cn/docs/foundry/i18n/zh/index.html)
- [Foundry 其他资源](https://github.com/crisgarner/awesome-foundry?tab=readme-ov-file)
