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

## 2.1 Memory Operations

在以下操作时将使用到内存：

- 外部调用合约的时候，返回值将放在内存中。
- 设置外部调用函数的参数。
- revert 时返回错误信息。
- 日志信息。
- 创建其他智能合约。
- 使用 keccak256 函数。

### 概述

- 内存相当于其他语言的“堆”
  - 但是没有垃圾回收机制与 `free` 命令。
  - 内存以 32 字节序列的形式布局。以 32 字节的增量来寻址。
  - [0x00 - 0x20) [0x20 - 0x40) [0x40 - 0x60) [0x60 - 0x80) [0x80 - 0x100)...
- 只有 4 条指令与内存有关
  - `mload`, `mstore`, `mstore8`, `msize`
- 用纯 yul 写的代码，内存比较容易使用。但在 solidity/yul 的混合代码中， solidity 使用了特殊的方式来使用内存。
- **重要的是**，每次内存访问都会消耗 gas，且访问距离越远，消耗的 gas 越多。
  - `mload(0xffffffffffffffff)` 指令将会超出 3000 万 gas 使用的限制。
  - 因此，像在存储中通过哈希值定位位置的方法在内存中不可行。
- `mstore(p, v)` 将值 `v` 存储到内存地址 `p` 开始的“内存槽”中（以后文章中“内存槽”指的是从该地址开始的 32 字节区域）。
- `mload(p)` 是从内存槽 `p[p..0x20]` 中获取 32 字节的数据。
- `mstore8(p, v)` 仅将 1 字节的数据存储到地址 `p` 中。
- `msize()` 返回当前事务中最大的可访问内存索引。

### 图解内存

比如，我想将 32 字节的 0xff...fff 存储到内存槽 0x00 中，将得到以下的结果，可以看到地址 0x00 - 0x19 都填了 ff。

![solidity memory 1](../pic/solidity%20memory%201.webp)

需要注意的是，内存的最小单位是 1 字节，这与存储槽不同。因此，如果将数据存储在内存槽 0x01 中，可以看到地址 0x01 - 0x20 都填了 ff。

![solidity memory 2](../pic/solidity%20memory%202.webp)

如果存储的数据较小，例如 mstore(0x00, 7)，首先会将 7 扩展为 32 字节，即 mstore(0x00, 0x000...0007)，结果如下：

![solidity memory 3](../pic/solidity%20memory%203.webp)

而使用 mstore8(0x00, 0x7) 时，则只会在地址 0x00 处存入 1 字节的值：

![solidity memory 4](../pic/solidity%20memory%204.webp)

## 2.2 How Solidity Uses Memory

### Solidity 是如何使用内存的

- Solidity 将地址 [0x00 - 0x20) 和 [0x20 - 0x40) 分配为哈希操作的“临时空间”。
- Solidity 预留地址 [0x40 - 0x60) 作为“空闲内存指针”，通常指向当前未被使用的内存区域。
- 地址 [0x60 - 0x80) 保持为空，作为 32 字节的零值插槽，用于需要零值的场合。
- 内存使用从 [0x80 - ...) 开始。
- Solidity 使用内存的场景：
  - `abi.encode` 和 `abi.encodePacked`.
  - 结构体和数组（但是你需要明确使用 `memory` 关键字）。
  - 在函数参数中，使用了 `memory` 关键字的结构体或数组。
  - 内存中的对象是按顺序排列的，因此数组不能像存储（storage）那样进行动态添加元素（push）。
- 在 Yul 中，
  - 变量存储在内存中的位置即为数组的起始位置。
  - 对于动态数组，需额外添加 32 字节（即 0x20）以跳过数组长度信息，从而访问数组的实际元素。这是因为在 Solidity 中，动态数组会存储数组长度（32 字节），随后才是实际的数组元素。

### 结构体

```solidity
contract Memory {
    struct Point {
        uint256 x;
        uint256 y;
    }

    event MemoryPointer(bytes32);
    event MemoryPointerMsize(bytes32, bytes32);

    // “空闲内存指针”本来指向 0x80，分配了结构体后，指向了 0xc0。这个差值为 64 字节，即 2 个 32 字节，正好是结构体中的 x 和 y。
    function memPointerV1() external {
        bytes32 x40;
        assembly {
            x40 := mload(0x40)
        }
        emit MemoryPointer(x40); // 0x0000000000000000000000000000000000000000000000000000000000000080

        Point memory p = Point({x: 1, y: 2});
        assembly {
            x40 := mload(0x40)
        }
        emit MemoryPointer(x40); // 0x00000000000000000000000000000000000000000000000000000000000000c0
    }

    // msize() 返回最大可访问的内存地址。当内存中未存放数据时，该值为 0x60；而当存放了一个结构体后，该值与“空闲内存指针”的值一致，为 0xc0。
    // 在最后一部分代码中使用 pop(mload(0xff)) 的主要目的是为了读取内存槽 0xff 的数据，即[0xff - 0x120)，让可访问空间变大，因此此时 msize() 变为 0x120。
    function memPointerV2() external {
        bytes32 x40;
        bytes32 _msize;
        assembly {
            x40 := mload(0x40)
            _msize := msize()
        }
        // 0x0000000000000000000000000000000000000000000000000000000000000080
        // 0x0000000000000000000000000000000000000000000000000000000000000060
        emit MemoryPointerMsize(x40, _msize);

        Point memory p = Point({x: 1, y: 2});
        assembly {
            x40 := mload(0x40)
            _msize := msize()
        }
        // 0x00000000000000000000000000000000000000000000000000000000000000c0
        // 0x00000000000000000000000000000000000000000000000000000000000000c0
        emit MemoryPointMsize(x40, _msize);

        assembly {
            pop(mload(0xff))
            x40 := mload(0x40)
            _msize := msize()
        }
        // 0x00000000000000000000000000000000000000000000000000000000000000c0
        // 0x0000000000000000000000000000000000000000000000000000000000000120
        emit MemoryPointMsize(x40, _msize);
    }
}
```

### 定长数组

```solidity
function fixedArray() external {
    bytes32 x40;
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40); // 0x0000000000000000000000000000000000000000000000000000000000000080

    uint256[2] memory arr = [uint256(5), uint256(6)];
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40); // 0x00000000000000000000000000000000000000000000000000000000000000c0
}
```

和结构体类似，存放数组后，“空闲内存指针”从 0x80 指向 0xc0，差值为 64 字节，说明定长数组不会保存数组长度信息。

### ABI Encode

```solidity
function abiEncode() external {
    bytes32 x40;
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40); // 0x0000000000000000000000000000000000000000000000000000000000000080

    abi.encode(uint256(5), uint256(19));
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40); // 0x00000000000000000000000000000000000000000000000000000000000000e0
}
```

同样是两个 uint256, 但是空闲内存指针指向了 0xe0, 这是为什么呢？

![how solidity uses memory 1](../pic/how%20solidity%20uses%20memory%201.webp)

因为在 bytes 类型的 abi 编码规则中，会先记录字节长度，可以看到此处在内存槽 0x80 处存储的是 0x40 （即 64）。

```solidity
function abiEncode2() external {
    bytes32 x40;
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40); // 0x0000000000000000000000000000000000000000000000000000000000000080

    abi.encode(uint256(5), uint128(19));
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40); // 0x00000000000000000000000000000000000000000000000000000000000000e0
}
```

即使在 abi.encode(uint256(5), uint128(19)) 中使用了 uint128(19)，结果依然与之前相同。这是由于 abi.encode 的填充规则所致。

### ABI Encode Packed

```solidity
function abiEncodePacked() external {
    bytes32 x40;
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40); // 0x0000000000000000000000000000000000000000000000000000000000000080

    abi.encodePacked(uint256(5), uint128(19));
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40); // 0x00000000000000000000000000000000000000000000000000000000000000d0
}
```

空闲内存指针指向了 0xd0, 比之前的 0xe0 要小，这和 `abi.encodePacked` 有关系，我们来看看内存布局情况。

![how solidity uses memory 2](../pic/how%20solidity%20uses%20memory%202.webp)

可以看到使用 `abi.encodePacked` 编码后的数据如下：

```shell
0x80:
0x0000000000000000000000000000000000000000000000000000000000000030
0xa0:
0x0000000000000000000000000000000000000000000000000000000000000005
0xc0:
0x0000000000000000000000000000001300000000000000000000000000000000
```

其中，0x30 是字节长度（即 48）。

### 函数参数中的数组

```solidity
event Debug(bytes32, bytes32, bytes32, bytes32);

function args(uint256[] memory arr) external {
    bytes32 location;
    bytes32 len;
    bytes32 valueAtIndex0;
    bytes32 valueAtIndex1;

    assembly {
        location := arr // 获取 arr 数组在内存中的起始位置。
        len := mload(arr) // 内存槽 arr 存放的是数组的长度。
        valueAtIndex0 := mload(add(arr, 0x20)) // 跳过存长度的内存槽，依次读取数据。
        valueAtIndex1 := mload(add(arr, 0x40))
    }

    // input: [1 ,2]
    //  "0": "0x0000000000000000000000000000000000000000000000000000000000000080",
    //  "1": "0x0000000000000000000000000000000000000000000000000000000000000002",
    //  "2": "0x0000000000000000000000000000000000000000000000000000000000000001",
    //  "3": "0x0000000000000000000000000000000000000000000000000000000000000002"
    emit Debug(location, len, valueAtIndex0, valueAtIndex1);
}
```

## 2.3 Dangers of Memory Misuse

### 注意事项

- **内存指针管理**：如果你不遵循 Solidity 的内存布局和空闲内存指针规则，你可能会遇到一些严重的 bug。
- **内存解包行为**：内存不会尝试将小于 32 字节的数据类型打包在一起。当数据从存储加载到内存时，即使这些数据在存储中是打包的（如 `uint8` 或 `bool`），在加载到内存中后，每个数据项都会占用完整的 32 字节空间。

#### 示例一：内存指针的误用

```solidity
function breakFreeMemoryPointer(uint256[1] memory foo) external view returns (uint256) {
    assembly {
        mstore(0x40, 0x80) // 人为修改空闲内存指针
    }

    uint256[1] memory bar = [uint256(6)];

    // input [1000], return 6
    return foo[0]; // 返回 foo[0] 的值
}
```

假如我们传入参数 `[1000]`，正常情况下，这个参数会被写入内存槽 `0x80`，空闲内存指针此时指向 `0xa0`。然而，我们将空闲内存指针强制设为 `0x80`，此时内存中再装入定长数组 `bar`，其值 `6` 会被写入到 `0x80` 处，覆盖原有的值。因此，`return foo[0]` 将会返回 `0x80` 地址处的内容，即 `6`。

#### 示例二：内存中的数据解包

```solidity
uint8[] foo = [1, 2, 3, 4, 5, 6];

function unpacked() external {
    uint8[] memory bar = foo;
}
```

![dangers of memory misuse](../pic/dangers%20of%20memory%20misuse.webp)

在这个示例中，虽然 `foo` 在存储槽中是紧密打包在一个槽中，但当它被读取到内存中时，数据会被解包。每个 `uint8` 数据类型的元素在内存中都会单独占用 32 字节空间。

## 2.4 Return, Require, and Keccak256

### return

```solidity
function return2and4() external pure returns (uint256, uint256) {
    assembly {
        mstore(0x00, 2)
        mstore(0x20, 4)
        return (0x00, 0x40)
    }

    // return 2 4
}
```

在 `assembly` 里，`return(p, s)` 是一条指令，表示结束执行并返回结果。这里的 `p` 是内存的起始位置，而 `s` 则表示数据的长度。上述代码会将内存地址 `0x00` 到 `0x40` 的数据返回，其中包括两个 32 字节的整数：2 和 4。通过这种方式，可以直接从内存中读取和返回所需的数据。

### revert

```solidity
function requireV1() external view {
    require(msg.sender == 0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2);
}

function requireV2() external view {
    assembly {
        if iszero(eq(caller(), 0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2)) {
            revert(0, 0)
        }
    }
}
```

`revert(p, s)` 指令用于中止执行并恢复所有状态变化。它与 `return` 类似，也可以返回数据。

### keccak256

```solidity
function hashV1() external pure returns (bytes32) {
    bytes memory toBeHashed = abi.encode(1, 2, 3);
    return keccak256(toBeHashed);
    // return 0x6e0c627900b24bd432fe7b1f713f1b0744091a646a9fe4a65a18dfed21f2949c
}

function hashV2() external pure returns (bytes32) {
    assembly {
        let freeMemoryPointer := mload(0x40)

        // store 1, 2, 3 in memory
        mstore(freeMemoryPointer, 1)
        mstore(add(freeMemoryPointer, 0x20), 2)
        mstore(add(freeMemoryPointer, 0x40), 3)

        // update memory pointer
        mstore(0x40, add(freeMemoryPointer, 0x60))

        mstore(0x00, keccak256(freeMemoryPointer, 0x60))
        return(0x00, 0x20)
    }
    // return 0x6e0c627900b24bd432fe7b1f713f1b0744091a646a9fe4a65a18dfed21f2949c
}
```

`keccak256(p, n)` 用于对内存地址 `p` 到 `p + n` 之间的数据进行哈希处理。在 `hashV2` 函数中，数据 `1, 2, 3` 被存储在内存中，然后使用 `keccak256` 进行哈希计算。结果被存储在内存地址 `0x00`，并返回给调用者。
