---
layout: post
title: "STM32库函数版学习"
date: 2023-05-11
tags: ["混饭吃"]
toc: true
comments: true
author: vdadh
excerpt: "基于正点原子F103战舰V3开发板+F103C8T6最小系统板+江科大的STM32库函数学习"
---

> 学习基础的外设驱动代码编写，从库函数版学习，了解底层寄存器配置；
>
> 学习重点：RCC、定时器、外部中断、ADC、基于定时器的PWM

##### GPIO八种工作模式

![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306121954157.png)

![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306101703259.png)

##### 其他：

* LED灯珠：长正短负