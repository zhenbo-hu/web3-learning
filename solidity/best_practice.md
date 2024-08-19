# solidity 开发 Best Practice

## 利用 foundry 开发 solidity 项目

### 项目初始化

```shell
# 利用foundry初始化整个项目
forge init
```

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
