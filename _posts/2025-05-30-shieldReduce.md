---
layout: post
title: shieldReduce
description: 阅读shieldReduce的笔记
tag: 记录
---

## 关键词
去重，增量压缩，本地压缩，SGX

## 背景
### plain data（没有加密的数据），有三种数据压缩办法
- 去重:通过**分块**和**生成哈希值**来实现。fingerprint index
- 增量压缩：通过生成特征来寻找相似块，如果两个块至少有一个特征相同则它们是相似块。feature index
- 本地压缩：无损方式。不对delta chunk进行laocal compresssion,因为收效甚微。

通过file recipe来重建文件。

### 加密去重的局限
通过一种加密方式保证来自不同用户的相同块能被加密成一样，服务器端无法获得源数据也能够实现去重。
- 可能通过频率分析的方式泄露数据
- 熵值高，难以进一步压缩

在客户端压缩再加密的方式也会影响多用户之间的数据压缩。

### SGX
细粒度体现在只在一部分环境enclave可信而不是整台机器。
加密是在数据压缩之后。
在enclave中执行去重，本地压缩，增量压缩，将加密结果存放在持久存储中。

客户端分块，并通过安全会话传输chunks。

### 威胁模型
攻击者试图获得源数据。

#### 对于不可信内存和云存储的监听
1. 攻击者可以获得不可信内存中的index，但是无法获得chunk

### 挑战
- SGX的资源限制
- base chunk多，不好管理

## 要解决的问题
- 存储节约和数据安全的矛盾
- 兼容去重和细粒度数据压缩

## 所使用的办法
DEBE通过基于频率的去重减少资源开销

过去的研究通常把第一个chunk作为base chunk

使用Finesse作为相似度匹配的技术，将其他相似度匹配技术作为未来的工作。

所有几KB的chunk都被组合成几MB的container,所有I/O操作都以container为单位进行。
为了解决增量压缩时访问base chunk的I/O开销，通过双向增量压缩保持物理局部性,进而减少开销。
由于是从持久存储上加载一批base chunk，所以保持物理局部性可以减少I/O次数。
根据参数在两种增量压缩的方式之间交换，以平衡I/O开销和存储节省。

- 在线压缩办法：去重后，如果局部性存在，执行在线的前向增量压缩和本地压缩。
    - 对于有base chunk的chunk，从持久存储中取出相应base chunk到enclave，在enclave中对base chunk解密解压缩，对chunk进行增量压缩，加密后存储到持久存储。
    - 对于没有相应base chunk的chunk，本地压缩后存到持久存储，做为新的base chunk。
- 离线压缩办法：去重后，如果局部性不存在，通过离线的反向增量压缩重建局部性。

数据结构：
- delta index：存储在不可信内存，用于追踪增量关系。key是base chunk的指纹,value是对应delta chunk的指纹。
- backward index:存储在不可信内存，将基础快的加密指纹映射到一组对应的数据块的加密指纹，在反向压缩结束后清除。
- deletion map:临时性，记录那些在持久存储中有需要删除的副本的chunk
- file recipe:记录备份文件的所有chunk,如果是delta chunk，还要记录相应的base chunk

### 在线
默认一批128个chunk，从每个chunk提取三个特征。进行局部性检测，用***所用container数量/一批chunk的总数***表示局部性，商越小局部性越明显。如果局部性存在，进行前向增量压缩和本地压缩；如果局部性不存在，准备离线压缩。

准备离线压缩：本地压缩每个chunk，存到持久存储。

### 离线
从持久存储中检索出：
1. 旧的base chunk
2. 延迟进行增量压缩的chunk
3. 相对于旧的base chunk已进行增量压缩的chunk

选择backward index中最新的chunk作为base chunk,对backward index中其他chunk和旧的base chunk进行增量压缩（1和2）。
某些旧的base chunk的增量压缩会被跳过，通过offline reduction target α来调节，小的α意味着更多旧的base chunk会被跳过。
***注意：新的base chunk和其他chunk可能没有相同特征，但他们都和1有相同特征。***
用相对于新的base chunk的delta chunk替换原来持久存储中的base chunk，更新：
1. feature index
2. delta index
3. fingerprint index
4. file recipe

### 存储组织
container被分成base chunk container和delta chunk container

## 实现
基于Intel SGX SDK Linux 2.15，index用std::unordered_map来实现，加密指纹用SHA-256来实现
每个用户使用FastCDC来分块
使用Finesse提取chunk的特征，使用Edelta进行增量压缩，使用LZ4进行本地压缩。

## 测试
数据集：
- 源代码版本
- 二进制快照
- 网站
- 操作系统镜像

1. baseline1:DEBE,没有增量压缩
2. baseline2:ForwardDelta,没有反向增量压缩
3. baseline3:SecureMeGA

### Exp#1(Analysis of ***data reduction***)
> ShieldReduce, ForwardDelta,and SecureMeGA achieve higher storage savings than DEBE due to delta compression.ShieldReduce has a similar data reduction ratio as ForwardDelta.

ShieldReduce ≈ ForwardDelta > SecureMeGA > DEBE

>ShieldReduce keeps almost the same data reduction ratio(63.6×) in SimOS for 0 ≤ α ≤ 0.7

### Exp#2 (Microbenchmarks)

### Exp#3 (Inline performance)

### Exp#4 (Multi-client performance)