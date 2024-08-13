# solidity 进阶

## 1. solidity 函数重载

solidity 中允许函数进行重载（`overloading`），即名字相同但输入参数类型不同的函数可以同时存在，它们被视为不同的函数。注意，solidity 不允许修饰器（`modifier`）重载。

例子，定义两个`saySomething()`的函数，一个没有任何参数，输出`"Nothing"`；另一个接收一个`string`参数，输出这个`string`。

```js
function saySomething() public pure returns(string memory) {
    return "Nothing";
}

function saySomething(string memory something) public pure returns(string memory) {
    return something;
}
```

最终重载函数在经过编译器编译后，由于不同的参数类型，都变成了不同的函数选择器（selector）。

### 实参匹配（Argument Matching）

在调用重载函数时，会把输入的实际参数和函数参数的变量类型做匹配。如果出现多个匹配的重载函数，则会报错。

```js
function f(uint8 _in) public pure returns(uint8 out) {
    out = _in;
}

function f(uint256 _in) public pure returns(uint256 out) {
    out = _in;
}
```

调用`f(50)`，因为 50 既可以被转换为`uint8`，也可以被转换为`uint256`，因此会报错。

## 2. solidity 库合约

库合约是一种特殊的合约，为了提升 solidity 代码的复用性和减少 gas 而存在。库合约是一系列的函数合集，由大神或者项目方创作，我们会用即可。

库合约和普通合约主要有几点不同：

1. 不能存在状态变量
2. 不能够继承或被继承
3. 不能接收以太币
4. 不可以被销毁

需注意的是，库合约中的函数可见性如果被设置为`public`或者`external`，则在调用函数时会触发一次`delegatecall`。而如果被设置为`internal`，则不会引起。对于设置为`private`可见性的函数来说，其仅能在库合约中可见，在其他合约中不可用。

### Strings 库合约（`library`）

Strings 库合约是将`uint256`类型转换为相应的`string`类型的代码库，样例代码如下：

```js
library Strings {
    bytes16 private constant _HEX_SYMBOLS = "0123456789abcdef";

    /**
     * @dev Converts a `uint256` to its ASCII `string` decimal representation.
     */
    function toString(uint256 value) public pure returns (string memory) {
        // Inspired by OraclizeAPI's implementation - MIT licence
        // https://github.com/oraclize/ethereum-api/blob/b42146b063c7d6ee1358846c198246239e9360e8/oraclizeAPI_0.4.25.sol

        if (value == 0) {
            return "0";
        }
        uint256 temp = value;
        uint256 digits;
        while (temp != 0) {
            digits++;
            temp /= 10;
        }
        bytes memory buffer = new bytes(digits);
        while (value != 0) {
            digits -= 1;
            buffer[digits] = bytes1(uint8(48 + uint256(value % 10)));
            value /= 10;
        }
        return string(buffer);
    }

    /**
     * @dev Converts a `uint256` to its ASCII `string` hexadecimal representation.
     */
    function toHexString(uint256 value) public pure returns (string memory) {
        if (value == 0) {
            return "0x00";
        }
        uint256 temp = value;
        uint256 length = 0;
        while (temp != 0) {
            length++;
            temp >>= 8;
        }
        return toHexString(value, length);
    }

    /**
     * @dev Converts a `uint256` to its ASCII `string` hexadecimal representation with fixed length.
     */
    function toHexString(uint256 value, uint256 length) public pure returns (string memory) {
        bytes memory buffer = new bytes(2 * length + 2);
        buffer[0] = "0";
        buffer[1] = "x";
        for (uint256 i = 2 * length + 1; i > 1; --i) {
            buffer[i] = _HEX_SYMBOLS[value & 0xf];
            value >>= 4;
        }
        require(value == 0, "Strings: hex length insufficient");
        return string(buffer);
    }
}
```

主要包含两个函数，`toString()`将`uint256`转为`string`，`toHexString()`将`uint256`转换为 16 进制，再转为`string`。

#### 如何使用库合约

1. using for 指令

指令`using A for B;`可用于附加库合约（从库 A）到任何类型 B。添加完指令后，库 A 中的函数会自动添加 B 类型变量的成员，可以直接调用。注意：在调用时，这个变量会被当作第一个参数传递给函数：

```js
// 利用using for指令
using Strings for uint256;
function getString1(uint256 _number) public pure returns(string memory){
    // 库合约中的函数会自动添加为uint256型变量的成员
    return _number.toHexString();
}
```

2. 通过库合约名称调用函数

```js
// 直接通过库合约名调用
function getString2(uint256 _number) public pure returns(string memory){
    return Strings.toHexString(_number);
}
```

### 常用的库合约

1. [Strings](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/4a9cc8b4918ef3736229a5cc5a310bdc17bf759f/contracts/utils/Strings.sol)：将`uint256`转换为`string`
2. [Address](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/4a9cc8b4918ef3736229a5cc5a310bdc17bf759f/contracts/utils/Address.sol)：判断某个地址是否为合约地址
3. [Create2](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/4a9cc8b4918ef3736229a5cc5a310bdc17bf759f/contracts/utils/Create2.sol)：更安全的使用`create2 evm opcode`
4. [Arrays](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/4a9cc8b4918ef3736229a5cc5a310bdc17bf759f/contracts/utils/Arrays.sol)：跟数组相关的库合约

## 3. solidity import

在 solidity 中，`import`可以帮助我们在一个文件中引用另一个文件的内容，提高代码的可重用型和组织性。

### `import`用法

- 通过源文件相对位置导入，例子：

```js
文件结构
├── Import.sol
└── Yeye.sol

// 通过文件相对位置import
import './Yeye.sol';
```

- 通过源文件网址导入网上合约的全局符号，例子：

```js
// 通过网址引用
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Address.sol";
```

- 通过`npm`导入，例子：

```js
import "@openzeppelin/contracts/access/Ownable.sol";
```

- 通过制定全局符号导入合约特定的全局符号，例子：

```js
import { Yeye } from "./Yeye.sol";
```

- 引用 import 在代码中的位置为：在声明版本号之后，在其余代码之前。

## 4. solidity 接收 eth receive 和 fallback

solidity 支持两种特殊的回调函数，`receive()`和`fallback()`，它们主要在两种情况下被使用：

1. 接收 ETH
2. 处理合约中不存在的函数调用（代理合约 proxy contract）

注意：在 solidity 0.6.x 版本之前，语法上只有`fallback()`函数，用来接收用户发送的 ETH 时调用以及在被调用函数签名没有匹配到时，来调用。0.6 版本之后，solidity 才将`fallback()`函数拆分成`receive()`和`fallback()`两个函数。

### 接收 ETH 函数 receive

`receive()`函数是在合约收到 ETH 转账时被调用的函数。一个合约最多有一个`receive()`函数，声明方式与一般函数不一样，不需要`function`关键字：`receive() external payable {...}`。`receive()`函数不能有任何的参数，不能返回任何值，必须包含`external`和`payable`。

当合约接收 ETH 的时候，`receive()`会被触发。`receive()`最好不要执行太多的逻辑因为如果别人用`send()`和`transfer()`方法发送 ETH 的话，gas 会限制在 2300，`receive()`太复杂可能会触发`out of gas`报错；如果用`call`就可以自定义 gas 执行更复杂的逻辑。

```js
// 定义事件
event Received(address Sender, uint Value);
// 接收ETH时释放Received事件
receive() external payable {
    emit Received(msg.sender, msg.value);
}
```

有些恶意合约，会在`receive()`函数（老版本的话，就是`fallback()`函数）中嵌入恶意消耗 gas 的内容或者使得执行故意失败的代码，导致一些包含退款和转账逻辑的合约不能正常工作，因此写包含退款等逻辑的合约时，一定要注意这种情况。

### 回退函数 fallback

`fallback()`函数会在调用合约不存在的函数时被触发。可用于接收 ETH，也可以用于代理合约`proxy contract`。`fallback()`声明时不需要`function`关键字，必须由`external`修饰，一般也会用`payable`修饰，用于接收 ETH：`fallback() external payable {...}`

```js
event fallbackCalled(address Sender, uint Value, bytes Data);

// fallback
fallback() external payable{
    emit fallbackCalled(msg.sender, msg.value, msg.data);
}
```

### receive 和 fallback 的区别

`receive`和`fallback`都能够用于接收 ETH，它们触发的规则如下：

```js
触发fallback() 还是 receive()?
           接收ETH
              |
         msg.data是空？
            /  \
          是    否
          /      \
receive()存在?   fallback()
        / \
       是  否
      /     \
receive()   fallback()
```

简单来说，合约接收 ETH 时，`msg.data`为空且存在`receive()`时，会触发`receive()`；`msg.data`不为空或不存在`receive()`时，会触发`fallback()`，此时`fallback()`必须为`payable`。

`receive()`和`payable fallback()`均不存在的时候，向合约**直接**发送 ETH 将会报错（你仍可以通过带有`payable`的函数向合约发送 ETH）。

## 5. solidity 发送 ETH

solidity 有三种方法向其他合约发送 ETH，分别是：`transfer()`,`send()`,`call()`，其中`call()`是建议使用的方法。

### 接收 ETH 合约

我们先部署一个接收 ETH 的合约`ReceiveETH`。`ReceiveETH`合约里有一个事件`Log`，记录收到的 ETH 数量和 gas 剩余。还有两个函数，一个是`receive()`函数，收到 ETH 被触发，并发送`Log`事件；另一个是查询合约 ETH 余额的`getBalance()`函数。

```js
contract ReceiveETH {
    // 收到eth事件，记录amount和gas
    event Log(uint amount, uint gas);

    // receive方法，接收eth时被触发
    receive() external payable{
        emit Log(msg.value, gasleft());
    }

    // 返回合约ETH余额
    function getBalance() view public returns(uint) {
        return address(this).balance;
    }
}
```

### 发送 ETH 合约

我们将实现三种方法向`ReceiveETH`合约发送 ETH。首先，先在发送 ETH 合约`SendETH`中实现`payable`的构造函数和`receive()`，让我们能够在部署时和部署后向合约转账。

```js
contract SendETH {
    // 构造函数，payable使得部署的时候可以转eth进去
    constructor() payable{}
    // receive方法，接收eth时被触发
    receive() external payable{}
}
```

#### transfer

- 用法是`接收方地址.transfer(发送ETH数额)`
- `transfer()`的 gas 限制是 2300，足够用于转账，但对方合约的`fallback()`或`receive()`函数不能实现太复杂的逻辑。
- `transfer()`如果转账失败，会自动`revert`（回滚交易）

```js
// 用transfer()发送ETH
function transferETH(address payable _to, uint256 amount) external payable{
    _to.transfer(amount);
}
```

#### send

- 用法是`接收方地址.send（发送ETH数额）`
- `send()`的 gas 限制是 2300，足够用于转账，但对方的合约的`fallback()`或`receive()`函数不能实现太复杂的逻辑
- `send()`如果转账失败，不会`revert`
- `send()`的返回值是`bool`类型，代表着转账成功或失败，需要额外代码处理一下

```js
error SendFailed(); // 用send发送ETH失败error

// send()发送ETH
function sendETH(address payable _to, uint256 amount) external payable{
    // 处理下send的返回值，如果失败，revert交易并发送error
    bool success = _to.send(amount);
    if(!success){
        revert SendFailed();
    }
}
```

#### call

- 用法是`接收方地址.call{value: 发送ETH数额}("")`
- `call()`没有 gas 限制，可以支持对方合约`fallback()`或`receive()`函数实现复杂逻辑
- `call()`如果转账失败，不会`revert`
- `call()`的返回值是`(bool, bytes)`，其中`bool`代表转账成功或失败，需要额外代码处理一下

```js
error CallFailed(); // 用call发送ETH失败error

// call()发送ETH
function callETH(address payable _to, uint256 amount) external payable{
    // 处理下call的返回值，如果失败，revert交易并发送error
    (bool success,) = _to.call{value: amount}("");
    if(!success){
        revert CallFailed();
    }
}
```

## 6. solidity 调用其他合约

在 solidity 中，一个合约可以调用另一个合约的函数，这在构建复杂的 DApps 时非常有用。

### 目标合约

先写一个简单的合约`OtherContract`，用于被其他合约调用

```js
contract OtherContract {
    uint256 private _x = 0; // 状态变量_x
    // 收到eth的事件，记录amount和gas
    event Log(uint amount, uint gas);

    // 返回合约ETH余额
    function getBalance() view public returns(uint) {
        return address(this).balance;
    }

    // 可以调整状态变量_x的函数，并且可以往合约转ETH (payable)
    function setX(uint256 x) external payable{
        _x = x;
        // 如果转入ETH，则释放Log事件
        if(msg.value > 0){
            emit Log(msg.value, gasleft());
        }
    }

    // 读取_x
    function getX() external view returns(uint x){
        x = _x;
    }
}
```

这个合约包含一个状态变量`_x`，一个事件`Log`在收到 ETH 时触发，三个函数：

- `getBalance()`：返回合约 ETH 余额
- `setX()`：`external payable`函数，可以设置`_x`的值，并向合约发送 ETH
- `getX()`：读取`_x`的值

### 调用`OtherContract`合约

#### 1. 传入合约地址

我们可以在函数里传入目标合约地址，生成目标合约的引用，然后调用目标函数。以调用`OtherContract`合约的`setX`函数为例：

```js
function callSetX(address _Address, uint256 x) external {
    OtherContract(_Address).setX(x);
}
```

#### 2. 传入合约变量

我们可以直接在函数里传入合约的引用，只需要把上面参数的`address`类型改为目标合约名，比如`OtherContract`。

```js
function callGetX(OtherContract _Address) external view returns(uint x) {
    x = _Address.getX();
}
```

**注意**：该函数参数`OtherContract _Address`底层类型仍然是`address`，生成的 ABI 中，调用`callGetX`时传入的参数都是`address`类型。

#### 3. 创建合约变量

可以创建合约变量，然后通过它来调用目标函数。

```js
function callGetX2(address _Address) external view returns(uint x) {
    OtherContract oc = OtherContract(_Address);
    x = oc.getX();
}
```

#### 4. 调用合约并发送 ETH

如果目标合约的函数是`payable`的，那么我们可以通过调用它来给合约转账：`_Name(_Address).f{value: _Value}()`，其中`_Name`是合约名，`_Address`是合约地址，`f`是目标函数名，`_Value`是要转的 ETH 数额（以 wei 为单位）

```js
function setXTransferETH(address otherContract, uint256 x) payable external {
    OtherContract(otherContract).setX{value: msg.value}(x);
}
```

## 7. solidity 调用合约 Call

`call`是`address`类型的低级成员函数，它用来与其他合约交互。它的返回值为`(bool, bytes memory)`，分别对应`call`是否成功以及目标函数的返回值。

- `call`是 solidity 官方推荐的通过触发`fallback`或`receive`函数发送 ETH 的方法
- 不推荐用`call`来调用另一个合约，因为当你调用不安全合约的函数时，你就把主动权交给了它。推荐方法仍是声明合约变量后调用函数，见[6. solidity 调用其他合约](#6-solidity-调用其他合约)
- 当我们不知道对方合约的源代码或 ABI 时，就没法生成合约变量；这时，我们仍可以通过`call`调用对方合约的函数

### `call`的使用规则

```js
目标合约地址.call(字节码);
```

其中字节码利用结构化编码函数`abi.encodeWithSignature`获得：

```js
abi.encodeWithSignature("函数签名", 逗号分隔的具体参数);
```

例子：`abi.encodeWithSignature("f(uint256,address)", _x, _addr)`

另外，`call`在调用合约时可以指定交易发送的 ETH 数额和 gas 数额：

```js
目标合约地址.call{value: 发送ETH数额, gas: gas数额}(字节码);
```

### 目标合约

一个简单的目标合约`OtherContract`:

```js
contract OtherContract {
    uint256 private _x = 0; // 状态变量x
    // 收到eth的事件，记录amount和gas
    event Log(uint amount, uint gas);

    fallback() external payable{}

    // 返回合约ETH余额
    function getBalance() view public returns(uint) {
        return address(this).balance;
    }

    // 可以调整状态变量_x的函数，并且可以往合约转ETH (payable)
    function setX(uint256 x) external payable{
        _x = x;
        // 如果转入ETH，则释放Log事件
        if(msg.value > 0){
            emit Log(msg.value, gasleft());
        }
    }

    // 读取x
    function getX() external view returns(uint x){
        x = _x;
    }
}
```

这个合约包含一个状态变量`x`，一个在收到 ETH 时触发的事件`Log`，三个函数：

- `getBalance()`：返回合约 ETH 余额
- `setX()`：`external payable`函数，可以设置`x`的值，并向合约发送 ETH
- `getX()`：读取`x`的值

### 利用`call`调用目标合约

#### 1. Response 事件

```js
// 定义Response事件，输出call返回的结果success和data
event Response(bool success, bytes data);
```

#### 2. 调用 setX 函数

```js
function callSetX(address payable _addr, uint256 x) public payable {
    // call setX()，同时可以发送ETH
    (bool success, bytes memory data) = _addr.call{value: msg.value}(
        abi.encodeWithSignature("setX(uint256)", x)
    );

    emit Response(success, data); //释放事件
}
```

#### 3. 调用 getX 函数

```js
function callGetX(address _addr) external returns(uint256){
    // call getX()
    (bool success, bytes memory data) = _addr.call(
        abi.encodeWithSignature("getX()")
    );

    emit Response(success, data); //释放事件
    return abi.decode(data, (uint256));
}
```

#### 4. 调用不存在的函数

如果我们给`call`输入的函数不存在于目标合约，那么目标合约的`fallback`函数会被触发。

```js
function callNonExist(address _addr) external{
    // call 不存在的函数
    (bool success, bytes memory data) = _addr.call(
        abi.encodeWithSignature("foo(uint256)")
    );

    emit Response(success, data); //释放事件
}
```

## 8. solidity Delegatecall

`delegatecall`与`call`类似，都是 solidity 中地址类型的低级成员函数。`delegate`是委托/代表的意思。`delegatecall`表示的是：

当用户 A 通过合约 B 来`call`合约 C 时，执行的是合约 C 的函数，上下文（Context，可以理解为包含变量和状态的环境）也是合约 C 的：`msg.sender`是 B 的地址，并且如果函数改变一些状态变量，产生的效果会作用于合约 C 的变量上。

而当用户 A 通过合约 B 来`delegatecall`合约 C 时，执行的是合约 C 的函数，但是上下文仍然是合约 B 的：`msg.sender`是 A 的地址，并且如果函数改变一些状态变量，产生的效果会作用于合约 B 的变量上。

大家可以这样理解：一个投资者（用户 A）把他的资产（B 合约的状态变量）都交给一个风险投资代理（C 合约）来打理。执行的是风险投资代理的函数，但是改变的是资产的状态。

`delegatecall`语法与`call`类似：

```js
目标合约地址.delegatecall(二进制编码);
```

其中`二进制编码`利用结构化编码函数`abi.encodeWithSignature`获得：

```js
abi.encodeWithSignature("函数签名", 逗号分隔的具体参数);
```

例如：`abi.encodeWithSignature("f(uint256,address)", _x, _addr)`

和`call`不一样，`delegatecall`在调用合约时可以指定交易发送的 gas，但不能指定发送的 ETH 数额。

**注意**：`delegatecall`有安全隐患，使用时要保证当前合约和目标合约的状态变量存储结构相同，并且目标合约安全，不然会造成资产损失

### 什么情况下会用到`delegatecall`？

目前`delegatecall`主要有两个应用场景：

1. 代理合约（`Proxy contract`）：将智能合约的存储合约和逻辑合约分开，代理合约（`Proxy contract`）存储所有相关的变量，并且保存逻辑合约的地址，所有函数存在逻辑合约（`Logic contract`）里，通过`delegatecall`执行。当升级时，只需要将代理合约指向新的逻辑合约即可
2. EIP-2535 Diamonds（钻石）：钻石是一个支持构建可在生产中扩展的模块化智能合约系统的标准。钻石是具有多个实施合约的代理合约。更多信息请查看：[钻石标准简介](https://eip2535diamonds.substack.com/p/introduction-to-the-diamond-standard)

### `delegatecall`例子

调用结构：你（A）通过合约 B 调用合约 C

#### 被调用的合约 C

```js
// 被调用的合约C
contract C {
    uint public num;
    address public sender;

    function setVars(uint _num) public payable {
        num = _num;
        sender = msg.sender;
    }
}
```

#### 发起调用的合约 B

```js
contract B {
    uint public num;
    address public sender;
}

// 通过call来调用C的setVars()函数，将改变合约C里的状态变量
function callSetVars(address _addr, uint _num) external payable{
    // call setVars()
    (bool success, bytes memory data) = _addr.call(
        abi.encodeWithSignature("setVars(uint256)", _num)
    );
}

// 通过delegatecall来调用C的setVars()函数，将改变合约B里的状态变量
function delegatecallSetVars(address _addr, uint _num) external payable{
    // delegatecall setVars()
    (bool success, bytes memory data) = _addr.delegatecall(
        abi.encodeWithSignature("setVars(uint256)", _num)
    );
}
```

## 9. solidity 在合约中创建新合约

在以太坊上，用户（外部账户，EOA）可以创建智能合约，智能合约同样也可以创建新的智能合约。去中心化交易所 uniswap 就是利用工厂合约（`PairFactory`）创建了无数个币对合约（pair）。

### `create`

有两种方法可以在合约中创建新合约，`create`和`create2`

`create`的用法很简单，就是`new`一个合约，并传入新合约构造函数所需的参数：

```js
Contract x = new Contract{value: _value}(params)
```

其中`Contract`是要创建的合约名，`x`是合约对象（地址），如果构造函数是`payable`，可以创建时转入`_value`数量的 ETH，`params`是新合约构造函数的参数。

### 极简 Uniswap

`Uniswap V2`[核心合约](https://github.com/Uniswap/v2-core/tree/master/contracts)中包含两个合约：

1. UniswapV2Pair：币对合约，用于管理币对地址、流动性、买卖
2. UniswapV2Factory：工厂合约，用于创建新的币对，并管理币对地址

下面我们用`create`方法实现一个极简版的`Uniswap`：`Pair`币对合约负责管理币对地址，`PairFactory`工厂合约用于创建新的币对，并管理币对地址。

#### `Pair`合约

```js
contract Pair {
    address public factory; // 工厂合约地址
    address public token0;  // 代币1
    address public token1;  // 代币2

    constructor() payable {
        factory = msg.sender;
    }

    // called once by the factory at time of deployment
    function initialize(address _token0, address _token1) external {
        require(msg.sender == factory, 'UniswapV2: FORBIDDEN'); // sufficient check
        token0 = _token0;
        token1 = _token1;
    }
}
```

`Pair`合约很简单，包含 3 个状态变量：`factory`,`token0`,`token1`

构造函数`constructor`在部署时将`factory`赋值为工厂合约地址。`initialize`函数会由工厂合约在部署完成后手动调用以初始化代币地址，将`token0`和`token1`更新为币对中两种代币的地址。

#### `PairFactory`

```js
contract PairFactory {
    mapping(address => mapping(address => address)) public getPair; // 通过两个代币地址查Pair地址
    address[] public allPairs; // 保存所有Pair地址

    function createPair(address tokenA, address tokenB) external returns(address pairAddr) {
        // 创建新Pair合约
        Pair pair = new Pair();
        // 调用新合约的initialize方法
        pair.initialize(tokenA, tokenB);
        // 更新地址mapping
        pairAddr = address(pair);
        allPairs.push(pairAddr);
        getPair[tokenA][tokenB] = pairAddr;
        getPair[tokenB][tokenA] = pairAddr;
    }
}
```

工厂合约（`PairFactory`）有两个状态变量`getPair`是两个代币地址到币对地址的 mapping，方便根据代币找到币对地址；`allPairs`是币对地址的数组，存储了所有代币对的地址。

`PairFactory`合约只有一个`createPair`函数，根据输入的两个代币地址`tokenA`和`tokenB`来创建新的`Pair`合约。

## 10. solidity Create2

`create2`在智能合约部署在以太坊网络之前就能够预测合约地址

### `create`如何计算地址

智能合约可以由其他合约和普通账户利用`create`操作码创建。在这两种情况下，新合约的地址都以相同的方式计算：创建者的地址（通常为部署的钱包地址或者合约地址）和`nonce`（该地址发送交易的总数，对于合约账户是创建的合约总数，每创建一个合约 nonce+1）的哈希。

```js
新地址 = hash(创建者地址, nonce);
```

创建者地址不会变，但`nonce`可能会随着时间而改变，因此用`create`创建的合约地址不好预测。

### `create2`如何计算地址

`create2`的目的是为了让合约地址独立于未来的时间。不管未来区块链上发生了什么，你都可以把合约部署在事先计算好的地址上。用`create2`创建的合约地址由 4 个部分决定：

- `0xFF`：一个常数，避免和`create`冲突
- `CreatorAddress`：调用`create2`的当前合约（创建合约）地址
- `salt`（盐）：一个创建者指定的`bytes32`类型的值，它的主要目的是用来影响新创建的合约的地址
- `initcode`：新合约的初始字节码（合约的 Creation Code 和构造函数的参数）

```js
新地址 = hash("0xFF", 创建者地址, salt, initcode);
```

### 如何使用`create2`

`create2`的用法和`create`类似，同样是`new`一个合约，并传入新合约构造函数所需的参数，只不过要多传一个`salt`参数：

```js
Contract x = new Contract{salt: _salt, value: _value}(params);
```

其中`Contract`是要创建的合约名，`x`是合约对象（地址），`_salt`是指定的盐；如果构造函数是`payable`，可以创建时转入`_value`数量的 ETH，`params`是新合约构造函数的参数。

### 极简 Uniswap2

#### `Pair`

```js
contract Pair {
    address public factory;
    address public token0;
    address public token1;

    constructor() payable{
        factory = msg.sender;
    }

    function initialize(address _token0, address _token1) external {
        require(msg.sender == factory, 'UniswapV2: FORBIDDEN');
        token0 = _token0;
        token1 = _token1;
    }
}
```

`Pair`合约很简单，包含 3 个状态变量：`factory`,`token0`,`token1`

构造函数`constructor`在部署时将`factory`赋值为工厂合约地址。`initialize`函数会在`Pair`合约创建的时候被工厂合约调用一次，将`token0`和`token1`更新为币对中两种代币的地址。

#### `PairFactory2`

```js
contract PairFactory2{
    mapping(address => mapping(address => address)) public getPair; // 通过两个代币地址查Pair地址
    address[] public allPairs; // 保存所有Pair地址

    function createPair2(address tokenA, address tokenB) external returns (address pairAddr) {
        require(tokenA != tokenB, 'IDENTICAL_ADDRESSES'); //避免tokenA和tokenB相同产生的冲突
        // 用tokenA和tokenB地址计算salt
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA); //将tokenA和tokenB按大小排序
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        // 用create2部署新合约
        Pair pair = new Pair{salt: salt}();
        // 调用新合约的initialize方法
        pair.initialize(tokenA, tokenB);
        // 更新地址map
        pairAddr = address(pair);
        allPairs.push(pairAddr);
        getPair[tokenA][tokenB] = pairAddr;
        getPair[tokenB][tokenA] = pairAddr;
    }
}
```

工厂合约（`PairFactory2`）有两个状态变量`getPair`是两个代币地址到币对地址的`mapping`，方便根据代币找到币对地址；`allPairs`是币对地址的数组，存储了所有币对地址。

`PairFactory2`合约只有一个`createPair2`函数，使用`create2`根据输入的两个代币地址`tokenA`和`tokenB`来创建新的`Pair`合约。

#### 事先计算`Pair`地址

```js
// 提前计算pair合约地址
function calculateAddr(address tokenA, address tokenB) public view returns(address predictedAddress){
    require(tokenA != tokenB, 'IDENTICAL_ADDRESSES'); //避免tokenA和tokenB相同产生的冲突
    // 计算用tokenA和tokenB地址计算salt
    (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA); //将tokenA和tokenB按大小排序
    bytes32 salt = keccak256(abi.encodePacked(token0, token1));
    // 计算合约地址方法 hash()
    predictedAddress = address(uint160(uint(keccak256(abi.encodePacked(
        bytes1(0xff),
        address(this),
        salt,
        keccak256(type(Pair).creationCode)
        )))));
}
```

### create2 的实际应用场景

1. 交易所为新用户预留创建钱包合约地址
2. 由`create2`驱动的`factory`合约，在`Uniswap V2`中交易对的创建是在`factory`中调用`create2`完成。这样的好处是：它可以得到一个确定的`pair`地址，使得`router`中就可以通过`(tokenA, tokenB)`计算出`pair`地址，不再需要执行一次`Factory.getPair(tokenA, tokenB)`的跨合约调用

## 11. solidity 删除合约 `selfdestruct`

`selfdestruct`命令可以用来删除智能合约，并将该合约剩余 ETH 转到指定地址。`selfdestruct`是为了应对合约出错的极端情况而设计的。在[v0.8.18](https://soliditylang.org/blog/2023/02/01/solidity-0.8.18-release-announcement/)版本中，`selfdestruct`关键字被标记为「不再建议使用」，在一些情况下它会导致预期之外的合约语义，但由于目前还没有代替方案，目前只是对开发者做了编译阶段的警告，相关内容可以查看[EIP-6049](https://eips.ethereum.org/EIPS/eip-6049)

然而，在以太坊坎昆（Cancun）升级中，[EIP-6780](https://eips.ethereum.org/EIPS/eip-6780)被纳入升级以实现对 Verkle Tree 更好的支持。EIP-6780 减少了`selfdestruct`功能。根据提案描述，当前`selfdestruct`仅会被用来将合约中的 ETH 转移到指定地址，而原先的删除功能只有在`合约创建-自毁`这两个操作处在同一笔交易时才能生效。所以目前来说：

1. 已经部署的合约无法被`selfdestruct`了
2. 如果要使用原先的`selfdestruct`功能，必须在同一笔交易中创建并自毁

### 如何使用`selfdestruct`

```js
selfdestruct(_addr);
```

其中`_addr`是接收合约中剩余 ETH 的地址。`_addr`地址不需要有`receive()`或`fallback()`也能接收 ETH。

### Demo

以下合约在坎昆升级前可以完成合约的自毁，在坎昆升级后仅能实现内部 ETH 余额的转移。

```js
contract DeleteContract {
    uint public value = 10;

    constructor() payable{}

    receive() external payable{}

    function deleteContract() external {
        selfdestruct(payable(msg.sender));
    }

    function getBalance() external view returns(uint balance) {
        balance = address(this).balance;
    }
}
```

## 12. solidity ABI 编码解码

`ABI`（Application Binary Interface，应用二进制接口）是与以太坊智能合约交互的标准。数据基于它们的类型编码；并且由于编码后不包含类型信息，解码时需要注明它们的类型。

在 solidity 中，ABI 编码有 4 个函数：`abi.encode`,`abi.encodePacked`,`abi.encodeWithSignature`,`abi.encodeWithSelector`。而 ABI 解码有 1 个函数：`abi.decode`，用于解码`abi.encode`的数据。

### ABI 编码

编码 4 个变量，类型分别是`uint256`,`address`,`string`,`uint256[2]`：

```js
uint x = 10;
address addr = 0x7A58c0Be72BE218B41C608b7Fe7C5bB630736C71;
string name = "0xAA";
uint256[2] array = [5, 6];
```

#### `abi.encode`

将给定参数利用[ABI 规则](https://learnblockchain.cn/docs/solidity/abi-spec.html)编码。ABI 被设计出来跟智能合约交互，将每个参数填充为 32 字节的数据，并拼接在一起。如果要和合约交互，要用的就是`abi.encode`

```js
function encode() public view returns(bytes memory result) {
    result = abi.encode(x, addr, name, array);
}
```

编码结果为：

```js
0x000000000000000000000000000000000000000000000000000000000000000a0000000000000000000000007a58c0be72be218b41c608b7fe7c5bb630736c7100000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000005000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000043078414100000000000000000000000000000000000000000000000000000000;
```

由于`abi.encode`将每个数据都填充为 32 字节，中间有很多 0

#### `abi.encodePacked`

将给定参数根据其所需最低空间编码。类似`abi.encode`，但是会把其中填充的很多 0 省略。比如，只用 1 字节来编码`uint8`类型。当你想省空间，并且不与合约交互的时候，可以使用`abi.encodePacked`，例如算一些数据的 hash 时。

```js
function encodePacked() public view returns(bytes memory result) {
    result = abi.encodePacked(x, addr, name, array);
}
```

编码结果为：

```js
0x000000000000000000000000000000000000000000000000000000000000000a7a58c0be72be218b41c608b7fe7c5bb630736c713078414100000000000000000000000000000000000000000000000000000000000000050000000000000000000000000000000000000000000000000000000000000006;
```

由于`abi.encodePacked`对编码进行了压缩，长度比`abi.encode`短很多。

#### `abi.encodeWithSignature`

与`abi.encode`功能类似，只不过第一个参数为`函数签名`，比如`"foo(uint256,address,string,uint256[2])"`。当调用其他合约的时候可以使用。

```js
function encodeWithSignature() public view returns(bytes memory result) {
    result = abi.encodeWithSignature("foo(uint256,address,string,uint256[2])", x, addr, name, array);
}
```

编码的结果为：

```js
0xe87082f1000000000000000000000000000000000000000000000000000000000000000a0000000000000000000000007a58c0be72be218b41c608b7fe7c5bb630736c7100000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000005000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000043078414100000000000000000000000000000000000000000000000000000000;
```

等同于在`abi.encode`编码结果前加上了 4 字节的函数选择器（就是通过函数名和参数进行签名处理【Keccak-Sha3】来标识函数，可以用于不同合约之间的函数调用）。

#### `abi.encodeWithSelector`

与`abi.encodeWithSignature`功能类似，只不过第一个参数为函数选择器，为函数签名的 Keccak 哈希的前 4 个字节。

```js
function encodeWithSelector() public view returns(bytes memory result) {
    result = abi.encodeWithSelector(bytes4(keccak256("foo(uint256,address,string,uint256[2])")), x, addr, name, array);
}
```

编码结果为：

```js
0xe87082f1000000000000000000000000000000000000000000000000000000000000000a0000000000000000000000007a58c0be72be218b41c608b7fe7c5bb630736c7100000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000005000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000043078414100000000000000000000000000000000000000000000000000000000;
```

与`abi.encodeWithSignature`结果一样。

### ABI 解码

#### `abi.decode`

`abi.decode`用于解码`abi.encode`生成的二进制编码，将它还原成原本的参数。

```js
function decode(bytes memory data) public pure returns(uint256 dx, address daddr, string memory, dname, uint256[2] darray) {
    (dx, daddr, dname, darray) = abi.decode(data, (uint256, address, string, uint256[2]));
}
```

### ABI 使用场景

1. 在合约开发中，ABI 常配合 call 来实现对合约的底层调用。

```js
bytes4 selector = contract.getValue.selector;

bytes memory data = abi.encodeWithSelector(selector, _x);
(bool success, bytes memory returnedData) = address(contract).staticcall(data);
require(success);

return abi.decode(returnedData, (uint256));
```

2. ethers.js 中常用 ABI 实现合约的导入和函数调用。

```js
const wavePortalContract = new ethers.Contract(
  contractAddress,
  contractABI,
  signer
);
/*
 * Call the getAllWaves method from your Smart Contract
 */
const waves = await wavePortalContract.getAllWaves();
```

3. 对不开源合约进行反编译后，某些函数无法查到函数签名，可通过 ABI 进行调研。

## 13. solidity 哈希

哈希函数是一个密码学概念，它可以将任意长度的消息转换为一个固定长度的值，这个值也称作哈希（hash）。这里简单介绍一下哈希函数的性质和哈希函数在 solidity 中的应用。

### Hash 的性质

一个好的 hash 函数应该具有以下几个特性：

- 单向性：从输入的消息到它的哈希的正向运算简单且唯一确定，而反过来非常难，通常只能暴力枚举
- 灵敏性：输入的消息改变一点对它的哈希改变很大
- 高效性：从输入的消息到哈希的运算高效
- 均一性：每个哈希值被取到的概率应该基本相等
- 抗碰撞性：
  - 弱抗碰撞性：给定一个消息`x`，找到另一个消息`x'`，使得`hash(x) = hash(x')`是困难的
  - 强抗碰撞性：找到任意`x`和`x'`，使得`hash(x) = hash(x')`是困难的

### Hash 的应用

- 生成数据唯一标识
- 加密签名
- 安全加密

### Keccak256

`Keccak256`函数是 solidity 中最常用的哈希函数，用法非常简单：

```js
哈希 = keccak256(数据);
```

#### Keccak256 和 sha3

1. sha3 由 keccak 标准化而来，在很多场合下 keccak 和 sha3 是同义词，但在 2015 年 8 月 sha3 最终完成标准化时，NIST 调整了填充算法。**所以 sha3 就和 keccak 计算的结果不一样**，这点在实际开发中要注意。
2. 以太坊在开发的时候 sha3 还在标准化，所以才用了 keccak，所以 Ethereum 和 solidity 智能合约代码中的 sha3 指的是 keccak256，而不是标准的 NIST-SHA3，为避免混淆，直接在合约代码中写成 keccak256 是最清晰的。

## 14. solidity 函数选择器 selector

当我们调用智能合约时，本质上是向目标合约发送了一段`calldata`，在 remix 中发送一次交易后，可以在详细信息中看见`input`即为此次交易的`calldata`

发送的`calldata`中前 4 个字节是`selector`（函数选择器）。

### msg.data

`msg.data`是 solidity 中的一个全局变量，值为完整的`calldata`（调用函数时传入的数据）。

```js
// event 返回msg.data
event Log(bytes data);

function mint(address to) external{
    emit Log(msg.data);
}
```

当参数为`0x2c44b726ADF1963cA47Af88B284C06f30380fC78`时，输出的`calldata`为

```js
0x6a6278420000000000000000000000002c44b726adf1963ca47af88b284c06f30380fc78;
```

这段字节码可以分为两部分：

```
前4个字节为函数选择器selector：
0x6a627842

后面32个字节为输入的参数：
0x0000000000000000000000002c44b726adf1963ca47af88b284c06f30380fc78
```

其实`calldata`就是告诉智能合约，我要调用哪个函数以及参数是什么。

### method id、selector 和函数签名

`method id`定义为函数签名的 keccak 哈希后的前 4 个字节，当`selector`与`method id`相匹配时，即表示调用该函数，那么`函数签名`是什么？

函数签名为`"函数名(逗号分隔的参数类型)"`。在同一个智能合约中，不同函数有不同的函数签名，因此我们可以通过函数签名来确定要调用哪个函数。

注意，在函数签名中，`uint`和`int`要写为`uint256`和`int256`。

由于计算`method id`时，需要通过函数名和函数的参数类型来计算。在 Solidity 中，函数的参数类型主要分为：基础类型参数，固定长度类型参数，可变长度类型参数和映射类型参数。

#### 基础类型惨呼时

solidity 中，基础类型的参数有：`uint256(uint8, ... , uint256), bool, address`等。在计算`method id`时，只需要计算`bytes4(keccak256("函数名(参数类型1,参数类型2,...)"))`。例如，如下函数，函数名为`elementaryParamSelector`，参数类型分别为`uint256`和`bool`。所以，只需要计算`bytes4(keccak256("elementaryParamSelector(uint256,bool)"))`便可得到此函数的`method id`。

```js
// elementary（基础）类型参数selector
// 输入：param1: 1，param2: 0
// elementaryParamSelector(uint256,bool) : 0x3ec37834
function elementaryParamSelector(uint256 param1, bool param2) external returns(bytes4 selectorWithElementaryParam){
    emit SelectorEvent(this.elementaryParamSelector.selector);
    return bytes4(keccak256("elementaryParamSelector(uint256,bool)"));
}
```

#### 固定长度类型参数

固定长度的参数类型通常为固定长度的数组，例如：`uint256[5]`等。例如，如下函数`fixedSizeParamSelector的参数为uint256[3]`。因此，在计算该函数的`method id`时，只需要通过`bytes4(keccak256("fixedSizeParamSelector(uint256[3])"))`即可。

```js
// fixed size（固定长度）类型参数selector
// 输入： param1: [1,2,3]
// fixedSizeParamSelector(uint256[3]) : 0xead6b8bd
function fixedSizeParamSelector(uint256[3] memory param1) external returns(bytes4 selectorWithFixedSizeParam){
    emit SelectorEvent(this.fixedSizeParamSelector.selector);
    return bytes4(keccak256("fixedSizeParamSelector(uint256[3])"));
}
```

#### 可变长度类型参数

可变长度参数类型通常为可变长的数组，例如：`address[], uint8[], string`等。例如，如下函数`nonFixedSizeParamSelector`的参数为`uint256[]`和`string`。因此，在计算该函数的`method id`时，只需要通过`bytes4(keccak256("nonFixedSizeParamSelector(uint256[],string)"))`即可。

```js
// non-fixed size（可变长度）类型参数selector
// 输入： param1: [1,2,3]， param2: "abc"
// nonFixedSizeParamSelector(uint256[],string) : 0xf0ca01de
function nonFixedSizeParamSelector(uint256[] memory param1,string memory param2) external returns(bytes4 selectorWithNonFixedSizeParam){
    emit SelectorEvent(this.nonFixedSizeParamSelector.selector);
    return bytes4(keccak256("nonFixedSizeParamSelector(uint256[],string)"));
}
```

#### 映射类型参数

映射类型参数通常有：`contract, enum, struct`等。在计算`method id`时，需要将该类型转化成为 ABI 类型。

例如，如下函数`mappingParamSelector中DemoContract`需要转化为`address`，结构体`User`需要转化为`tuple`类型`(uint256,bytes)`，枚举类型`School`需要转化为`uint8`。因此，计算该函数的`method id`的代码为`bytes4(keccak256("mappingParamSelector(address,(uint256,bytes),uint256[],uint8)"))`。

```js
contract DemoContract {
    // empty contract
}

contract Selector{
    // Struct User
    struct User {
        uint256 uid;
        bytes name;
    }
    // Enum School
    enum School { SCHOOL1, SCHOOL2, SCHOOL3 }
    ...
    // mapping（映射）类型参数selector
    // 输入：demo: 0x9D7f74d0C41E726EC95884E0e97Fa6129e3b5E99， user: [1, "0xa0b1"], count: [1,2,3], mySchool: 1
    // mappingParamSelector(address,(uint256,bytes),uint256[],uint8) : 0xe355b0ce
    function mappingParamSelector(DemoContract demo, User memory user, uint256[] memory count, School mySchool) external returns(bytes4 selectorWithMappingParam){
        emit SelectorEvent(this.mappingParamSelector.selector);
        return bytes4(keccak256("mappingParamSelector(address,(uint256,bytes),uint256[],uint8)"));
    }
    ...
}
```

### 使用 selector

可以利用`selector`来调用目标函数。例如想调用`elementaryParamSelector`函数，只需要理由`abi.encodeWithSelector`将`elementaryParamSelector`函数的`method id`作为`selector`和参数打包编码，传给`call`函数：

```js
// 使用selector来调用函数
function callWithSignature() external{
...
    // 调用elementaryParamSelector函数
    (bool success1, bytes memory data1) = address(this).call(abi.encodeWithSelector(0x3ec37834, 1, 0));
...
}
```

## 15. solidity Try Catch

`try-catch`是现代编程语言几乎都有的处理异常的一种标准方式，solidity0.6 版本也添加了它。

### `try-cath`

在 solidity 中，`try-cath`只能被用于`external`函数或创建合约时`constructor`（被视为`external`函数）的调用。基本语法如下：

```js
try externalContract.f() {
    // call成功的情况下 运行一些代码
} catch {
    // call失败的情况下 运行一些代码
}
```

其中`externalContract.f()`是某个外部合约的函数调用，`try`模块在调用成功的情况下允许，而`catch`模块则在调用失败时运行。

同样可以使用`this.f()`来替代`externalContract.f()`，`this.f()`也被视作为外部调用，但不可在构造函数中使用，因为此时合约还为创建。

如果调用的函数有返回值，那么必须在`try`之后声明`returns(returnType val)`，并且在`try`模块中可以使用返回的变量；如果是创建合约，那么返回值是新创建的合约变量。

```js
try externalContract.f() returns(returnType val){
    // call成功的情况下 运行一些代码
} catch {
    // call失败的情况下 运行一些代码
}
```

另外，`catch`模块支持捕获特殊的异常原因：

```js
try externalContract.f() returns(returnType){
    // call成功的情况下 运行一些代码
} catch Error(string memory /*reason*/) {
    // 捕获revert("reasonString") 和 require(false, "reasonString")
} catch Panic(uint /*errorCode*/) {
    // 捕获Panic导致的错误 例如assert失败 溢出 除零 数组访问越界
} catch (bytes memory /*lowLevelData*/) {
    // 如果发生了revert且上面2个异常类型匹配都失败了 会进入该分支
    // 例如revert() require(false) revert自定义类型的error
}
```

### `try-catch`实战

创建一个外部合约`OnlyEven`，并使用`try-catch`来处理异常：

```js
contract OnlyEven{
    constructor(uint a){
        require(a != 0, "invalid number");
        assert(a != 1);
    }

    function onlyEven(uint256 b) external pure returns(bool success){
        // 输入奇数时revert
        require(b % 2 == 0, "Ups! Reverting");
        success = true;
    }
}
```

`OnlyEven`合约包含一个构造函数和一个`onlyEven`函数。

- 构造函数有一个参数`a`，当`a=0`时，`require`会抛出异常；当`a=1`时，`assert`会抛出异常；其他情况均正常。
- `onlyEven`函数有一个参数`b`，当`b`为奇数时，`require`会抛出异常。

#### 处理外部函数调用异常

首先，在`try-catch`合约中定义一些事件和状态变量：

```js
// 成功event
event SuccessEvent();

// 失败event
event CatchEvent(string message);
event CatchByte(bytes data);

// 声明OnlyEven合约变量
OnlyEven even;

constructor() {
    even = new OnlyEven(2);
}
```

`SuccessEvent`是调用成功会释放的事件，而`CatchEvent`和`CatchByte`是抛出异常时会释放的事件，分别对应`require/revert`和`assert`异常的情况。`even`是个`OnlyEven`合约类型的状态变量。

然后我们在`execute`函数中使用`try-catch`处理调用外部函数`onlyEven`中的异常：

```js
// 在external call中使用try-catch
function execute(uint amount) external returns (bool success) {
    try even.onlyEven(amount) returns(bool _success){
        // call成功的情况下
        emit SuccessEvent();
        return _success;
    } catch Error(string memory reason){
        // call不成功的情况下
        emit CatchEvent(reason);
    }
}
```

#### 处理合约创建异常

这里，我们利用`try-catch`来处理合约创建时的异常。只需要把`try`模块改写为`OnlyEven`合约的创建就行：

```js
// 在创建新合约中使用try-catch （合约创建被视为external call）
// executeNew(0)会失败并释放`CatchEvent`
// executeNew(1)会失败并释放`CatchByte`
// executeNew(2)会成功并释放`SuccessEvent`
function executeNew(uint a) external returns (bool success) {
    try new OnlyEven(a) returns(OnlyEven _even){
        // call成功的情况下
        emit SuccessEvent();
        success = _even.onlyEven(a);
    } catch Error(string memory reason) {
        // catch失败的 revert() 和 require()
        emit CatchEvent(reason);
    } catch (bytes memory reason) {
        // catch失败的 assert()
        emit CatchByte(reason);
    }
}
```
