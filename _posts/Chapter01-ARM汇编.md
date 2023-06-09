# ARM汇编

> “你们的程序都有漏洞，只是不值得被攻击而已。”

## 汇编语言基础

### 编译过程

- 预处理 `*.c -> *.i（文本）`
  - 生成经预处理的源程序
- 编译 `*.i -> *.s（文本）`
  - 生成汇编程序
- 汇编 `*.s -> *.o（二进制）`
  - 生成可重定向目标程序
- 链接 `*.o -> exe（二进制）`
  - 生成可执行目标程序
  - “能看这个的……都是特殊的人。”

### 指令

> “很多时候看汇编需要一些想象力。”

- 汇编语言每条指令只完成非常基本的操作

### 操作数

#### 寄存器

- `<char><number>: X1`
- ARM64的逻辑上有31个64位通用寄存器。
- `X0`指向完整的64位
- `W0`指向末32位

#### 中间结果

#### 内存引用（虚拟内存）

`M[addr]`

##### 虚拟内存抽象概述

- 虚拟内存 `M` 可以看作一个巨大数组
- 通过 `addr` 访问内存的特定单元
- `addr` 的格式由寻址模式决定
- 虚拟内存中的数据不区分类型
  - 反汇编得到的高级语言和原本的源代码功能一致，但是文本上可能不同，反汇编推断的数据类型也仅供参考

##### 寻址模式

###### 偏移量模式 `M[base + offset]`

- `base` 可以使任何64位通用寄存器
- `offset` 可以是
  - 省略
  - 立即数
  - 64位通用寄存器 `<reg>`
  - 修改过的寄存器 `<reg>, <op>`
    - `<op>` 可以是
      - 算术运算 `add #imm`
      - 移位运算 `lsl #imm`
      - 位扩展 `sxtb`

###### 索引模式

- 索引模式会**更新** `base` 的值
- 前索引`[base, offset]!`
  - 先计算新的 `base`，然后根据新 `base` 寻址
- 后索引`[base], offset`
  - 根据旧的 `base` 寻址，然后更新 `base`

#### 立即数

`#<number>: #4`

## 程序执行

### 虚拟内存空间

```text
┌──────────┐
│          │
│  Kernel  │
│          │
├──────────┤
│   Stack  │ <- 用户栈
├──────────┤
│          │
├──────────┤
│    Lib   │ <- 共享库的内存映射
├──────────┤
│          │
├──────────┤
│   Heap   │ <- 运行时堆
├──────────┤
│    R/W   │ <- 读写段 (.data, .bss)
├──────────┤
│    RO    │ <- 只读段 (.init, .text, .rodata)
└──────────┘
```

### 程序员可见的程序状态

- 程序计数器 Program Counter PC
  - 指向下一条要执行的指令
- 栈指针寄存器 Stack Pointer SP
  - 指向运行时栈的栈顶
- 程序状态寄存器 PSTATE
  - 存放程序执行状态的信息
  - 包含多个标志位
    - 例如用于实现条件跳转的条件码
- 寄存器文件
  - 包含若干个整数和浮点数寄存器

### 汇编指令及其执行模式

#### 汇编指令

- 每条指令只完成一个基本操作
- 大部分指令操作存储在寄存器中的数据
- 访存指令能够在内存和寄存器之间传递数据

#### 汇编程序

- 汇编指令的序列

#### 汇编程序的执行模式

- 默认情况下顺序执行
- 跳转指令可以改变指令执行流

#### 指令序列的执行

- 指令会被存放在虚拟内存的代码段

#### ARM架构下程序的执行

- 每条指令是一个定长的二进制序列，共64bit
- PC存储下一条指令在内存中的地址
- CPU从该地址读取一条指令
- CPU执行指令，并更新PC

#### 对齐

- 对齐是对某些对象的存储地址的限制
  - 要求起始地址必须是某个值 k 的倍数
  - k一般为2，4或8
- 可以简化处理器与内存系统之间硬件接口的设计

##### ARM架构的对齐

- ARM的硬件在数据没有对齐的情况下仍然能正确工作
- 但是数据对齐能够提高内存系统的性能
- 可以在汇编中对对齐限制进行说明
  - `.align 8`

##### Linux的对齐

- 1字节数据结构地址没有限制
- 2字节数据结构地址必须是2的倍数
- 4字节数据结构地址必须是4的倍数
- 更大的数据结构地址必须是8的倍数
- `malloc()` 返回的泛型指针对齐要求为8字节

##### 结构体的对齐

- 成员各自满足对齐限制
- 结构体的对齐限制等于最大的成员对齐限制
- 结构体有时需要在成员间插入空隙
- 结构体有时需要在末尾处增加填充

## ARM汇编指令简介

### 操作类指令

#### 数据搬移 `mov`

`mov dest, src`

- 源操作数 `src` 可以是
  - 立即数
  - 寄存器
  - 修改过的寄存器
- 目的操作数 `dest` 只能是寄存器
  - 被搬移数据的大小可以由目的操作数长度推测

#### 访存指令

##### `ldr` `str`

`ldr reg, offset` `str reg, offset`

- `reg` 是读内存的数据目标，或写内存的数据来源
  - 内存单元大小可以由寄存器长度推测
- `offset` 是[偏移量模式](#偏移量模式-mbase--offset)或[索引模式](#索引模式)表示的内存地址

##### `ldp` `stp`

`ldp reg1, reg2, offset` `stp reg1, reg2, offset`

- 同时将两个寄存器值写入内存，或将内存的值写入两个寄存器
- 先读写 `reg1` 再读写 `reg2`
- 常见于栈帧构建与释放时批量存储 `LR` 和 `FP` 寄存器
  - “他们发现寄存器经常是两个两个一起存的，所以就有了这个指令”

#### 符号扩展

##### 无符号扩展 `uxtb`, `uxth`

- `uxtb` Unsigned Extend Byte
  - 将 `1Byte` 无符号扩展至目标寄存器大小
- `uxth` Unsigned Extend Half-word (2 Byte)
  - 将 `2Byte` 无符号扩展至目标寄存器大小

##### 有符号扩展 `sxtb`, `sxth`, `sxtw`

- `sxtb` Signed Extend Byte
- `sxth` Signed Extend Half-word
- `sxtw` Signed Extend Word
  - 将 `4Byte` 有符号扩展至目标寄存器大小

#### 算术与逻辑

- `add Rd, Rn, Op2` 加
- `sub Rd, Rn, Op2` 减
- `mul Rd, Rn, Op2` 乘
- `sdiv Rd, Rn, Op2` 有符号除法
- `udiv Rd, Rn, Op2` 无符号除法
- `neg Rd, Rn` 取反
- `eor Rd, Rn, Op2` 按位异或
- `orr Rd, Rn, Op2` 按位或
- `and Rd, Rn, Op2` 按位与
- `lsl Rd, Rn Op2` 左移
- `asr Rd, Rn, Op2` 算术右移
- `lsr Rd, Rn, Op2` 逻辑右移
- `ror Rd, Rn, Op2` 循环右移

#### 例子：访问数组

假设一个 `int32` 类型的数组 `E` 的基地址 `Xe` 存放在 `x19`，下标 `i` 存放在 `x20`

| Expression | Dtype  |       Value       |                             ASM                              |
| :--------: | :----: | :---------------: | :----------------------------------------------------------: |
|    `E`     | `int*` |       `Xe`        |                        `mov x0, x19`                         |
|   `E[0]`   | `int`  |      `M[Xe]`      |                       `ldr x0, [x19]`                        |
|   `E[i]`   | `int`  |    `M[Xe+4i]`     |                 `ldr x0, [x19, x20, lsl #2]`                 |
|  `&E[2]`   | `int*` |     `Xe + 8`      |                      `add x0, x19, #8`                       |
|  `E+i-1`   | `int*` |   `Xe + 4i - 4`   |    `lsl x21, x20, #2` `sub x0, x21, #4` `add x0, x19, x0`    |
| `*(E+i-3)` | `int`  | `M[Xe + 4i - 12]` | `lsl x21, x20, #2` `add x19, x19, x21` `ldr x0, [x19, #-12]` |

#### 例子：结构体与联合

- 结构体 struct
  - 结构体的所有成员被存储在一个连续的内存区域
  - 结构体的指针是该区域第一个字节的地址
- 联合 union
  - 一个联合对象可以以不同的数据类型访问
  - 联合中的所有成员引用都指向同一块内存区域

### 条件码

- 由 `PSTATE` 寄存器维护
- 描述上一条执行的特殊指令的属性
  - 并非所有ARM64指令都会改变条件码
  - 带有 `s` 后缀的算术或逻辑指令 (如 `adds`)
  - 比较指令与测试指令 (`cmp`, `cmn`, `tst`)
- 包括四个标志位
  - `C`：进位/借位标志
  - `V`：溢出标志
  - `N`：上一指令结果为0
  - `Z`：上一指令结果为负

#### 条件码的设置

##### 通过 `s` 后缀的指令隐式设置

`s` 表示 *set conditional flags*

##### 通过比较指令 `cmp` 显式设置

`cmp src1, src2`

- 计算 `src1 - src2` 但是不保存结果
  - `C` 当减法产生借位时设置
  - `V` 当运算产生有符号溢出时设置
  - `Z` 当两个操作数相等时被设置
  - `N` 当 `src1` 小于 `src2` 时被设置

##### 通过测试指令 `tst` 显式设置

`tst src1, src2`

- `C`, `V` 在此时不会被设置
- `Z` 当 `src1 & src2 == 0` 时
- `N` 当 `src1 & src2 < 0` 时

#### 条件码的访问

- 条件码不能被直接读取
- 访问方式通常是根据某个预设的条件是否满足来设置寄存器的值
  - 使用 `cset Rd, condition`
    - `condition` 为预设的一些测试条件
    - 若测试条件为真，则将 `Rd` 设为 1，否则为 0

##### 条件一览表

| Condition | ConditionCode |   Semantics    |
| :-------: | :-----------: | :------------: |
|    EQ     |       Z       |   相等或为0    |
|    NE     |      ~Z       |   不等或非0    |
|    MI     |       N       |      负数      |
|    PL     |      ~N       |      非负      |
|    LT     |      N^V      |   有符号小于   |
|    LE     |  (N^V) \| Z   | 有符号小于等于 |
|    GT     |   ~(N^V)&~Z   |   有符号大于   |
|    GE     |    ~(N^V)     | 有符号大于等于 |
|    HI     |     C&~Z      |   无符号大于   |
|    LS     |     ~C\|Z     | 无符号小于等于 |
|    LO     |      ~C       |   无符号小于   |

### 控制类指令

#### 跳转指令

- `b <LABEL>`：无条件直接跳转
  - `000101 <IMM26>`
  - 相对PC地址偏移
  - 一次跳跃最远范围为 $2^{26}$，跳转距离为 $2^{28}$
- `br <Reg>`：无条件间接跳转
- `b<condition> <LABEL>`：有条件直接跳转
  - 支持19位相对PC地址偏移

相对位置跳转在可重定向目标文件中并不会计算出绝对的跳转地址，而是只记录跳转偏移量。在链接后才会计算出完整的跳转地址。

### 过程调用

- 过程/函数调用时另一种形式的无条件跳转：控制流在两段代码之间转移
  - 过程调用后会返回
  - 涉及参数和返回值传递
  - 存在局部变量
  - 涉及寄存器和程序状态保存

#### 调用被调者 `bl`

##### 指令

- 直接调用 `bl <LABEL>`
- 间接调用 `blr <Reg>`

##### 行为描述

- 将返回值存储在**链接寄存器 `LR`**，这里的链接寄存器通常是寄存器 **`x30`**
- 跳转到被调者的**入口地址**

#### 返回调用者 `ret`

- 根据 `LR` 设置返回地址
- 跳转到调用者内的返回地址

#### 内存栈帧结构

##### 程序栈

$$\begin{vmatrix}
  栈顶\\
  运行时栈\\
  \vdots\\
  共享库的内存映射\\
  \vdots\\
  运行时堆\\
  读写段\\
  只读段
\end{vmatrix}$$

- 从高地址向低地址扩张
- 堆叠正在执行的函数的**帧**
  - 每个函数对应一个帧
  - 执行函数时会为被调用者创建一个新的帧
  - 从被调者返回时会销毁被调用者的帧
  - 推出一帧只需要调整栈指针 `SP`
  - 不会“清空”原本分配的帧

##### 帧

- 栈帧是在运行时栈上为某个函数分配的内存空间
- 栈指针 `SP` 指示运行时栈的栈顶
- 帧指针 `FP` (寄存器 `x29`) 指示当前帧的基地址

##### 栈帧操作

- 创建新栈帧
  - 移动栈指针（分配新的帧）
  - 保存返回地址 `x30`
  - 保存旧帧地址 `x29`

#### 数据传递

```text
┌────────┐
│  LR''  │
├────────┤
│  FP''  │
├────────┤ <- FP'
│  argN  │
├────────┤
│  ...   │
├────────┤
│  arg9  │
├────────┤ <- SP'
│ callee │
└────────┘ <- FP/SP
```

- 使用寄存器 `x0` 至 `x7` 传递**前8个参数**
  - 放不下的参数由调用者压到栈上
  - 按声明顺序**从右到左**
    - 第9个参数在栈顶
  - 所有数据对齐到8字节
  - 会导致 `FP` 和 `SP` 指向不一致
    - `SP` 永远指向栈顶
- 被调用者通过 `SP` + 偏移值访问参数
- 使用寄存器 `x0` 传递**返回值**

#### 寄存器管理

将寄存器划分为调用者保存和被调者保存

- 调用者保存：调用者调用函数前保存，被调者可以随意使用
  - 寄存器 `x9` 至 `x15`
  - 调用后的值可能发生改变，需要调用者负责保存和恢复
  - 使用较少，一般只存放函数调用前的临时值
- 被调者保存：被调函数在使用前保存，调用者可以随意使用
  - 寄存器 `LR` 和 `FP`、旧 `LR` 和 `FP`、寄存器 `x19` 至 `x28`
  - 被调者内部可能调用其他函数，此时 `LR` `FP` 将被覆盖，因此被调者需要负责保存旧的 `LR` 和 `FP`
  - 保存在栈帧的底部，参数构造区之前

```text
  ──┌────────┐
    │  LR''  │
 c  ├────────┤
 a  │  FP''  │
 l  ├────────┤ <- FP'
 l  │  argN  │
 e  ├────────┤
 r  │  ...   │
    ├────────┤
    │  arg9  │
  ──├────────┤ <- SP'
    │  x20   │
 c  ├────────┤
 a  │  x19   │
 l  ├────────┤
 l  │ callee │
 e  │ frame  │
 e  ├────────┤
    │  LR'   │
    ├────────┤
    │  FP'   │
  ──└────────┘ <- FP/SP
```

#### 运行时栈

- 寄存器放不下或者不能放（内存地址）的局部变量将存放于栈上

##### 局部变量

- 局部变量区位于被保存的寄存器下方
- 在分配栈帧时被一起分配

#### 栈帧架构

```text
   ┌─────────────────────┐
   │                     │
   │   Previous frames   │
   │                     │
───┼─────────────────────┤
   │         LR''        │
 c ├─────────────────────┤
 a │         FP''        │
 l ├─────────────────────┼───
 l │         argN        │
 e ├─────────────────────┤
 r │         ...         │ Func Arguments
   ├─────────────────────┤
   │         arg9        │
───┼─────────────────────┼───
   │         x20         │
 c ├─────────────────────┤ Reg Saves
 a │         x19         │
 l ├─────────────────────┼───
 l │      local var      │
 e ├─────────────────────┤
 e │         LR'         │
   ├─────────────────────┤
   │         FP'         │
───┴─────────────────────┘
```
