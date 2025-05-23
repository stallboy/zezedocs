---
layout: page
title: 序列化
parent: intro
nav_order: 13
---

* TOC
{:toc}

# Serialize

Zeze有一个很小的自己的序列化实现。麻雀虽小五脏俱全。
Zeze.Transaction.Bean是在ByteBuffer基础上构建的一套对象序列化实现。
主要涉及以下三个类:

1. Zeze.Serialize.Serializable 接口
2. Zeze.Serialize.IByteBuffer 接口(目前只用于提供ByteBuffer的只读方法)
3. Zeze.Serialize.ByteBuffer 实现

以下是序列化标准的定义:

## 类型枚举(4-bit)

- 0: 有符号整数(byte,short,int,long,bool)
- 1: 单精度浮点数(float)
- 2: 双精度浮点数(double)
- 3: 二进制数据/字符串(binary,string)
- 4: 序列容器(list,set)
- 5: 关联容器(map)
- 6: bean
- 7: 动态bean(dynamic)
- 8: vector2(float*2)
- 9: vector2int(int*2)
- 10: vector3(float*3)
- 11: vector3int(int*3)
- 12: vector4(float*4)
- 13~15: 可自行扩展定义非标准类型

## 整数压缩

比如：用户定义了long类型，当值小于64时，仅用一个字节保存。

```
* 有符号整数(支持64位补码有符号整数的所有值)
  正整数:
    1字节(<  0x               40): 00xx xxxx  (取低有效位,按大端排列)
    2字节(<  0x             2000): 010x xxxx  +1B
    3字节(<  0x          10 0000): 0110 xxxx  +2B
    4字节(<  0x         800 0000): 0111 0xxx  +3B
    5字节(<  0x      4 0000 0000): 0111 10xx  +4B
    6字节(<  0x    200 0000 0000): 0111 110x  +5B
    7字节(<  0x 1 0000 0000 0000): 0111 1110  +6B
    8字节(<  0x80 0000 0000 0000): 0111 1111  0xxx xxxx  +6B
    9字节(             unlimited): 0111 1111  1xxx xxxx  +7B
  负整数:
    1字节(>=-0x               40): 11xx xxxx
    2字节(>=-0x             2000): 101x xxxx  +1B
    3字节(>=-0x          10 0000): 1001 xxxx  +2B
    4字节(>=-0x         800 0000): 1000 1xxx  +3B
    5字节(>=-0x      4 0000 0000): 1000 01xx  +4B
    6字节(>=-0x    200 0000 0000): 1000 001x  +5B
    7字节(>=-0x 1 0000 0000 0000): 1000 0001  +6B
    8字节(>=-0x80 0000 0000 0000): 1000 0000  1xxx xxxx  +6B
    9字节(             unlimited): 1000 0000  0xxx xxxx  +7B

* 无符号整数(支持32位无符号整数的所有值)
  仅用于序列化长度,数量,增量等特定情况
    1字节(<0x       80): 0xxx xxxx  (取低有效位,按大端排列)
    2字节(<0x     4000): 10xx xxxx  +1B
    3字节(<0x  20 0000): 110x xxxx  +2B
    4字节(<0x1000 0000): 1110 xxxx  +3B
    5字节(   unlimited): 1111 0000  +4B
```

## 布尔(bool)

- 与有符号整数的序列化兼容
- 序列化时false当作0,true当作1; 反序列化时0当作false,其余当作true

## 单精度浮点数(float)

- 按IEEE754标准序列化成小端排列的固定4个字节

## 双精度浮点数(double)

- 按IEEE754标准序列化成小端排列的固定8个字节

## 二进制数据(binary)

- 无符号整数: 数据的字节长度
- 数据的原始内容

## 字符串(string)

- 按二进制数据序列化字符串的UTF-8编码数据

## 序列容器(list,set)

- 单字节: (高位)nnnn tttt(低位)
    - t: 容器元素的类型枚举
    - n=0~14: 容器元素的数量
    - n=15: 附加一个无符号整数(x),用15+x表示容器元素的数量
- 按以上指定类型及枚举顺序连续序列化容器的所有元素

## 关联容器(map)

- 单字节: (高位)kkkk vvvv(低位)
    - k: 容器键(key)的类型枚举
    - v: 容器值(value)的类型枚举
- 无符号整数: 容器键值对的数量
- 按以上指定类型及枚举顺序连续序列化容器的所有键值, 即"键值键值..."的顺序

## 二维浮点向量类型(vector2)

- 按float类型序列化x字段
- 按float类型序列化y字段

## 二维整数向量类型(vector2int)

- 按有符号整数类型序列化x字段
- 按有符号整数类型序列化y字段

## 三维浮点向量类型(vector3)

- 按float类型序列化x字段
- 按float类型序列化y字段
- 按float类型序列化z字段

## 三维整数向量类型(vector3int)

- 按有符号整数类型序列化x字段
- 按有符号整数类型序列化y字段
- 按有符号整数类型序列化z字段

## 四维浮点向量类型(vector4)

- 按float类型序列化x字段
- 按float类型序列化y字段
- 按float类型序列化z字段
- 按float类型序列化w字段

## 字段标签(tag)

- 单字节: (高位)iiii tttt(低位)
    - i=0: 特殊含义
        - t=0: 结束标签
        - t=1: 结束当前层的标签,后续是上一层(父类)的序列化
        - t=2~15: 未定义,保留扩展
    - i=1~14: 距上个字段ID的增量(初始为首个字段ID), t表示字段的类型枚举
    - i=15: 附加一个无符号整数(x),用15+x表示ID增量, t表示字段的类型枚举

## bean

- 按"字段标签,字段值,字段标签,字段值,...,结束标签(单字节0)"的顺序序列化
- 序列化字段的顺序要求按字段ID从小到大排列, 字段ID的支持范围: [1,0x7fffffff]
- 字段值如果等于默认值,可省略该字段标签及其值的序列化
- 反序列化时要求先重置bean的所有字段为默认值再反序列化
- 有继承关系的bean要求先序列化子类字段,然后插入结束当前层的标签,再序列化上一级父类
- 默认值的定义:
    - 数值=0
    - 二进制数据长度=0
    - 字符串长度=0
    - 容器元素数量=0
    - bean的所有字段均为默认值
    - 动态bean为未定义值(空值)

## 动态bean(dynamic)

- 有符号整数: 动态bean的ID
- bean的序列化

## 协议帧(protocol)

- 小端排列的固定4字节无符号整数: 模块ID(moduleId)
- 小端排列的固定4字节无符号整数: 协议ID(protocolId)
- 小端排列的固定4字节无符号整数: 下面bean的序列化字节长度
- bean的序列化

## 关于序列化的兼容性

- 有符号整数,单精度浮点数,双精度浮点数之间可以自动转换,但要注意高位截断和降低精度问题
- binary和string之间可以互相转换,但要注意binary转到string时有无法正确解码UTF-8而抛出异常的可能
- list和set之间可以互相转换,但要注意list转成set后再序列化可能会改变顺序
- bean可以转成动态bean,默认typeId=0; 动态bean可以转换成bean,但要注意不同bean类型之间的字段兼容性
- 以上没提到的类型转换说明不兼容,反序列化会自动忽略不兼容的字段
