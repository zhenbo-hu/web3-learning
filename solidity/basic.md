<!-- trunk-ignore-all(prettier) -->
# solidity 基础

## 1. solidity 值类型

1. 值类型（Value Type）：包括布尔型，整型等，这类变量赋值时直接传递数值
2. 引用类型（Reference Type）：包括数组和结构体，这类变量占用空间大，赋值时直接传递地址（类似C/C++中的指针）
3. 映射类型（Mapping Type）：solidity中存储键值对的数据结构，可以理解为哈希表

### 1.1 值类型

#### 1.1.1 布尔型

布尔型取值为`true`或`false`

布尔值运算符

- `!` 逻辑非
- `&&` 逻辑与
- `||` 逻辑或
- `==` 等于
- `!=` 不等于

```solidity
bool public _bool = true;

bool public _bool1 = ! _bool; // false
bool public _bool2 = _bool && _bool1; // false
bool public _bool3 = _bool || _bool1; // true
bool public _bool4 = _bool == _bool1; // false
bool public _bool5 = _bool != _bool1; // true
```

#### 1.1.2 整型

整型包括`int`,`uint`,`uint256`

整型运算符

- 比较运算符（返回值为布尔型）：`<=`, `<`, `==`, `!=`, `>=`, `>`
- 算数运算符：`+`, `-`, `*`, `/`, `%`, `**`

```solidity
int public _int = -1;
uint public _uint = 1;
uint256 public _number = 20220330;

uint256 public _number1 = _number + 1;
uint256 public _number2 = 2**2;
uint256 public _number3 = 7 % 2;
bool public _number_bool = _number2 > _number3;
```

#### 1.1.3 地址类型 address

地址类型有两类

- address：存储一个20字节的值（以太坊地址的大小）
- payable address：比普通地址多了`transfer`和`send`两个成员方法，用于接收转账

```solidity
address public _address = 0x7A58c0Be72BE218B41C608b7Fe7C5bB630736C71;
address payable public _address1 = payable(_address);

uint256 public balance = _address1.balance;
```

#### 1.1.4 定长字节数组

字节数组分为定长和不定长两种：

- 定长字节数组：属于值类型，数组长度声明后不能改变。根据字节数组的长度分为`bytes1`,`bytes8`,`bytes32`等类型。定长字节数组最多存储32字节的数据，即`bytes32`。
- 不定长字节数组：属于引用类型，数组长度在声明后可以改变，后续介绍

```solidity
uint[8] array1;
bytes1[5] array2;
address[100] array3;
```

#### 1.1.5 枚举 `enum`

枚举是用于用户定义的数据类型。主要用于为`uint`分配名称，使程序易于阅读和维护。与C/C++中的`enum`类型，使用名称来代替从0开始的`uint`。

枚举可以显示的与`uint`相互转换，并会检查转换的数值是否在枚举的长度内，否则会报错。

```solidity
enum ActionSet {Buy, Hold, Sell}

ActionSet action = ActionSet.Buy;

function enumToUint() external view returns(uint) {
    return uint(action);
}
```

### 1.2 引用类型

#### 1.2.1 可变长度数组 array

可变长度数组（动态数组）：在声明时不指定长度

```solidity
uint[] array1;
bytes[] array2;
address[] array3;
bytes array4;
```

注意： `bytes`比较特殊，是数组，但不用加`[]`。另外，不能用`byte[]`声明单字节数组，可以使用`bytes`或`bytes1[]`。`bytes`比`bytes1[]`省gas。

##### 创建数组的规则

- 对于`memory`修饰的动态数组，可以用`new`操作符来创建，但是必须声明长度，且长度声明后不能修改。

```solidity
uint[] memory array5 = new uint[](5);
bytes memory array6 = new bytes(9);
```

## 2. solidity 函数可见性 `internal|external|public|private`

**`public`**：内部和外部均可见。

**`private`**：只能从本合约内部访问，继承的合约也不能使用。

**`external`**：只能从合约外部访问（但内部可以通过 `this.f()` 来调用，f是函数名）。

**`internal`**: 只能从合约内部访问，继承的合约可以用。

**注意 1**：合约中定义的函数需要明确指定可见性，它们没有默认值。

**注意 2**：`public|private|internal` 也可用于修饰状态变量。`public`变量会自动生成同名的`getter`函数，用于查询数值。未标明可见性类型的状态变量，默认为`internal`。

### Code

```solidity
// internal function
function minus() internal {
    number = number - 1;
}

// external function: 可以调用internal function
function minusCall() external {
    minus();
}
```

## 3. solidity 函数关键字 `pure`和`view`

solidity 引入这两个关键字主要是因为 **以太坊交易需要支付gas fee**。合约的状态变量存储在链上，gas fee 很贵，如果计算不改变链上状态，就可以不用付 gas。包含 `pure` 和 `view` 关键字的函数是不改写链上状态的，因此用户直接调用它们是不需要付 gas 的（注意，合约中非 `pure/view` 函数调用 `pure/view` 函数时需要付gas）。

在以太坊中，以下语句被视为修改链上状态：

- 写入状态变量。
- 释放事件。
- 创建其他合约。
- 使用 `selfdestruct`.
- 通过调用发送以太币。
- 调用任何未标记 `view` 或 `pure` 的函数。
- 使用低级调用（low-level calls）。
- 使用包含某些操作码的内联汇编。

`pure`函数既不能读取也不能写入链上的状态变量

`view`函数能读取链上的状态变量，但不能写入状态变量

非`pure/view`函数既可读取也可写入状态变量

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract PureAndView{
    uint256 public number = 5;

    // 默认function
    function add() external {
        number = number + 1;
    }

    // pure function
    function addPure(uint256 _number) external pure returns(uint256 new_number) {
        new_number = _number + 1;
    }

    // view function 
    function addView() external returns(uint256 new_number) {
        new_number = number + 1;
    }
}

```

## 4. solidity 函数关键字 `payable`

`payable`用于指示函数或合约可以接受以太币（Ether）或发送以太币

`payable`含义和用法：

- 合约接收以太币：使用`payable`声明的函数可以接收以太币作为函数调用的一部分。这使得其他地址可以向合约发送以太币，从而实现资金的存储和管理
- 合约发送以太币：使用`address`类型的变量可以调用`.transfer()`或`.send()`方法向其他地址发送以太币。当向一个合约地址发送以太币时，该合约需要使用payable修饰的函数来接收和处理传入的以太币

```solidity
function storeValue() public payable {
    require(msg.value > 0, "Must send some Ether to store the value");
    storeValue += msg.value;
}

function sendValue(address payable recipient) public {
    require(storeValue > 0, "No store value to send");
    recipient.transfer(storeValue);
    storeValue = 0;
}
```
