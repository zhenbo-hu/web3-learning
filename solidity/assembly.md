# Solidity Yul Assembly

## 1.1 Types

Yul 中只有一种类型——u256，也可以理解为 bytes32

使用 solidity 实现一个简单的读取函数

```solidity
function getNumber() external pure returns (uint256) {
    return 42;
}
```

利用 Yul Assembly 实现

```solidity
function getNumber() external pure returns (uint256) {
    uint256 x;

    assembly {
        x := 42
    }

    return x;
}
```

字符串赋值

```solidity
function demoString() external pure returns (bytes32) {
    bytes32 myString = "";

    assembly {
        myString := "hello world"
    }

    return myString;
}
```

将 bytes32 转成 string

```solidity
function demoString() external pure returns (string memory) {
    bytes32 myString = "";

    assembly {
        myString := "hello world"
    }

    return string(abi.encode(myString));
}
```

## 1.2 Basic Operations

Yul 中的 for 循环和 if 条件语句

### for 循环

```solidity
function isPrime(uint256 x) public pure returns (bool p) {
    p = true;

    assembly {
        let halfX := add(div(x, 2), 1) // x / 2 + 1
        for {let i := 2} lt(i, halfX) {i := add(i, 1)} {
            if isZero(mod(x, i)) {
                p := 0
                break
            }
        }
    }
}
```

### if 条件语句

```solidity
function isTruthy() external pure returns (uint256 result) {
    result = 2;

    assembly {
        if 2 {
            result := 1
        }
    }

    return result; // return 1
}

function isFalsy() external pure returns (uint256 result) {
    result = 1;
    assembly {
        if 0 {
            result := 2
        }
    }

    return result; // return 1
}
```

通过 `iszero` 来判断条件是否为 `false`. `iszero(0)` 返回 32 字节的 `0x000...01`

```solidity
function negation() external pure returns (uint256 result) {
    result := 1

    assembly{
        if iszero(0) {
            result := 2
        }
    }

    return result; // return 2
}
```

`not(0)` 对 0 的每一位进行取反运算，返回 `0xfff...ff`

```solidity
function unsafeNegationPart1() external pure returns (uint256 result) {
    result = 1;

    assembly {
        if not(0) {
            result := 2
        }
    }

    return result; // return 2
}
```

在内联汇编中，if 条件语句没有 else 分支

```solidity
function max(uint256 x, uint256 y) external pure returns (uint256 maximum) {
    assembly {
        if lt(x, y) {
            maximum := y
        }

        if iszero(lt(x, y)) {
            maximum := x
        }
    }
}
```

`lt(x, y)` 用于判断 x 是否小于 y，如果 x < y，则返回 1；否则返回 0

在 solidity 内联汇编中，for 循环和 if 条件语句的用法和 solidity 编程很像，但是条件判断依赖于数值是否为 0，而不是 true 或 false

[完整的 yul 指令列表](https://docs.soliditylang.org/en/develop/yul.html#evm-dialect)

## 1.3 Storage Slots

在 solidity 中，我们可以通过一下代码来存储变量

```solidity
contract StorageBasics {
    uint256 private x;

    function setX(uint256 newVal) external {
        x = newVal;
    }

    function getX() external view returns (uint256) {
        return x;
    }
}
```

### 使用 .slot 后缀获取变量存储槽

```solidity
function getXYulSlot() external pure returns (uint256 ret) {
    assembly {
        ret := x.slot // 0
    }
}
```

### 使用 `sload(slot)` 读取指定槽中存储的数据

```solidity
function getXYul() external view returns (uint256 ret) {
    assembly {
        ret := sload(x.slot)
    }
}
```

### 使用 `sstore(slot, value)` 修改存储数据

```solidity
contract StorageBasics {
    uint256 x = 11;
    uint256 y = 22;
    uint256 z = 33;

    function getVarYul(uint256 slot) external view returns (uint256 ret) {
        assembly {
            ret := sload(slot)
        }
    }

    function setVarYul(uint256 slot, uint256 value) external {
        assembly {
            sstore(slot, value)
        }
    }
}
```

### 变量打包和存储槽共享

```solidity
contract StorageBasics {
    uint256 x = 11;
    uint256 y = 22;
    uint256 z = 33;
    uint128 a = 1;
    uint128 b = 2;

    function getSlot() external pure returns (uint256 slot) {
        assembly {
            slot := b.slot // return 3
        }
    }

    function getVarYul(uint256 slot) external view returns (bytes32 ret) {
        assembly {
            ret := sload(slot) // return 0x0000000000000000000000000000000100000000000000000000000000000002
        }
    }
}
```

在这个例子中，变量 a 和 b 共享第 3 个存储槽。

## 1.4 Storage Offsets and Bitshifting

### 读取同一个槽中指定数据

```solidity
contract StorageBasics {
    uint128 public C = 4;
    uint96 public D = 6;
    uint16 public E = 8;
    uint8 public F = 1;

    function readBySlot(uint256 slot) external view returns (bytes32 value) {
        assembly {
            value := sload(slot)
        }
    }
}
```

在这个合约中，C、D、E 和 F 共享第 0 个存储槽。我们可以通过 readBySlot 函数读取第 0 个槽的内容，其结果如下：`0x0001000800000000000000000000000600000000000000000000000000000004`

接下来，我们可以通过`.offset`读取 E 的偏移量：

```solidity
function getOffsetE() external pure returns (uint256 slot, uint256 offset) {
    assembly {
        slot := E.slot
        offset := E.offset
    }
}
```

获取 E 的偏移量为 28，意味着我们需要将槽右移 28 个字节（224 位，正好为 128+96 ）能读取到 E 的值。以下是具体的实现：

```solidity
function readE() external view returns (bytes32 e) {
    assembly {
        let value := sload(E.slot)
        let shifted := shr(mul(E.offset, 8), value)

        e := and(0xffff, shifted) // equivalent to 0x000000000000000000000000000000000000000000000000000000000000ffff
    }
}
```

读取 E 的做法是，将槽 0 中的内容读出，然后对该值右移 28 \* 8 位。右移后，由于 E 前面还有值 F, 故需要用掩码把 E 前面的值全成为 0.

读出 E 的值为`0x0000000000000000000000000000000000000000000000000000000000000008`

我们也可以使用除法来代替右移，但需要注意的是，除法的 gas 费消耗比右移大：

```solidity
function readE() external view returns (bytes32 e) {
    assembly {
        let value := sload(E.slot)
        let shifted := div(value, 0x100000000000000000000000000000000000000000000000000000000) // 1后面56个0

        e := and(0xffff, shifted)
    }
}
```

### 修改同一个槽中的指定数据

接下来，我们来看如何修改槽中的指定数据。首先回顾下 and/or 的逻辑：

- V and 00 = 00
- V and FF = V
- V or 00 = V

在修改槽中数据时，我们通常遵循以下步骤：

1. 使用 0xfff...000...fff 掩码和槽中数据进行“与”运算，将指定位置的数据清零。
2. 将修改后的数据位移到刚清零的位置，然后与第一步处理后的数据进行“或”运算，将新数据写入槽中。

具体实现：

```solidity
function writeToE(uint16 newE) external {
    assembly {
        // newE = 0x000000000000000000000000000000000000000000000000000000000000000a
        let c := sload(E.slot)
        // c = 0x0001000800000000000000000000000600000000000000000000000000000004
        let clearedE := and(c, 0xffff0000ffffffffffffffffffffffffffffffff)
        // mask     = 0xffff0000ffffffffffffffffffffffffffffffffffffffffffffffffffffffff
        // c        = 0x0001000800000000000000000000000600000000000000000000000000000004
        // clearedE = 0x0001000000000000000000000000000600000000000000000000000000000004
        let shiftedNewE := shl(mul(E.offset, 8), newE)
        // shiftedNewE = 0x0000000a00000000000000000000000000000000000000000000000000000000
        let newVal := or(shiftedNewE, clearedE)
        // shiftedNewE = 0x0000000a00000000000000000000000000000000000000000000000000000000
        // clearedE    = 0x0001000000000000000000000000000600000000000000000000000000000004
        // newVal      = 0x0001000a00000000000000000000000600000000000000000000000000000004
        sstore(C.slot, newVal)
    }
}
```

我们首先读取槽中的值 c，然后使用掩码将 E 的位置清零。接着，将新值 newE 进行左移操作，将其对齐到 E 的位置。最后，通过“或”运算将新值写入槽中。

## 1.5 Storage of Arrays and Mappings

### 数组

#### 固定长度数组

```solidity
contract StorageComplex {
    uint256[3] fixedArray;

    constructor() {
        fixedArray = [99, 999, 9999];
    }

    function fixedArrayView(uint256 index) external view returns (uint256 ret) {
        assembly {
            ret := sload(add(fixedArray.slot, index))
        }
    }
}
```

#### 动态数组

```solidity
contract StorageBasics {
    uint256[] bigArray;
    uint8[] smallArray;

    constructor() {
        bigArray = [10, 20, 30];
        smallArray = [1, 2, 3];
    }

    function bigArrayLength() external view returns (uint256 ret) {
        assembly {
            ret := sload(bigArray.slot) // return 3
        }
    }

    function readBigArrayLocation(uint256 index) external view returns (uint256 ret) {
        uint256 slot;
        assembly {
            slot := bigArray.slot
        }
        // 计算方法为：keccak256(slot) + index, 即“存放长度的槽的哈希”就是开始存放元素的存储槽的位置，然后从这个位置开始依次存放元素。
        bytes32 location = keccak256(abi.encode(slot))
        assembly {
            ret := sload(add(location, index))
        }
    }

    function smallArrayLength() external view returns (uint256 ret) {
        assembly {
            ret := sload(smallArray.slot) // return 3
        }
    }

    // input 0 returns 0x0000000000000000000000000000000000000000000000000000000000030201
    function readSmallArrayLocation(uint256 index) external view returns (bytes32 ret) {
        uint256 slot;
        assembly {
            slot := smallArray.slot
        }
        // 对于非 uint256 类型的动态数组，几个元素会共用一个槽。
        bytes32 location = keccak256(abi.encode(slot))
        assembly {
            ret := sload(add(location, index))
        }
    }
}
```

### 映射 Mapping

#### 单层映射

映射的存储方式与动态数组相似。不同之处在于，动态数组存储槽的位置是 keccak256(slot)，然后加上索引；而映射则是将 slot 和 key 连接后再做哈希处理，即 keccak256(key, slot)。

```solidity
contract StorageComplex {
    mapping(uint256 => uint256) public myMapping;

    constructor() {
        myMapping[10] = 5;
        myMapping[11] = 6;
    }

    function getMapping(uint256 key) external view returns (uint256 ret) {
        uint256 slot;
        assembly {
            slot := myMapping.slot
        }

        bytes32 location = keccak256(abi.encode(key, uint256(slot)));

        assembly {
            ret := sload(location) // input 10 return 5
        }
    }
}
```

#### 嵌套映射

对于嵌套映射，需要进行多次哈希计算，比如对于两次的嵌套计算方式为：`keccak256(key2, keccak256(key1, slot))`.

```solidity
contract StorageComplex {
    mapping(uint256 => mapping(uint256 => uint256)) public nestedMapping;

    constructor() {
        nestedMapping[2][4] = 7;
    }

    function getNestedMapping() external view returns (uint256 ret) {
        uint256 slot;
        assembly {
            slot := nestedMapping.slot
        }

        bytes32 location = keccak256(abi.encode(uint256(4), keccak(abi.encode(uint256(2), uint256(slot)))));

        assembly {
            ret := sload(location) // return 7
        }
    }
}
```

#### 映射与数组的组合

```solidity
contract StorageComplex {
    mapping(address => uint256[]) public addressToList;

    constructor() {
        addressToList[0x5B38Da6a701c568545dCfcB03FcB875f56beddC4] = [42, 1337, 777];
    }

    // lengthOfNestedList 方法用于获取映射对应数组的长度，与单层映射类似，计算方式为 keccak256(key, slot)。不同之处在于，这个槽存储的是数组的长度，而单层映射中，这个槽存储的是具体的元素。
    function lengthOfNestedList() external view returns (uint256 ret) {
        uint256 slot;

        assembly {
            slot := addressToList.slot
        }

        bytes32 location = keccak256(abi.encode(address(0x5B38Da6a701c568545dCfcB03FcB875f56beddC4), uint256(slot)));

        assembly {
            ret := sload(location) // 3
        }
    }

    // 回顾获取数组元素的方法：keccak256(slot) + index。这意味着需要先对存放数组长度的槽进行哈希计算，然后加上索引。也就是说把 keccak256(key, mapping_slot) 代入到 slot 中，那么，计算公式为：keccak256(keccak256(key, mapping_slot)) + index，这样我们就可以获取数组中的元素。
    function getAddressToList(uint256 index) external view returns (uint256 ret) {
        uint256 slot;
        assembly {
            slot := addressToList.slot
        }

        bytes32 location = keccak256(abi.encode(keccak256(address(0x5B38Da6a701c568545dCfcB03FcB875f56beddC4), uint256(slot))));

        assembly {
            ret := sload(add(location, index))
        }
    }
}
```
