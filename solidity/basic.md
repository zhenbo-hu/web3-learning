<!-- trunk-ignore-all(prettier) -->
# solidity 基础

## 1. solidity 类型

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

```js
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

```js
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

```js
address public _address = 0x7A58c0Be72BE218B41C608b7Fe7C5bB630736C71;
address payable public _address1 = payable(_address);

uint256 public balance = _address1.balance;
```

#### 1.1.4 定长字节数组

字节数组分为定长和不定长两种：

- 定长字节数组：属于值类型，数组长度声明后不能改变。根据字节数组的长度分为`bytes1`,`bytes8`,`bytes32`等类型。定长字节数组最多存储32字节的数据，即`bytes32`。
- 不定长字节数组：属于引用类型，数组长度在声明后可以改变，后续介绍

```js
uint[8] array1;
bytes1[5] array2;
address[100] array3;
```

#### 1.1.5 枚举 `enum`

枚举是用于用户定义的数据类型。主要用于为`uint`分配名称，使程序易于阅读和维护。与C/C++中的`enum`类型，使用名称来代替从0开始的`uint`。

枚举可以显示的与`uint`相互转换，并会检查转换的数值是否在枚举的长度内，否则会报错。

```js
enum ActionSet {Buy, Hold, Sell}

ActionSet action = ActionSet.Buy;

function enumToUint() external view returns(uint) {
    return uint(action);
}
```

### 1.2 引用类型

#### 1.2.1 可变长度数组 array

可变长度数组（动态数组）：在声明时不指定长度

```js
uint[] array1;
bytes[] array2;
address[] array3;
bytes array4;
```

注意： `bytes`比较特殊，是数组，但不用加`[]`。另外，不能用`byte[]`声明单字节数组，可以使用`bytes`或`bytes1[]`。`bytes`比`bytes1[]`省gas。

##### 创建数组的规则

- 对于`memory`修饰的动态数组，可以用`new`操作符来创建，但是必须声明长度，且长度声明后不能修改。

```js
uint[] memory array5 = new uint[](5);
bytes memory array6 = new bytes(9);
```

- 数组字面常熟（Array Literals）是写作表达式形式的数组，用方括号包着来初始化array的一种方式，并且里面每一个元素的type是以第一个元素为准的，例如`[1,2,3]`里面所有的元素都是`uint8`类型，因为在solidity中，如果一个值没有指定type的话，会根据上下文推断出元素的类型，默认就是最小单位的type。而`[uint(1),2,3]`里面的元素都是`uint`类型，因为第一个元素指定是`uint`类型了。

- 如果创建的是动态数组，需要一个一个元素的赋值

```js
uint[] memory x = new uint[](3);
x[0] = 1;
x[1] = 10;
x[2] = 100;
```

##### 数组成员

- `length`：数组有一个包含元素数量的`length`成员，`memory`数组的长度在创建后是固定的。
- `push()`：动态数组拥有 `push()`成员函数，可以在数组最后添加一个`0`元素，并返回该元素的引用。
- `push(x)`：动态数组拥有`push(x)`成员函数，可以在数组最后添加一个`x`元素。
- `pop()`：动态数组拥有`pop()`成员函数，可以移除数组最后一个元素。

#### 1.2.2 结构体 struct

solidity支持通过结构体的形式定义新的类型。结构体中的元素可以是原始类型，也可以是引用类型；结构体可以作为数组或映射的元素。创建结构体的方法：

```js
struct Student{
    uint256 id;
    uint256 score;
}

Student student;
```

结构体赋值的四种方法：

```js
// 方法1:在函数中创建一个storage的struct引用
function initStudent1() external {
    Student storage _student = student;
    _student.id = 1;
    _student.score = 100;
}

// 方法2:直接饮用状态变量的struct
function initStudent2() external {
    student.id = 2;
    student.score = 80;
}

// 方法3:构造函数式
function initStudent3() external {
    student = Student(3, 90);
}

// 方法4:key value形式
function initStudent4() external {
    student = Student({id: 4, score: 70});
}
```

### 1.3 映射类型 mapping

在映射中，可以通过键（`key`）来快速查询到对应的值（`value`），比如：通过一个人的`id`来查询他的钱包地址。

声明映射的格式为`mapping(_KeyType => _ValueType)`，其中`_KeyType`和`_ValueType`分别是`Key`和`Value`的变量类型。

```js
mapping(uint => address) public idToAddress;
mapping(address => uint) public swapPair;
```

#### 1.3.1 映射的规则

- 规则1:映射的`_KeyType`只能选择solidity的内置类型，比如`uint`,`address`等，不能用自定义的结构体。而`_ValueType`可以使用自定义的类型。下面的例子会报错，因为`_KeyValue`使用了自定义的结构体：

```js
struct Student {
    uint256 id;
    uint256 score;
}

mapping(Student => uint) public tmpMapping;
```

- 规则2:映射的存储位置必须是`storage`，因此可以用于合约的状态变量，函数中的`storage`变量和library函数的参数。不能用于`public`函数的参数或返回结果中，因为`mapping`记录的是一种关系（key-value pair）。
- 规则3:如果映射声明为`public`，那么solidity会自动创建一个`getter`函数，可以通过`key`来查询对应的`value`。
- 规则4:给映射新增键值对的语法为`_var[key] = _value`，其中`_var`是映射名，`_key`和`_value`对应新增的键值对。例子：

```js
function writeMap(uint _key, address _value) public {
    idToAddress[_key] = _value;
}
```

#### 1.3.2 映射的原理

- 原理1:映射不存储任何键`key`的信息，也没有`length`信息。
- 原理2:映射使用`keccak256(abi.encodePacked(key, slot))`当成offset存取`value，其中`slot`是映射变量定义所在的插槽位置。
- 原理3:因为Ethereum会定义所有未使用的空间为0，所以为赋值(`value`)的键(`key`)初始值都是各个type的默认值，如uint的默认值是0.

### 1.4 变量初始值

在solidity中，声明但没有赋值的变量都有它的初始值或默认值。

#### 1.4.1 值类型的初始值

- `bool`: `false`
- `string`: `''`
- `int`: `0`
- `uint`: `0`
- `enum`: 枚举中的第一个元素
- `address`: `0x0000000000000000000000000000000000000000`或`address(0)`
- `function`: 空白函数（`internal`, `external`）

#### 1.4.2 引用类型的初始值

- 映射`mapping`:所有元素都为其默认值的`mapping`
- 结构体`struct`:所有成员设为其默认值的结构体
- 数组`array`
  - 动态数组:`[]`
  - 静态数组(定长):所有成员设为其默认值的静态数组

#### 1.4.3 `delete`操作符

`delete a`会让变量`a`的值变为初始值

### 1.5 常数 `constant`和`immutable`

`constant`(常数)，`immutable`(不变量)。状态变量声明时使用这两个关键字之后，不能在初始化后更改数值。好处是提升合约的安全性并节省gas。

只有数值变量可以声明为`constant`和`immutable`；`string`和`bytes`可以声明为`constant`，不能声明为`immutable`.

#### 1.5.1 `constant`

`constant`变量必须在声明时初始化，之后再也不能改变。如果尝试改变，编译不通过。

```js
uint256 constant CONSTANT_NUM = 10;
string constant CONSTANT_STRING = "abcd";
bytes constant CONSTANT_BYTES = "web3";
address constant CONSTANT_ADDRESS = 0x0000000000000000000000000000000000000000
```

#### 1.5.2 `immutable`

`immutable`变量可以在声明时或构造函数中初始化，因此更加灵活。若`immutable`变量既在声明时初始化，又在`constructor`中初始化，会使用`constructor`中初始化的值。

```js
uint256 public immutable IMMUTABLE_NUM = 99999;
address public immutable IMMUTABLE_ADDRESS;

constructor() {
    IMMUTABLE_NUN = 100;
    IMMUTABLE_ADDRESS = address(this);
}
```

## 2. solidity 函数

### 2.1 函数可见性 `internal|external|public|private`

**`public`**：内部和外部均可见。

**`private`**：只能从本合约内部访问，继承的合约也不能使用。

**`external`**：只能从合约外部访问（但内部可以通过 `this.f()` 来调用，f是函数名）。

**`internal`**: 只能从合约内部访问，继承的合约可以用。

**注意 1**：合约中定义的函数需要明确指定可见性，它们没有默认值。

**注意 2**：`public|private|internal` 也可用于修饰状态变量。`public`变量会自动生成同名的`getter`函数，用于查询数值。未标明可见性类型的状态变量，默认为`internal`。

```js
// internal function
function minus() internal {
    number = number - 1;
}

// external function: 可以调用internal function
function minusCall() external {
    minus();
}
```

### 2.2 函数关键字 `pure`和`view`

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

```js
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

### 2.3 函数关键字 `payable`

`payable`用于指示函数或合约可以接受以太币（Ether）或发送以太币

`payable`含义和用法：

- 合约接收以太币：使用`payable`声明的函数可以接收以太币作为函数调用的一部分。这使得其他地址可以向合约发送以太币，从而实现资金的存储和管理
- 合约发送以太币：使用`address`类型的变量可以调用`.transfer()`或`.send()`方法向其他地址发送以太币。当向一个合约地址发送以太币时，该合约需要使用payable修饰的函数来接收和处理传入的以太币

```js
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

### 2.4 返回值

solidity中与函数输出相关的两个关键字：`return`和`returns`。区别在于：

- `return`: 用于函数主体中，返回指定的变量。
- `returns`: 跟在函数名后，用于声明返回的变量类型及变量名。

```js
function returnMultiple() public pure returns(uint256, book, uint256[3] memory) {
    return (1, true, [uint256(1),2,5]);
}
```

#### 2.4.1 命名式返回

在`returns`中标明返回变量的名称。solidity会初始化这些变量，并自动返回这些变量，无需使用`return`。

```js
function returnNamed() public returns(uint256 _number, bool _bool, uint256[3] memory _array) {
    _number = 2;
    _bool = false;
    _array = [uint256(3),2,1];
}
```

当然也可以在命名式返回中用`return`来返回变量。

```js
function returnNamed() public returns(uint256 _number, bool _bool, uint256[3] memory _array) {
    (1, true, [uint256(1),2,5]);
}
```

#### 2.4.2 解构式赋值

solidity支持使用解构式赋值规则来读取函数的全部或部分返回值。

- 读取所有返回值：声明变量，然后将要赋值的变量用`,`隔开，按顺序排列。

```js
uint256 _number;
bool _bool;
uint256[3] memory _array;

(_number, _bool, _array) = returnNamed();
```

- 读取部分返回值：声明要读取的返回值对应的变量，不读取的留空。

```js
(, _bool2,) = returnNamed();
```

### 2.5 变量数据存储和作用域

#### 2.5.1 solidity中的引用类型

**引用类型（Reference Type）**：包括数组(`array`)和结构体(`struct`)。由于这类变量比较复杂，占用存储空间大，我们在使用时必须要声明数据存储的位置

##### 数据位置

solidity数据存储位置有三类：`storage`,`memory`和`calldata`。不同存储位置的gas成本不同。`storage`类型的数据存储在链上，类似计算机的硬盘，消耗gas多；`memory`和`calldata`类型的存储位置是临时存储在内存中，消耗gas少。用法：

- `storage`：合约里的状态变量默认都是`storage`，存储在链上。
- `memory`：函数里的参数和临时变量一般用`memory`，存储在内存中，不上链。尤其是如果返回数据类型是变长的情况下，必须加`memory`修饰，例如：`string`,`bytes`和自定义结构等类型。
- `calldata`：和`memory`类似，存储在内存中，不上链。与`memory`的不同点在于`calldata`变量不能修改(`immutable`)，一般用于函数的参数。例子：

```js
function fCalldata(uint[] calldata _x) public pure returns(uint[] calldata) {
    return _x;
}
```

##### 数据位置和赋值规则

在不同存储类型相互赋值时，有时会产生独立的副本（修改新变量不会影响原变量），有时会产生引用（修改新变量会影响原变量）。规则如下：

- 赋值本质上是创建**引用**指向本体，因此修改本体或者是引用，变化可以被同步：
  - `storage`（合约的状态变量）赋值给本地`storage`（函数里的）时候，会创建引用，改变新变量会影响原变量。例子如下：
  - `memory`赋值给`memory`，会创建引用，改变新变量会影响原变量。

```js
uint[] x = [1,2,3];

function fStorage() public {
    uint[] storage xStorage = x;
    xStorage[0] = 100; // x[0] = 100;
}
```

- 其他情况下，赋值创建的是本体的副本，即对二者之一的修改并不会同步到另一方。

##### 变量的作用域

solidity中变量按作用域划分为三种，分别是状态变量（state variable），局部变量（local variable）和全局变量（global variable）

###### 1. 状态变量

状态变量是数据存储在链上的变量，所有合约内函数都可以访问，gas消耗高。状态变量在合约内、函数外声明。可以在函数里更改状态变量的值。

```js
contract Variables {
    uint public x = 1;
    uint public y;
    string pubic z;
}

function foo() external {
    x = 5;
    y = 2;
    z = "hello world";
}
```

###### 2. 局部变量

局部变量是仅在函数执行过程中有效的变量，函数退出后，变量无效。局部变量的数据存储在内存内，不上链，gas低。局部变量在函数内声明：

```js
function bar() external pure returns(uint) {
    uint xx = 1;
    uint yy = 2;
    uint zz = xx + yy;
    return zz;
}
```

###### 3. 全局变量

全局变量是全局范围工作的变量，都是solidity预留关键字。它们可以在函数内不声明直接使用：

```js
function global() external view returns(address, uint, bytes memory) {
    address sender = msg.sender;
    uint blockNum = block.number;
    bytes memory data = msg.data;
    return (sender, blockNum, data);
}
```

在上面的例子里，使用了3个常用的全局变量：`msg.sender`,`block.number`,`msg.data`，它们分别代表请求发起抵制，当前区块高度，请求数据。下面是一些常用的全局变量，更完整的列表请看[链接](https://learnblockchain.cn/docs/solidity/units-and-global-variables.html#special-variables-and-functions)

- `blockhash(uint blockNumber)`:(`bytes32`)给定区块的哈希值 - 只适用于256个最近区块，不包含当前区块
- `block.coinbase`:(`address payable`)当前区块矿工的地址
- `block.gaslimit`:(`uint`)当前区块的gas limit
- `block.number`:(`uint`)当前区块的number
- `block.timestamp`:(`uint`)当前区块的时间戳，为unix纪元以来的秒
- `gasleft()`:(`uint256`)剩余gas
- `msg.data`:(`bytes calldata`)完整call data
- `msg.sender`:(`address payable`)消息发送者(当前caller)
- `msg.sig`:(`bytes4`)calldata的前四个字节(function identifier)
- `msg.value`:(`uint`)当前交易发送的`wei`值
- `block.blobbasefee`:(`uint`)当前区块的blob基础费用。这是Cancun升级新增的全局变量
- `blobhash(uint index)`:(`bytes32`)返回跟当前交易管理的第`index`个blob的版本化哈希（第一个字节为版本号，当前应为`0x01`，后面接KZG承诺的SHA256哈希的最后31个字节）。若当前交易不包含blob，则返回空字节。这是Cancun升级新增的全局变量

###### 4. 全局变量 - 以太单位与时间单位

**以太单位**

solidity中不存在小数点，以大整型数代替，来确保交易的精确度，并且防止精度损失，利用以太单位可以避免无算的问题，方便在合约中处理货币交易。

- `wei`: 1
- `gwei`: 1e9 = 1000000000
- `ether`: 1e18 = 1000000000000000000

```js
function weiUnit() external pure returns(uint) {
    assert(1 wei == 1e0);
    assert(1 wei == 1);
    return 1 wei;
}

function gweiUnit() external pure returns(uint) {
    assert(1 gwei == 1e9);
    assert(1 gwei == 1000000000);
    return 1 gwei;
}

function etherUnit() external pure returns(uint) {
    assert(1 ether == 1e18);
    assert(1 ether == 1000000000000000000);
    return 1 ether;
}
```

**时间单位**

可以在合约中规定一个操作必须在一周内完成，或者某个时间在一个月后发送。这样就能让合约的执行更加精确，不会应为技术上的误差而影响合约的结果。因此，时间单位在solidity中是一个重要的概念，有助于提高合约的可读性和可维护性。

- `seconds`: 1
- `minutes`: 60 seconds == 60
- `hours`: 60 minutes == 3600
- `days`: 24 hours == 86400
- `weeks`: 7 days == 604800

```js
function secondsUnit() external pure returns(uint) {
    assert(1 seconds == 1);
    return 1 seconds;
}

function minutesUnit() external pure returns(uint) {
    assert(1 minutes == 60);
    assert(1 minutes == 60 seconds);
    return 1 minutes;
}

function hoursUnit() external pure returns(uint) {
    assert(1 hours == 3600);
    assert(1 hours == 60 minutes);
    return 1 hours;
}

function daysUnit() external pure returns(uint) {
    assert(1 days == 86400);
    assert(1 days == 24 hours);
    return 1 days;
}

function weeksUnit() external pure returns(uint) {
    assert(1 weeks == 604800);
    assert(1 weeks == 7 days);
    return 1 weeks;
}
```

## 3. solidity 控制流

solidity控制流与其他语言类似，主要包含一下几种：

1. `if-else`

```js
function ifElseTest(uint256 _number) public pure returns(bool){
    if(_number == 0){
        return(true);
    }else{
        return(false);
    }
}
```

2. `for`

```js
function forLoopTest() public pure returns(uint256){
    uint sum = 0;
    for(uint i = 0; i < 10; i++){
        sum += i;
    }
    return(sum);
}
```

3. `while`

```js
function whileTest() public pure returns(uint256){
    uint sum = 0;
    uint i = 0;
    while(i < 10){
        sum += i;
        i++;
    }
    return(sum);
}
```

4. `do-while`

```js
function doWhileTest() public pure returns(uint256){
    uint sum = 0;
    uint i = 0;
    do{
        sum += i;
        i++;
    }while(i < 10);
    return(sum);
}
```

5. 三元运算符

```js
function ternaryTest(uint256 x, uint256 y) public pure returns(uint256){
    // return the max of x and y
    return x >= y ? x: y; 
}
```

## 4. solidity 构造函数和修饰器

### 4.1 构造函数

构造函数（`constructor`）是一种特殊的函数，每个合约可以定义一个，并在部署合约时自动运行一次。它可以用来初始化合约的一些参数，例如初始化合约的`owner`地址：

```js
address owner;

constructor(address initialOwner) {
    owner = initialOwner;
}
```

**注意**：构造函数在不同solidity版本中的语法并不一致，在solidity 0.4.22之前，构造函数不使用`constructor`而是使用与合约名同名的函数作为构造函数。由于这种旧写法容易使开发这在书写时发生疏漏（如合约名为`Parents`，构造函数名写成`parents`），使构造函数变长普通函数，引发漏洞，所以在0.4.22版本及以后，采用了`constructor`写法。

### 4.2 修饰器

修饰器（`modifier`）是solidity特有的语法，类似于面向对象编程中的装饰器（`decorator`），声明函数拥有的特性，并减少代码冗余。`modifier`的主要使用场景是运行函数前的检查，例如地址，变量，余额等。

```js
modifier onlyOwner {
   require(msg.sender == owner); // 检查调用者是否为owner地址
   _; // 如果是的话，继续运行函数主体；否则报错并revert交易
}

function changeOwner(address _newOwner) external onlyOwner{
   owner = _newOwner; // 只有owner地址运行这个函数，并改变owner
}
```

## 5. solidity 事件

solidity中的事件`event`是`EVM`上日志的抽象，它具有两个特点：

- 响应：应用程序（如`ether.js`）可以通过RPC接口订阅和监听这些事件，并在前端做响应。
- 经济：事件是`EVM`上比较经济的存储数据的方式，每个大概消耗2000gas；相比之下，链上存储一个新变量至少需要20000gas。

### 声明事件

事件的声明由`event`关键字开头，接着是事件名称，括号里面写好事件需要记录的变量类型和变量名。以`ERC20`代币合约的`Transfer`事件为例：

```js
event Transfer(address indexed from, address indexed to, uint256 value);
```

`Transfer`事件共记录了3个变量`from`,`to`,`value`，分别对应代币的转账地址，接收地址和转账数量，其中`from`和`to`前面带有`indexed`关键字，它们会保存在`EVM`日志的`topics`中，方便以后检索。

### 释放事件

我们可以在函数里释放事件。在下面的例子中，每次用`_transfer()`函数进行转账操作的时候，都会释放`Transfer`事件，并记录相应的变量。

```js
function _transfer(
    address from,
    address to,
    uint256 amount
) external {

    _balances[from] = 10000000; // 给转账地址一些初始代币

    _balances[from] -=  amount; // from地址减去转账数量
    _balances[to] += amount; // to地址加上转账数量

    // 释放事件
    emit Transfer(from, to, amount);
}
```

### EVM日志

以太坊虚拟机（EVM）用日志`Log`来存储solidity事件，每条日志记录都包含主题`topics`和数据`data`两部分。

#### 主题`topics`

日志的第一部分是主题数组，用于描述事件，长度不能超过4.它的第一个元素是事件的签名（哈希）。对于上面的`Transfer`事件，它的事件哈希就是：

```js
keccak256("Transfer(address,address,uint256)")

//0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
```

除了事件哈希，主题还可以包含至多3个`indexed`参数，也就是`Transfer`事件中的`from`和`to`。

`indexed`标记的参数可以理解为检索事件的索引“键”，方便之后检索。每个`indexed`参数的大小为固定的256比特，如果参数太大了（比如字符串），就会自动计算哈希存储在主题中。

#### 数据`data`

事件中不带`indexed`的参数会被存储在`data`部分中，可以理解为事件的“值”。`data`部分的变量不能被直接检索，但可以存储任意大小的数据。因此一般`data`部分可以用来存储复杂的数据结构，例如数组和字符串等，因为这些数据超过了256比特，即使存储在事件的`topics`部分中，也是以哈希的方式存储。另外，`data`部分的变量在存储上消耗的gas相比于`topics`更少。

## 6. solidity 继承

### 继承规则

- `virtual`：父合约中的函数，如果希望自合约重写，需要加上`virtual`关键字。
- `override`：子合约重写了父合约中的函数，需要加上`override`关键字。

**注意**：用`override`修饰`public`变量，会重写与变量同名的`getter`函数，例如：

```js
mapping(address => uint256) public override balanceOf;
```

### 简单继承

先写一个简单的爷爷合约`Yeye`，里面包含1个`Log`事件和3个function: `hip()`, `pop()`, `yeye()`，输出都是“Yeye”。

```js
contract Yeye {
    event Log(string msg);

    function hip() public virtual {
        emit Log("Yeye");
    }

    function pop() public virtual {
        emit Log("Yeye");
    }

    function yeye() public virtual {
        emit Log("Yeye");
    }
}
```

再定义一个爸爸合约`Baba`，让他继承`Yeye`合约，语法就是`contract Baba is Yeye`，非常直观。在`Baba`合约里，我们重写一下`hip()`和`pop()`这两个函数，加上`override`关键字，并将他们的输出改为“Baba”；并且加一个新的函数`baba`，输出也是“Baba”。

```js
contract Baba is Yeye {
    function hip() public virtual override {
        emit Log("Baba");
    }

    function pop() public virtual override{
        emit Log("Baba");
    }

    function baba() public virtual{
        emit Log("Baba");
    }
}
```

部署合约，可以看到`Baba`合约里有4个函数，其中`hip()`和`pop()`的输出被成功改写成“Baba”，而继承来的`yeye()`的输出仍然是”Yeye”。

### 多重继承

solidity的合约可以继承多个合约。规则：

1. 继承时要按辈分最高到最低的顺序排序。比如我们写一个`Erzi`合约，继承`Yeye`合约和`Baba`合约，那么就要写成`contract Erzi is Yeye, Baba`，而不能写成`contract Erzi is Baba, Yeye`，不然就会报错。
2. 如果某一个函数在多个继承的合约里都存在，比如例子中的`hip()`和`pop()`，在子合约里必须重写，不然会报错。
3. 重写在多个父合约中都重名的函数时，`override`关键字后面要加上所有的父合约的名字，例如`override(Yeye, Baba)`。

```js
contract Erzi is Yeye, Baba {
    function hip() public virtual override(Yeye, Baba) {
        emit Log("Erzi");
    }

    function pop() public virtual override(Yeye, Baba) {
        emit Log("Erzi");
    }
}
```

#### 修饰器的继承

solidity中的修饰器（`Modifier`）同样可以继承，用法与函数继承类似，在相应的地方加`virtual`和`override`关键字即可。

```js
contract Base1 {
    modifier exactDividedBy2And3(uint _a) virtual {
        require(_a % 2 == 0 && _a % 3 == 0);
        _;
    }
}

contract Identifier is Base1 {

    //计算一个数分别被2除和被3除的值，但是传入的参数必须是2和3的倍数
    function getExactDividedBy2And3(uint _dividend) public exactDividedBy2And3(_dividend) pure returns(uint, uint) {
        return getExactDividedBy2And3WithoutModifier(_dividend);
    }

    //计算一个数分别被2除和被3除的值
    function getExactDividedBy2And3WithoutModifier(uint _dividend) public pure returns(uint, uint){
        uint div2 = _dividend / 2;
        uint div3 = _dividend / 3;
        return (div2, div3);
    }
}
```

`Identifier`合约可以直接在代码中使用父合约中的`exactDividedBy2And3`修饰器，也可以利用`override`关键字重写修饰器：

```js
modifier exactDividedBy2And3(uint _a) override {
    _;
    require(_a % 2 == 0 && _a % 3 == 0);
}
```

#### 构造函数的继承

子合约有两种方法继承父合约的构造函数。举个简单的例子，父合约`A`里面有一个状态变量`a`，并由构造函数的参数来确定：

```js
abstract contract A {
    uint public a;

    constructor(uint _a) {
        a = _a;
    }
}
```

1. 在继承时声明父构造函数的参数，例如：`contract B is A(1)`
2. 在子合约的构造函数中声明构造函数的参数，例如：

```js
contract C is A {
    constructor(uint _c) A(_c * _c) {}
}
```

#### 调用父合约的函数

子合约有两种方式调用父合约的函数，直接调用和利用`super`关键字。

1. 直接调用：子合约可以直接用`父合约名.函数名()`的方式来调用父合约函数，例如`Yeye.pop()`

```js
function callParent() public {
    Yeye.pop();
}
```

2. `super`关键字：子合约可以利用`super.函数名()`来调用最近的父合约函数。solidity继承关系按声明时从右到左的顺序是：`contract Erzi is Yeye, Baba`，那么`Baba`是最近的父合约，`super.pop()`将调用`Baba.pop()`而不是`Yeye.pop()`：

```js
function callParentSuper() public {
    super.pop();
}
```

#### 钻石继承

在面向对象编程中，钻石继承（菱形继承）指一个派生类同时有两个或两个以上的基类。

在多重+菱形继承链条上使用`super`关键字时，需要注意的是使用`super`会调用继承链条上的每一个合约的相关函数，而不是只调用最近的父合约。

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

/* 继承树：
  God
 /  \
Adam Eve
 \  /
people
*/

contract God {
    event Log(string message);

    function foo() public virtual {
        emit Log("God.foo called");
    }

    function bar() public virtual {
        emit Log("God.bar called");
    }
}

contract Adam is God {
    function foo() public virtual override {
        emit Log("Adam.foo called");
        super.foo();
    }

    function bar() public virtual override {
        emit Log("Adam.bar called");
        super.bar();
    }
}

contract Eve is God {
    function foo() public virtual override {
        emit Log("Eve.foo called");
        super.foo();
    }

    function bar() public virtual override {
        emit Log("Eve.bar called");
        super.bar();
    }
}

contract people is Adam, Eve {
    function foo() public override(Adam, Eve) {
        super.foo();
    }

    function bar() public override(Adam, Eve) {
        super.bar();
    }
}
```

在这个例子中，调用合约`people`中的`super.bar()`会依次调用`Eve`,`Adam`，最后是`God`合约。

虽然`Eve`,`Adam`都是`God`的子合约，但整个过程中`God`合约只会被调用一次。原因是solidity借鉴了Python的方式，强制构建一个由基类构成的DAG（有向无环图）使其保证一个特定的顺序。更多细节可查阅[solidity官方文档](https://solidity-cn.readthedocs.io/zh/develop/contracts.html?highlight=%E7%BB%A7%E6%89%BF#index-16)。

## 7. solidity 抽象合约和接口

### 抽象合约

如果一个智能合约里至少有一个未实现的函数，即某个函数缺少主题`{}`中的内容，则必须将该合约标为`abstract`，不然编译会报错；另外，未实现的函数需要加`virtual`，以便子合约重写。

### 接口

接口类似于抽象合约，但它不实现任何功能。接口的规则：

1. 不能包含状态变量
2. 不能包含构造函数
3. 不能继承除接口外的其他合约
4. 所有函数都必须是external且不能有函数体
5. 继承接口的非抽象合约必须实现接口定义的所有功能

虽然接口不是闲任何功能，但它非常重要。接口是智能合约的骨架，定义了合约的功能以及如何触发它们：如果智能合约实现了某种接口（比如`ERC20`或`ERC721`），其他DApps和智能合约就知道如何与它交互。因为接口提供了两个重要的信息：

1. 合约里每个函数的`bytes4`选择器，以及函数签名`函数名(每个参数类型)`。
2. 接口id（更多信息见[EIP165](https://eips.ethereum.org/EIPS/eip-165)）

另外，接口与合约`ABI`（Application Binary Interface）等价，可以相互转换：编译接口可以得到合约的`ABI`，利用[abi-to-sol工具](https://gnidan.github.io/abi-to-sol/)，也可以将`ABI json`文件转换为`接口sol`文件。

以`ERC721`接口合约`IERC721`为例，它定义了3个`event`和9个`function`，所有`ERC721`标准的NFT都实现了这些函数。可以看到，接口和常规合约的区别在于每个函数都以`;`代替函数体`{}`结尾。

```js
interface IERC721 is IERC165 {
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);
    
    function balanceOf(address owner) external view returns (uint256 balance);

    function ownerOf(uint256 tokenId) external view returns (address owner);

    function safeTransferFrom(address from, address to, uint256 tokenId) external;

    function transferFrom(address from, address to, uint256 tokenId) external;

    function approve(address to, uint256 tokenId) external;

    function getApproved(uint256 tokenId) external view returns (address operator);

    function setApprovalForAll(address operator, bool _approved) external;

    function isApprovedForAll(address owner, address operator) external view returns (bool);

    function safeTransferFrom( address from, address to, uint256 tokenId, bytes calldata data) external;
}
```

#### IERC721事件

`IERC721`包含3个事件，其中`Transfer`和`Approval`事件在`Erc20`中也有。

- `Transfer`事件：在转账时被释放，记录代币的发出地址`from`，接收地址`to`和`tokenId`。
- `Approval`事件：在授权时被释放，记录授权地址`owner`，被授权地址`approved`和`tokenId`。
- `ApprovalForAll`事件：在批量授权时被释放，记录批量授权的发出地址`owner`，被授权地址`operator`和授权与否的`approved`。

#### IERC721函数

- `balanceOf`：返回某地址的NFT持有量`balance`。
- `ownerOf`：返回某`tokenId`的主人`owner`。
- `transferFrom`：普通转账，参数为转出地址`from`，接收地址`to`和`tokenId`.
- `safeTransferFrom`：安全转账（如果接收方是合约地址，会要求实现`ERC721Receiver`接口）。参数为转出地址`from`，接收地址`to`和`tokenId`。
- `approve`：授权另一个地址使用你的NFT。参数为被授权地址`approve`和`tokenId`。
- `getApproved`：查询`tokenId`被批准给了哪个地址。
- `setApprovalForAll`：将自己持有的该系列NFT批量授权给某个地址`operator`。
- `isApprovedForAll`：查询某地址的NFT是否批量授权给了另一个`operator`地址。
- `safeTransferFrom`：安全转账的重载函数，参数里面包含了`data`。

#### 什么时候使用接口？

如果我们知道一个合约实现了`IERC721`接口，我们不需要知道它具体代码实现，就可以与它交互。

无聊猿`BAYC`属于`ERC721`代币，实现了`IERC721`接口的功能。我们不需要知道它的源代码，只需知道它的合约地址，用`IERC721`的接口就可以与它交互，比如用`balanceOf()`来查询某个地址的`BAYC`余额，用`safeTransferFrom()`来转账`BAYC`。

```js
contract interactBAYC {
    // 利用BAYC地址创建接口合约变量（ETH主网）
    IERC721 BAYC = IERC721(0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D);

    // 通过接口调用BAYC的balanceOf()查询持仓量
    function balanceOfBAYC(address owner) external view returns (uint256 balance){
        return BAYC.balanceOf(owner);
    }

    // 通过接口调用BAYC的safeTransferFrom()安全转账
    function safeTransferFromBAYC(address from, address to, uint256 tokenId) external{
        BAYC.safeTransferFrom(from, to, tokenId);
    }
}
```

## 8. solidity 异常

合约编写中难以避免会出现bug，异常能够帮助我们进行debug。solidity中三种抛出异常的方法：`error`,`require`,`assert`。

### Error

`error`是solidity 0.8.4版本新加的内容，方便且高效（省gas）地向用户解释操作失败的原因，同时还可以在抛出异常的同时携带参数，帮助开发者更好地调试。人们可以在`contract`之外定义异常。下面我们定义一个`TransferNotOwner`的异常，当用户不是代币`owner`但却尝试转账时，抛出错误：

```js
error TransferNotOwner(); // 自定义error

error TransferNotOwner(address sender); // 自定义的带参数的error
```

在执行当中，`error`必须搭配`revert`（回退）命令使用。

```js
function transferOwner1(uint256 tokenId, address newOwner) public {
    if(_owners[tokenId] != msg.sender){
        revert TransferNotOwner();
        // revert TransferNotOwner(msg.sender);
    }
    _owners[tokenId] = newOwner;
}
```

我们定义了一个`transferOwner1()`函数，它会检查代币的`owner`是不是发起人，如果不是，就会抛出`TransferNotOwner`异常；如果是的话，就会转账。

### Require

`require`命令是solidity 0.8版本之前的抛出异常的常用方法，目前很多主流合约仍然还在使用它。它很好用，唯一的缺点就是gas随着描述异常的字符串长度增加，比`error`命令要高。使用方法：`require(检查条件,"异常的描述")`，当检查条件不成立的时候，就会抛出异常。

```js
function transferOwner2(uint256 tokenId, address newOwner) public {
    require(_owners[tokenId] == msg.sender, "Transfer Not Owner");
    _owners[tokenId] = newOwner;
}
```

### Assert

`assert`命令一般用于程序员写程序时debug，因为它不能解释抛出异常的原因（比`require`少个字符串）。它的用法很简单，`assert(检查条件)`，当检查条件不成立的时候，就会抛出异常。

```js
function transferOwner3(uint256 tokenId, address newOwner) public {
    assert(_owners[tokenId] == msg.sender);
    _owners[tokenId] = newOwner;
}
```

### 三种方法的gas比较

通过remix控制台，能够查到每次函数调用的gas消耗（使用solidity 0.8.17版本编译）：

1. `error`方法gas消耗：24457（加入参数后gas消耗：24660）
2. `require`方法gas消耗：24755
3. `assert`方法gas消耗：24473

可以看到，`error`方法gas消耗最少，其次是`assert`，`require`消耗gas最多。因此，应尽量使用`error`。（注意，由于部署测试时间的不同，每个函数的gas消耗会有所不同，但是比较结果会是一致的。）

**备注**: solidity 0.8.0之前的版本，`assert`抛出的是一个`panic exception`，会把剩余的 gas 全部消耗，不会返还。更多细节见[官方文档](https://docs.soliditylang.org/en/v0.8.17/control-structures.html)。
