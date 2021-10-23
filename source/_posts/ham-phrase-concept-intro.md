---
title: Ham（业余无线电）简语/概念介绍
date: 2021-10-23 20:59:25
tags:
- Ham
- 业余无线电
---

记录下业余无线电通信中，出现的简语/短语/术语/概念。

<!-- more -->

##### ham

英文直译是“火腿”的意思，在业余无线电通信中，表示“业余”的含义，但在实际中，惯用为称呼“业余无线电爱好者”。

为什么叫`ham`，可看下维基百科：[Amateur radio - Wikipedia](https://en.wikipedia.org/wiki/Amateur_radio)，里面有介绍其历史和来源。

##### SDR

是Software Defined Ratio的缩写，即“软件定义无线电”。简单理解，就是硬件和软件相结合，组成一套功能又强大又丰富的无线电收发系统，可玩性和趣味性高出很多。大致原理是把信号处理的工作交给了通用处理器完成，这样做的一个优点是可以适配各式各样的通信协议。关于SDR的更多概念，参考维基百科：[Software-defined radio - Wikipedia](https://en.wikipedia.org/wiki/Software-defined_radio)

##### Web-SDR

这个概念扩展自SDR，把SDR搬到了互联网（表现形式一般是一个Web网站）上，让没有SDR硬件的爱好者能远程**接收**无线电信号。（通常只会开放接收，发送一般是被禁止的）

如果感兴趣，可以在这篇文章中找到相关网站：{% post_link record-some-hamradio-website 记录一些业余无线电相关的网站 %}。

##### 关于波长与频率

**波长**：固定频率下，沿着波的传播方向、在波的图形中，离平衡位置的”位移“和”时间“皆相同的两个质点之间的最短距离。

**频率**：单位时间内某事件发生的次数。

##### 长波（LW）/中波（LW）/短波（SW）

所谓“x波”表示的是波长的一个范围，我们知道波长和频率呈反比：
$$
\mathcal{c} = \lambda\mathcal{f}
$$
那么波长对应的会存在一个频率。

长波：Long Wave，没有明确的划分，一般波长超过1000m的成为长波，对应频率为300KHz。尽管没有明确规定，但长波的频率一般不会高于中波的分界线520KHz。

中波：Middle Wave，一般指波长范围为100米～1000米的电磁波，对应频率为300KHz～3MHz。

短波：Short Wave，波长范围10米～100米。

> 注意：
>
> 1. 这些英文缩写要符合特定上下文环境才能如此理解，否则，例如SW，某些条件下可理解为Software
> 2. 关于长/中/短波的特性此处不做过多阐述，感兴趣可以自行查阅资料了解下。这里列举几个：
>    1. [长波通讯、中波通讯、短波通讯和微波通讯的区别在哪？ - 湖南省工业和信息化厅 (hunan.gov.cn)](https://gxt.hunan.gov.cn/gxt/wxd_ycl/hyzx_ycl/kpyd_ycl/202103/t20210322_15034447.html)
>    2. [从极长波到长波的传播特性 | 中国无线电管理 (srrc.org.cn)](http://www.srrc.org.cn/article23562.aspx)
>    3. [Longwave - Wikipedia](https://en.wikipedia.org/wiki/Longwave)
>    4. [Medium wave - Wikipedia](https://en.wikipedia.org/wiki/Medium_wave)
>    5. [Shortwave radiation - Wikipedia](https://en.wikipedia.org/wiki/Shortwave_radiation)

##### 等幅电报通信（CW）

全称Continuous Wave，是一种通信方式。一般是利用摩斯电码发送信息，与其它无线电通信方式比，它的优点是对设备的要求简单、占用频宽窄、发射效率高、同等条件下传播距离更远。

参考：[在无线电术语，CW是什么意思呢？ - 〓电子管技术区〓 - 矿石收音机论坛 - Powered by Discuz! (crystalradio.cn)](http://www.crystalradio.cn/thread-307539-1-1.html)

