# solidity 开发 Best Practice

## 利用 foundry 开发 solidity 项目

### foundry 初始化

```shell
# 利用foundry初始化整个项目
forge init
```

#### solidity lint

```shell
# 初始化solhint 代码检查工具
solhint --init
```

将`.solhint.json`更新为:

```json
{
  "extends": "solhint:default",
  "rules": {
    "explicit-types": "error",
    "max-states-count": "error",
    "no-console": "error",
    "no-empty-blocks": "error",
    "no-global-import": "error",
    "no-unused-import": "error",
    "no-unused-vars": "error",
    "one-contract-per-file": "error",
    "payable-fallback": "error",
    "reason-string": "error",
    "const-name-snakecase": "error",
    "contract-name-camelcase": "error",
    "event-name-camelcase": "error",
    "func-name-mixedcase": "error",
    "immutable-vars-naming": "error",
    "use-forbidden-name": "error",
    "var-name-mixedcase": "error",
    "imports-on-top": "error",
    "visibility-modifier-order": "error",
    "gas-custom-errors": "error",
    "quotes": "error",
    "avoid-call-value": "error",
    "avoid-low-level-calls": "error",
    "avoid-sha3": "error",
    "avoid-suicide": "error",
    "avoid-throw": "error",
    "avoid-tx-origin": "error",
    "check-send-result": "error",
    "compiler-version": "error",
    "func-visibility": ["error", { "ignoreConstructors": true }],
    "multiple-sends": "error",
    "no-complex-fallback": "error",
    "no-inline-assembly": "error",
    "not-rely-on-block-hash": "error",
    "reentrancy": "error",
    "state-visibility": "error"
  }
}
```

可直接使用[.solhint.json](./.solhint.json)或者根据[solhint 官方](https://protofire.github.io/solhint/)设定符合项目的 lint rules。

### 构建 & 测试

```shell
forge build

forge test
```

```shell
# 查看测试覆盖率，为保障合约安全，建议覆盖率100%
forge coverage
```

### 部署 & 验证

```shell
# 部署同时自动验证合约
forge create --rpc-url <your_rpc_url> \
             --private-key <your_private_key> \
             --etherscan-api-key <your_etherscan_api_key> \  # 用于通过Etherscan，进行合约验证
             --verify \
             src/MyContract.sol:MyContract
```

**注意**：一次只能部署一个合约

## 利用 hardhat 开发 solidity 项目

## hardhat 初始化

```shell
# 在项目空文件夹中
npm init

# 安装hardhat依赖
npm install --save-dev hardhat

# 初始化hardhat项目
npx hardhat
```

[solidity lint](#solidity-lint)

## hardhat 编译 & 测试

```shell
# 编译合约
npx hardhat compile

# 测试合约
npx hardhat test
```

### 合约覆盖率

利用[solidity-coverage](https://github.com/sc-forks/solidity-coverage)

```shell
npm install --save-dev solidity-coverage
```

在`hardhat.config.js`添加 plugin

```json
require('solidity-coverage')
```

```shell
# 查看测试覆盖率，为保障合约安全，建议覆盖率100%
npx hardhat coverage
```

可以通过`coverage/index.html`查看具体结果，例[index.html](index.html)

## hardhat 部署

在`ignition/modules`文件夹下实现对应合约的部署代码，如

```js
const { buildModule } = require("@nomicfoundation/hardhat-ignition/modules");

const JAN_1ST_2030 = 1893456000;
const ONE_GWEI = 1_000_000_000n;

module.exports = buildModule("LockModule", (m) => {
  const unlockTime = m.getParameter("unlockTime", JAN_1ST_2030);
  const lockedAmount = m.getParameter("lockedAmount", ONE_GWEI);

  const lock = m.contract("Lock", [unlockTime], {
    value: lockedAmount,
  });

  return { lock };
});
```

部署:

```shell
npx hardhat ignition deploy ./ignition/modules/Lock.js
```

通过 hardhat 启动一个本地以太坊节点，用于本地测试

```shell
npx hardhat node
```

部署到本地节点

```shell
npx hardhat ignition deploy ./ignition/modules/Lock.js --network localhost
```

通过修改`hardhat.config.js`中的 networks，添加或修改需要部署的区块链网络，如

```json
networks: {
    polygon: {
      url: "https://polygon.llamarpc.com",
      accounts: [POLYGON_API_KEY],
      chainId: 137,
    },
  }
```

部署到 polygon 网络

```shell
npx hardhat ignition deploy ./ignition/modules/Lock.js --network polygon
```
