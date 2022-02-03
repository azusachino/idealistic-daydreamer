---
title: 计算机是怎样跑起来的 - 笔记
description: The basic of computer science
date: 2022-02-03
slug: understand-how-computer-run
image: img/2022/02/WinterHalo.jpg
categories:
  - learning
tags:
  - cs
---

## 计算机的三大原则

1. 计算机是执行输入、运算和输出的机器
2. 程序是指令和数据的集合
3. 计算机的处理方式与人们的思维习惯不同

## 冯诺伊曼模型

![ ](img/2022/02/Von_Neumann_Architecture.png)

## 微型计算机模型

![ ](img/2022/02/cs-basic-computer.png)

> 注：带有黑点的电路交叉处才表示通路

### 组成元件

![ ](img/2022/02/cs-basic-computer-component.jfif)

#### Z80 CPU

- Z80 CPU 的地址总线引脚有 16 个，能够指定 65536 个地址
- Z80 CPU 的数据总线引脚有 8 个，能够一次性输入输出 8 比特数据；如果数据大于 8 比特，就要以 8 比特为单位进行拆分

Z80 CPU 通过 MREQ(Memory Request)引脚和 IORQ(I/O Request)引脚区分当前与 CPU 做交互的是主存还是 IO。

#### Z80 PIO

Z80 PIO 上共有 4 个寄存器。2 个用于设定 PIO 本身的功能，2 个用于存储与外部设备进行输入输出的数据。

![ ](img/2022/02/cs-basic-component-pio.jfif)

若表示 IC 引脚作用的代号上化有横线，则表示通过赋予该引脚 0(0V)使之生效，反之通过赋予该引脚 1(5V)使之生效。C 表示控制模式，D 表示数据模式。

### 主要元件的引脚描述

![ ](img/2022/02/cs-basic-component-leg.jfif)

## 其他

文章后部分内容就没有太多新奇之处了，涉猎较多但不深入，这里暂不做探讨了。

## Reference

- [微信读书 - 计算机是怎样跑起来的](https://weread.qq.com/web/reader/b9b324005dd9f0b9b9e6f17kc81322c012c81e728d9d180)
