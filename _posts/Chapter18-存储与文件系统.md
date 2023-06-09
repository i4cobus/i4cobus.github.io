# 存储与文件系统

## 磁盘（机械硬盘）

### 存储结构

一个机械硬盘主体结构包括一个磁柱、一组磁碟、一组机械臂和磁头。

- 在磁碟上通过磁效应记录信息
- 每个磁碟的两个表面都可以存储数据
- 数据保存在以磁柱为中心的同心圆环（磁道）上
- 常见转速为 5400RPM 或 7200RPM

一些术语

- 盘片 Platter
  - 两面都可以用
  - 每个磁头对应一面
- 磁道 Track
- 柱面 Cylinder
  - 不同盘面在相同半径上的磁道
- 扇区 Sector
  - 最小访问单元

### Cylinder-Head-Sector CHS编址方法

- 使用柱面、磁头、扇区的三元组来唯一确定一个扇区的位置
- 现代机械硬盘提供内置控制器，用来在机械硬盘内部计算地址
- 操作系统只需要使用逻辑块地址 $A$ 来对机械硬盘的第 $A$ 个扇区进行访问

$$ A = (C \times N_{heads} + H) \times N_{sectors} + S - 1$$

### 机械硬盘访问特点

- 磁盘高速旋转
- 机械臂带动磁头
- 随机访问性能远差于顺序访问性能
  - 差距可能达到100倍甚至更高

## 日志文件系统 Log-structured File System LFS

### 概述

- 假设文件被缓存在内存中，文件读请求可以被很好地处理
  - 瓶颈在于写文件
- 块存储设备的顺序写入比随机写入速度快很多
- 将文件系统的修改以日志的方式顺序写入存储设备

### 结构概述

- 有一些固定的位置
  - 超级块
  - 检查点区域
- 其余部分以Log结构顺序保存
  - inode块、间接块、数据块
  - inode map：记录每个inode当前的位置
  - 段概要：记录段中的有效块
  - 段使用表：记录段中有效字节数、最后修改时间等
  - 目录修改日志

#### 一个例子

假设文件系统本来长这样

```text
    file1       file2         /
   ┌─────┐     ┌─────┐     ┌─────┐
   │     │     │     │     │     │
┌──▼──┬──┴──┬──▼──┬──┴──┬──▼──┬──┴──┬─────┐
│     │     │     │     │     │     │     │
│block│inode│block│inode│block│inode│ map │
│     │     │     │     │     │     │     │
└─────┴──▲──┴─────┴──▲──┴─────┴──▲──┴──┬──┘
         │           │           │     │
         └───────────┴───────────┴─────┘
```

接下来使用 `echo hello >/file3` 创建一个新的文件

```text
    file1       file2         /               file3         /
   ┌─────┐     ┌─────┐     ┌─────┐           ┌─────┐     ┌─────┐
   │     │     │     │     │     │           │     │     │     │
┌──▼──┬──┴──┬──▼──┬──┴──┬──▼──┬──┴──┬─────┬──▼──┬──┴──┬──▼──┬──┴──┬─────┐
│     │     │     │     │ xxx │ xxx │ xxx │     │     │     │     │     │
│block│inode│block│inode│bxxxk│ixxxe│ xxx │block│inode│block│inode│ map │
│     │     │     │     │ xxx │ xxx │ xxx │     │     │     │     │     │
└─────┴──▲──┴─────┴──▲──┴─────┴─────┴─────┴─────┴──▲──┴─────┴──▲──┴──┬──┘
         │           │                             │           │     │
         └───────────┴─────────────────────────────┴───────────┴─────┘
```

### 空间回收利用

- 随文件系统使用，日志写入位置会接近存储设备的末端
- 同时之前会有一些不再使用的空间
  - 文件系统数据块被修改后，之前的块就被无效化
- 需要重新利用之前的空间

#### 串联

- 将所有空闲块用空闲链表串联
  - 弊端：磁盘空间会逐渐碎片化，影响到LFS的大块顺序写的性能

#### 拷贝

- 将所有有效空间整理拷贝到新的存储设备
  - 弊端：需要拷贝数据到**新设备**

#### 基于段的回收管理

> 串联和拷贝结合

- 一个存储设备被拆分为定长的**段**
  - 大小可能是512KB、1MB等
- 每段内只能顺序写入
  - 只有段内全部是无效数据时才能重新使用
- 干净段使用空闲链表串联

##### 段使用表

- 记录每个段中的有效字节数
  - 有效字节归零时，该段变为干净段
- 记录每个段的最近写入时间
  - 将干净段按时间顺序串联在一起，形成逻辑上的连续空间

##### 段清理

1. 将一些段读入内存准备清理
2. 识别有效数据
3. 将有效数据整理后写入干净段
4. 将被清理的段标记为干净段

> 那么怎么识别有效数据？

##### 识别有效数据

- 每个段中保存**段概要**
  - 记录每个块被那个文件的哪个位置使用
    - 例如，段中每个数据块可以使用inode号和数据块序号来表示位置
  - 记录块有效性
    - 可以根据对应inode的数据块指针是否指向自己来判断
    - 如果不指向自己当前的位置，则自己是无效块

#### 挂载和恢复

> 挂载和从崩溃中恢复时，需要扫描所有日志，重建整个文件系统的内存结构
> 此时大量的无效数据也被扫描，浪费很多时间

##### 检查点

- 检查点位于磁盘固定位置
  - 保存以下数据
    - inode map的位置（用于找到所有文件的内容）
    - 段使用表
    - 当前时间
    - 最后写入的段的指针
- 通常有两个
  - 防止写一个检查点的时候崩了，这个时候至少另一个还是可用的
- 定期将有效数据写入检查点
  - 只需要扫描检查点之后写入的日志，减少挂载和恢复时间

##### 前滚恢复

- 尽可能恢复检查点后写入的数据
- 通过段概要记录的新inode恢复新的inode
  - inode中的数据块也会被一并自动恢复
- 没有inode的数据块会被删除
- 但是前滚恢复不能保证inode链接数的一致性
  - 例如inode本身被持久化，但是指向这个inode的目录没有被持久化，则inode的引用数可能会和实际指向它的数量不一致

##### 目录修改日志

- 解决inode链接数一致性
- 记录了每个目录操作的信息
  - 例如 `create` `link` `rename` `unlink`
- 记录了每个操作的具体信息
  - 例如目录项位置、内容、inode链接数
- 先持久化目录修改日志，再对目录本身进行修改
  - 恢复时根据日志保证inode链接数一致性

## 存储设备与文件系统的演进

### 瓦式磁盘

- 机械硬盘通常比较容易读，但是不容易写：读时的磁头可以比写的磁头更窄。
- 瓦式磁盘允许磁道重叠，进而提高存储容量
  - 写的时候宽一些，但是读的时候不需要这么宽
- 问题在于瓦式磁盘的实现决定了它不能支持随机写入
  - 否则会覆盖旁边磁道的数据
  - 因此通常把瓦式磁盘划分成多个band
    - band内只能顺序写入
    - band间可以随机写入

#### band内的随机写入

##### 多次拷贝

- 假设要修改 band `x` 中的 4KB 数据
- 首先找到空闲 band `y`
- 将 `x` 拷贝到 `y`，拷贝的同时写入需要写入的 4KB 数据
- 将 `y` 拷贝回 `y`

代价是写入时巨大的访存开销。为了写入 4KB，需要2次完整的band读和2次完整的band写

##### 缓存+动态映射

- 硬件增加大容量持久缓存
  - 该部分磁道不会重叠，可以随机写入
  - 对外部不可见
  - 通常容量在几十GB
- 动态映射 Shingle Translation Layer STL
  - 负责从外部地址到内部物理地址的映射

此时，要修改 band `x` 中的 4KB 数据

- 将 `x` 标记为dirty
- 修改 STL 映射，将 `x` 的逻辑地址指向持久化缓存
- 空闲时，根据缓存内容清理
- 此时 4KB 的随机写只需要修改 4KB 缓存

###### 问题

- 如果持久化缓存满了，则瓦式磁盘的吞吐量会暴毙

##### Ext4对瓦式磁盘的支持

- 如果持久化缓存的写入比较分散，则清理时需要清理大量band
  - 造成随机写性能拉胯
- 如果多个随机写能写在同一个band，则只需要清理一个band
- Ext4的日志系统包含了大量的、分散的随机写，导致瓦式磁盘吞吐量低
  - Ext4将磁盘划分为2GB的**块组**，每个块组有自己的元数据（包含了块组内的inode table）
    - 保证了一些局部性，不用突然回到头部读inode table
    - 但是对瓦式磁盘就是灾难
  - 将元数据从日志区写会时，会非常分散

###### 解决方法

- 以LFS形式增加元数据的缓存
  - 最新的元数据将放置在该缓存区，而不是每个块组的开头
- 通过 Journal Map 实现从元数据本来的地址到缓存区地址的映射
  - 上层仍然只需要读原本的元数据的地址，Journal Map会负责映射到缓存区的最新元数据
- 如果日志区满了，则执行清理
  - 无效的元数据可以直接回收
  - 对于不太访问的元数据，可以择机写会原区域
  - 经常被修改的热元数据仍然保留在缓存区

### 闪存盘、固态硬盘与F2FS

#### 结构

`Die -> Plane -> Block -> Page -> Cell -> Level`

#### 特性

- 多通道
  - 可以批量读取数据
- 不能覆盖写
  - 需要先擦除旧数据，才能重新写入
  - 读写粒度为 `page` (4-16KB)
  - 擦除粒度为 `block` (4-8MB)
    - 反复擦除可能消耗这个 `block` 的寿命
- 磨损
  - 同一个位置擦写过多，将会无法写入
  - 需要磨损均衡
  - 文件系统的bitmap和inode table经常会有写入，如果不做任何操作，则这些位置非常容易写穿

#### 动态映射 Flash Translation Layer

- 由SSD内置的控制芯片实现
  - 用于垃圾回收、磨损均衡等
- 逻辑地址到物理地址的转换
- 对外使用逻辑地址
- 对内使用物理地址

## LFS的递归更新

- 考虑一个多级索引的文件数据
  - 修改数据后，索引块要改
  - 索引快修改后，二级索引块要改
  - 二级索引修改后，inode要修改
  - inode修改后，inode map要修改

### Node Address Translation NAT

- 维护一个固定位置的 NAT
  - 负责从逻辑地址到物理地址的翻译
  - 所有索引通过NAT寻址
  - 修改时只需要修改NAT

> 硬件在变，但是OS增加抽象和映射的方法论一直不变
