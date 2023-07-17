---
layout: post
title: "RT-Thread学习"
date: 2023-05-11
tags: ["混饭吃"]
toc: true
comments: true
author: vdadh
excerpt: "为接手公司产品的RTOS移植而学习"
---

原理源码分析，参考：

[RT-Thread - 矜辰所致的专栏 - 掘金 (juejin.cn)](https://juejin.cn/column/7135359793488724005)

FreeRTOS学习，参考：

[(5条消息) FreeRTOS(教程非常详细）_不秃也很强的博客-CSDN博客](https://blog.csdn.net/qq_61672347/article/details/125748646)

[2. 初识FreeRTOS — FreeRTOS内核实现与应用开发实战指南—基于STM32 文档 (embedfire.com)](https://doc.embedfire.com/rtos/freertos/zh/latest/zero_to_one/first_sight.html)

[FreeRTOS - 矜辰所致的专栏 - 掘金 (juejin.cn)](https://juejin.cn/column/7135654433915928584)

## 一、内核

### 1、内核启动流程

博客参考：[RT-Thread记录（二、RT-Thread内核启动流程 — 启动文件和源码分析） - 掘金 (juejin.cn)](https://juejin.cn/post/7136036194197962788)

#### 1-1.基础介绍

裸机程序：一般在 .s 文件中就跳转到 _main 从而跳转到 main() 函数启动；

RT-Thread：程序启动，通过 startup_xxxx.s 文件（汇编语言）跳转到 RT-Thread 启动函数 rtthread_startup()（进行一系列必要的初始化），再通过 rtthread_startup() 跳转到 main() 函数。

![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306210922275.png)

在 RT-Thread 中，会把 `main()`函数 当成是一个线程。这个在 `rtthread_startup()` 就会将 `main()` 创建成一个线程，除此之外，`rtthread_startup()` 还会创建`timer `线程 和 空闲线程 这两个线程。

#### 1-2.汇编部分 — startup_xxxx.s说明

startup_xxxx.s文件位置：

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306201545328.png" style="zoom:67%;" />

startup_xxxx.s文件说明：

![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306211156771.png)

#### 1-3.C部分 — rtthread_startup说明

搁置......

### 2、线程管理

<!--RT官方提及到一个案例：即一个任务中有两个线程，一个线程负责传感器采集，另一个线程负责数据显示，这个可以用于我的毕设MAX30102阻塞运行的bug上，两个线程切换的很快，就能让用户感觉OLED上一直都在显示数据变化-->

1. 每个线程都有重要的属性，如线程控制块、线程栈、入口函数等。

   线程调度为抢占式，处于就绪状态且任务优先级最高的线程，最先获取资源。

2. **线程控制块：**线程控制块是操作系统用于管理线程的一个数据结构，它会存放线程的一些信息，例如*优先级*、线程名称、线程状态等，也包含线程与线程之间连接用的链表结构，线程等待事件集合等。

3. **线程栈：**线程切换上下文存于线程栈中。或者放局部变量；

4. **线程优先级：**最大支持 256 个线程优先级 (0~255)，数值越小的优先级越高，0 为最高优先级。

   从就绪线程列表中查找最高优先级线程，保证最高优先级的线程能够被运行，*最高优先级的任务一旦就绪，总能得到 CPU 的使用权*。

5. **线程状态切换**：

   当线程调用 rt_thread_delay() 时，线程将主动挂起；当调用 rt_sem_take()，rt_mb_recv() 等函数时，资源不可使用也将导致线程挂起。

   ![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306042106372.png)

6. **线程入口函数**：用户编写，两种模式，它是线程实现预期功能的函数。

7. **线程创建：**

   * 使用 rt_thread_create() 创建一个*动态线程*，rt_thread_delete() 删除

   * 使用 rt_thread_init() 初始化一个*静态线程*， rt_thread_detach() 函数使得线程从线程队列和内核对象管理器中被脱离，自动调用。

   * 动态线程与静态线程的区别是：动态线程是系统自动从动态内存堆上分配栈空间与线程句柄（初始化 heap 之后才能使用 create 创建动态线程）<!--用户仅仅指明栈大小即可-->，静态线程是由用户分配栈空间与线程句柄<!--用户需创建栈（一个数组之类的），指明大小-->。

   * 动态 -- 创建和删除；静态 -- 初始化和脱离：

     ​	静态线程需要自己显示的定义一个 char 数组，也即在未调用 init() 函数之前，该数组将会被分配一段连续的内存空间，后续的 init() 函数调用只是对前面已经存在的一段内存空间进行赋值，基于此 RT 官方才会把静态线程的创建称为初始化线程；

     ​	而动态线程，我们一开始定义一个 rt_thread_t 类型的指针，初始化是 NULL，只有在调用 rt_thread_create() 成功之后，才会开辟出一块存放线程控制块的内存空间，从无到有的一个过程，将称为创建。

8. **线程销毁：**

   * 一种是用户主动调用 rt_thread_delete() 函数销毁；

   * 一种是运行状态的线程，在运行结束，自动调用 rt_thread_exit() 函数；

     以上两种方式，其销毁过程一样：先设置为 RT_THREAD_CLOSE 状态（后续不再参与调度），然后放入 rt_thread_defunct 僵尸队列（资源未回收，处于关闭状态的线程队列），最后空闲线程调度时，才会完成删除。

9. **空闲线程：**一种系统线程，（未调度前）一直处于就绪状态（不会进入挂起状态），其入口函数是一个死循环。RT_Thread中的空闲线程不是主动要求运行的任务，而是在其他任务都已经完成或者被阻塞时才会被调度，它只是单纯地占用CPU时间片。

10. **钩子函数：**

   * *空闲线程的钩子函数*：空闲线程执行前，调用用以做一些其他事情，比如系统指示灯、功耗管理、看门狗喂狗、CPU使用率。可以设置4个空闲钩子函数。<!--需要确保该钩子函数内部不可调用任何可能让空闲线程转换为挂起状态的操作-->
   * *调度器钩子函数*：有时用户可能会想知道在一个时刻发生了什么样的线程切换，可以通过调用下面的函数接口设置一个相应的钩子函数。*只能设置1个调度钩子函数*，在系统线程切换时，这个钩子函数将被调用。

11. 线程如果内部不是 while 死循环状态，那么它运行一次后，后续还会运行吗？也许它自动 delete了？

12. *时间片*：

    不同的时间片大小可能会对线程的调度产生影响。如果时间片设置得较大，线程就有更长的时间来运行，减少了被强行剥夺CPU的时间，可以保证线程能够有足够的时间完成任务，减少了线程切换的次数，提高了系统性能。但是过长的时间片也会让某些需要等待执行的线程无法得到CPU时间片，降低了CPU利用率。同时也会增加每个线程的响应时间。

    而时间片设置过短则可能导致线程切换频繁，调度开销变大，操作系统负担增加，会降低系统性能。另外，时间片长度还需要根据线程的类型、优先级和任务特性灵活配置，以达到最佳的系统效率与性能。
    
    <!--时间片就是时钟节拍的数量-->
    
13. `ps` 或 `list_thread` 命令可以查看当前系统所有的线程信息；

###    3、时钟管理

时钟节拍、定时器（基于时钟节拍）

#### 3-1.时钟节拍（OS Tick）

##### 3-1-1.概念与实现原理

时钟节拍是操作系统中最小的时间单位，是*特定的周期性中断*。

时钟节拍由配置为中断触发模式的硬件定时器产生，当中断到来时，将调用回调函数，回调函数中会把时钟节拍数加1；对于我们使用的 Corex-M 芯片来说，就是由滴答定时器 Systick 实现系统时钟节拍。

![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306211135043.png)

上图代码中的 全局变量，`rt_tick` 的值表示了系统从启动开始总共经过的时钟节拍数，即系统时间。rt_tick 在每经过一个时钟节拍时，值就会加 1。

##### 3-1-2.时钟节拍的长度

时钟节拍的长度 = 1/RT_TICK_PER_SECOND 秒，RT_TICK_PER_SECOND 在 rtconfig.h 中有定义，默认是 1000。那么系统的时钟节拍周期就为 1ms（1s 跳动 1000 下，每一下就为 1ms）<!--可于rtconfig.h中查看时钟节拍长度的定义-->；

所以，如果我们调整 RT_TICK_PER_SECOND 为 100，那么 1/100 = 10^-2s = 10ms，也即没 10ms，时钟节拍触发中断加1。

上述我们讲到系统 “心跳” —— 时钟节拍，由滴答定时器 Systick 实现，又提到 RT_TICK_PER_SECOND 可影响时钟节拍，其具体的实现代码见下图：

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306211149625.png" style="zoom:75%;" />

#### 3-2.定时器管理

##### 3-2-1.根据实现方式分类

1）**硬件定时器**：是芯片本身提供的定时功能。一般是由外部晶振提供给芯片输入时钟，芯片向软件模块提供一组配置寄存器，接受控制输入，到达设定时间值后芯片中断控制器产生时钟中断。硬件定时器的精度一般很高，可以达到纳秒级别，并且是中断触发方式。

2）**软件定时器**：是由操作系统提供的一类系统接口，它构建在硬件定时器基础之上，使系统能够提供不受数目限制的定时器服务。

3）注意：RT-Thread 操作系统提供软件实现的定时器，以时钟节拍（OS Tick）的时间长度为单位，即*定时数值必须是 OS Tick 的整数倍*，例如一个 OS Tick 是 10ms，那么上层软件定时器只能是 10ms，20ms，100ms 等，而不能定时为 15ms。RT-Thread 的定时器也基于系统的节拍，提供了基于节拍整数倍的定时能力。

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306211414726.png" style="zoom:67%;" />

##### 3-2-2.根据触发机制分类

1）单次触发定时器：根据设定的超时时间，当时间一到，触发一次定时器事件，执行超时函数。

2）周期性触发定时器：周期性的触发，直到用户手动停止，否则将永远持续执行下去。

##### 3-2-3.根据超时函数执行环境分类

1）HARD_TIMER 模式：该模式的定时器超时函数在中断上下文环境中执行，要求超时函数快进快出，不要执行时间非常长，也不要调用会让当前上下文挂起的系统函数。RT-Thread 定时器默认的方式是 HARD_TIMER 模式。

设置 HARD_TIMER 模式——通过 rt_timer_create() 的 flag 参数配置。

2）SOFT_TIMER 模式：该模式被启用后，系统会在初始化时创建一个 timer 线程，然后 SOFT_TIMER 模式的定时器超时函数在都会在 timer 线程的上下文环境中执行。

启用 SOFT_TIMER 模式——通过`rtconfig.h`中的宏定义 RT_USING_TIMER_SOFT 来决定是否启用；

设置 SOFT_TIMER 模式——通过 rt_timer_create() 的 flag 参数配置。

---

上述有关定时器的触发机制、超时函数执行环境配置，可以在 rt_timer_create() 的 flag 参数中设置。

flag 参数的可填值，见 include/rtdef.h 中定义的一些宏，通过 `|` 进行组合配置。

```c
#define RT_TIMER_FLAG_ONE_SHOT      0x0     /*单次定时*/
#define RT_TIMER_FLAG_PERIODIC      0x2     /*周期定时*/

#define RT_TIMER_FLAG_HARD_TIMER    0x0     /*硬件定时器*/
#define RT_TIMER_FLAG_SOFT_TIMER    0x4     /*软件定时器*/
```

##### 3-2-4.初始化工作

在系统启动时需要初始化定时器管理系统。可以通过下面的函数接口完成（详见rtthread_startup()函数）：

```c
void rt_system_timer_init(void); 
//该函数内部进行了硬件定时器链表的一个初始化工作
```

如果需要使用 SOFT_TIMER，则系统初始化时，应该调用下面这个函数接口：

```c
void rt_system_timer_thread_init(void);	//软件定时器初始化
```

### 4、线程间同步

消息队列：发送方放满了，接收方没及时获取，那么消息队列中就会顶掉部分数据，用以存放新发送数据。

信号量：

互斥量：两个线程访问一个资源，需要互斥避免冲突

事件集：比如一个线程完成某件事，通知另一线程

邮箱：之间发送到内容更多！

---

同步是指线程按照预定的先后顺序进行运行，对于多线程共同访问的的一块区域，称为共享内存块或*临界区*，临界区需要互斥访问，涉及线程执行顺序问题，即特殊的一种线程同步。

同步方式有：信号量、互斥量、事件集

#### 4-1.信号量

![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306051345517.png)

* 当信号量值大于零时，线程将获得信号量，并且相应的信号量值会减 1--take；

* 当信号量的值等于零时，并且有线程等待这个信号量时，释放信号量将唤醒等待在该信号量线程队列中的第一个线程，由它获取信号量；否则将把信号量的值加 1--release；

* **信号量使用场合**：

  （1）线程同步（2）中断与线程的同步（3）资源计数
  
* **疑点解答**：

  ![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306051346338.png)

#### 4-2.互斥量

* 互斥量是一种特殊的二值信号量（信号量始终在0、1之间变换），用以保护临界区（共享资源，就不要使用前面生产者消费者中的二值信号量了）；

* 使用互斥量，实现*优先级继承协议*，可以解决*优先级翻转*问题；

* 需要切记的是互斥量不能在中断服务例程中使用。

* **互斥量使用场合**：

  互斥量的使用比较单一，因为它是信号量的一种，并且它是以锁的形式存在。在初始化的时候，互斥量永远都处于开锁的状态，而被线程持有的时候则立刻转为闭锁的状态。互斥量更适合于：

  （1）线程多次持有互斥量的情况下。这样可以避免同一线程多次递归持有而造成死锁的问题。

  （2）可能会由于多线程同步而造成优先级翻转的情况。

#### 4-3.事件集

* 利用事件集可以完成一对多、多对多的线程同步。
* 每个线程可拥有 32 个事件标志，即一个 32 位的无符号整型变量，每一个 bit 代表一个事件；
* 每个线程拥有一个事件信息标记，RT_EVENT_FLAG_AND(逻辑与)，RT_EVENT_FLAG_OR(逻辑或）以及 RT_EVENT_FLAG_CLEAR(清除标记）——也即 32bit 之间可以是或运算（独立型同步--只要有一个等待的事件发生就激活线程），也可以是与运算（关联型同步--所有等待的事件都发生时才会激活线程），清除标记为可以把事件标志位 1 置为 0；

### 5、线程间通信

概述：前面线程同步只是调度上的一个限制，线程通信就涉及到线程间发消息了。

#### 5-1.邮箱

* *发送线程*：是将邮件复制到邮箱的，依照邮箱已满，对应有两种发送处理模式
  
  1. rt_mb_send（直接发送模式）：直接发送，看到邮箱已满，会返回错误码：“-RT_EFULL”；
  2. rt_mb_send_wait（等待发送模式）：等待挂起，根据超时时间来等待邮箱空出来，然后唤醒继续发送。如果超时后依然满，唤醒线程返回错误码。
  
* *接收线程*：从邮箱中复制出邮件，如果邮箱为空，根据设置的超时时间，等待挂起在邮箱的等待线程队列上，超时后依然为空，返回错误码：“-RT_ETIMEOUT”

* *邮箱初始化方法*：第4个参数--size 参数指定的是邮箱的容量，即如果 msgpool 指向的缓冲区的字节数是 N，那么邮箱容量应该是 N/4。

* 一个邮件的大小为 4 字节，所以一般用 char[] 定义邮件内容；

  ![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306051346867.png)


#### 5-2.信号

* *信号值*：应用程序能够使用的信号为 SIGUSR1（10）和 SIGUSR2（12）<!--只有这两个是开发给用户可以使用的-->

* *信号本质是软中断*，用来通知线程发生了异步事件，用做线程之间的异常通知、应急处理。

  一个线程不必通过任何操作来等待信号的到达，线程也不清楚信号何时到的。

  线程之间可以互相通过调用 rt_thread_kill() 发送软中断信号。

* *信号阻塞*：信号不会传递给安装此信号的线程，不会引发软中断处理；

* *信号处理方法*：

  1. 写一个函数，类似中断处理函数，线程收到信号后，就由于该函数负责处理；
  2. 忽略信号，不做处理；
  3. 对信号的处理保留系统的默认值

* *使用过程*：

  1. 给线程1安装信号；

     ```c
     /* 安装一个信号，返回安装结果。
     * 此函数有两个参数，分别为信号值和信号值的处理方法
     * signo：信号值(SIGUSR1 或 SIGUSR2)
     * handler：处理方式(SIG_IGN、SIG_DFL、自定义处理函数)
     * 返回值: 成功返回 handler 值，错误返回 SIG_ERR
     */
     rt_sighandler_t rt_signal_install(int signo, rt_sighandler_t handler);
     ```

     - 传入 SIG_DFL，即使用系统的默认方式，系统会调用默认的处理函数 _signal_default_handler()，它只是打印
     - 传入 SIG_IGN，就是忽略，接收到信号后不做任何处理

  2. 解除信号1阻塞；

     ```c
     void rt_signal_unmask(int signo);
     ```

  3. 此后，其他线程就可以发送信号，触发线程1处理该信号；

     ```c
     /* 发送信号。
     * tid：接收信号的线程
     * sig：信号值
     * 返回值: 成功返回 RT_EOK，错误返回-RT_EINVAL
     */
     int rt_thread_kill(rt_thread_t tid, int sig);
     ```

  4. 当信号被传递给线程 1 时，如果它正处于挂起状态，那会把状态改为就绪状态去处理对应的信号。
  
     如果它正处于运行状态，那么会在它当前的线程栈基础上建立新栈帧空间去处理对应的信号，需要注意的是使用的线程栈大小也会相应增加。

### 6、内存管理

* *两种内存管理方式*：*动态内存堆管理*和*静态内存池管理*；

* *RT内存管理中需要注意的地方*：

  1. 内存空间的查找时间需要注明，你不能告诉说查找时间不确定，在实时操作系统中对时间要求很严格。

  2. 不同的板子其内存资源不同（KB or MB），工作环境不同（有的在野外，不能说定期重启清除碎片），所以需要一套很好的内存分配算法。

  3. 内存分配管理算法总体上可分为两类：内存堆管理与内存池管理，而*内存堆管理又根据具体内存设备划分为三种情况*：

     第一种是针对小内存块的分配管理（小内存管理算法）；

     第二种是针对大内存块的分配管理（slab 管理算法）；

     第三种是针对多内存堆的分配情况（memheap 管理算法）

     ---

     * 小内存管理算法主要针对系统资源比较少，一般用于小于 2MB 内存空间的系统；

     * 而 slab 内存管理算法则主要是在系统资源比较丰富时，提供了一种近似多内存池管理算法的快速算法。

     * 除上述之外，RT-Thread 还有一种针对多内存堆的管理算法，即 memheap 管理算法。memheap 方法适用于系统存在多个内存堆的情况，它可以将多个内存 “粘贴” 在一起，形成一个大的内存堆，用户使用起来会非常方便。

     这几类内存堆管理算法在系统运行时只能选择其中之一或者完全不使用内存堆管理器，他们提供给应用程序的 API 接口完全相同。

### 7、自动初始化原理--源码分析

RT 定义了 6 个自动初始化宏定义，其本质是一个数据段，它们该有层级（层级1~6，层级1优先级最高，也即最先进行调用执行）；所以，在具体实现上将函数指针存储与对应层级的数据段中，即可。<!--上述描述详细说明，请见下图-->

![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306151805819.png)

---

![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306191150864.png)

博客参考：[RT-Thread 自动初始化详解_init_app_export_Nameless-Y的博客-CSDN博客](https://blog.csdn.net/yang1111111112/article/details/93982354)

### 其他

1. ALIGN( RT_ALIGN_SIZE)：功能是字节对齐

   ```c
   其实一般的对齐要求都是由硬件决定的,比如armv7-a的mmu表要求16k对齐, 如上面所说cache line也有对齐要求….还有比较常见的原因是某些cpu对内存的访问在地址对齐时速度比较快(比如访问u32时, 地址是4对齐时只要一次搬运就可以, 不对齐时需要两次)等等 , 综上所述由于硬件的限制或是出于性能的考虑,才需要有对对齐的需求.
   ```

   在使用小熊派编写的时候，"ALIGN( RT_ALIGN_SIZE)" 报出语法错误！改成小写？

   实际编译后，正确的是替换为：rt_align(RT_ALIGN_SIZE)

2. 我的一些疑惑解决：

   ![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306051348749.png)
   
3. 小熊派移植RT-Thread，下载项目后，按下复位键即可看到串口打印版本信息。

4. 使用句柄的，都是动态；一般静态，都是使用控制块，然后通过 &取址传入 init() 函数。

5. PIN脚编号，查看驱动文件 drv_gpio.c 确定。

6. 小熊派的 UART1 与 ST-LINK相连，用户通过串口1和小熊派通信；

   UART2--用户串口

   UART3--TFT_LCD

#### 调度器锁

1. **调度器锁**：

   是用于线程同步的一种方式，RT-Thread提供的调度器锁在使用时比较简单，只有上锁（rt_enter_critical）和解锁（rt_exit_critical）两个接口，但结合业务逻辑来说，则需注意，比如上述基本问题。

   *调度器锁*与*中断锁*类似，上锁后只有解锁后其他线程才能获取CPU资源执行；不同的是，调度器锁上锁后如有中断进入，系统仍然可以响应中断，中断锁则是屏蔽了包括中断在内的所有任务响应。

   根据调度锁特点，在业务应用层使用到调度锁时，需要考虑上锁后处理任务的复杂度和占用CPU资源，长时间占有CPU资源会降低系统的实时性及导致任务翻转（高优先级任务未能及时执行），或者中断响应的任务未能执行。
   ————————————————
   版权声明：本文为CSDN博主「pl0020」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
   原文链接：https://blog.csdn.net/pl0020/article/details/119753795

2. ![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306051348069.png)

3. 使用总结：

   1）“成对出现”，与malloc/free、new/delete内存分配类似，保证“成对出现”。调度锁上锁和解锁必须在同一线程内，理论上在线程内其他地方解锁都可以，但良好的习惯应该保证在同一函数内。<!--及时解锁-->
   2）可以嵌套使用，但仍要遵守“成对出现”规则，每一次上锁，对应一次解锁，RTT的调度器锁最大嵌套深度是 65535。
   3）注意逻辑条件，考虑是否存在某种条件下直接执行函数返回，但并未解锁。
   4）上锁任务占用的CPU资源应该尽可能小，并及时退出。

## 二、设备和驱动

### 1、I/O设备模型

博客参考：[RT-Thread记录（十、全面认识 RT-Thread I/O 设备模型） - 掘金 (juejin.cn)](https://juejin.cn/post/7144935396521017351#heading-24)

![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306121058284.png)

#### 1-1.基础介绍

1. *框架分为三层*：

   I/O设备管理层：负责提供接口供应用程序使用<!--rt-thread/src/设备管理层-->；

   设备驱动框架层：对同类硬件设备驱动的抽象，主要是抽取相同部分，不同部分留出接口<!--rt-thread/components/drivers/设备驱动框架层-->；

   设备驱动层：和硬件直接打交道的驱动程序，负责创建和注册 I/O 设备<!--drivers/驱动框架层（以drv开头命令）-->。

2. I/O设备类型：

   rt_device_t：设备句柄--动态创建

   字符设备：每次传输一个字节；

   块设备：每次传输多个字节（一个数据块），对于不足一个块的数据，需要合并空数据为一整个块读写

3. I/O设备创建注册：

   设备先创建，再注册到 I/O 设备管理器中，应用程序才能访问；

   在 FinSH 命令行使用 `list_device` 命令，可查看系统中所有注册成功的设备信息（名称、类型、被引用次数）
   
4. *设备控制块 & rt_device_t*：

   和前面学习的线程、信号量等都是一样的。RT 有着面向对象的思想，所有这些 IPC 机制都被当成一个对象，都有一个结构体控制块（指针-句柄）。<!--如下图为RT中拥有的控制块类型-->
   
   <img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306121111506.png" style="zoom:65%;" />
   
   <!--如下图为设备控制块定义代码-->
   
   <img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306121118503.png" style="zoom:65%;" />

#### 1-2.操作 API

1. *创建与注册*：RT 已提供；

2. *操作函数*：

   <img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306121154927.png" style="zoom:65%;" />

   我们在 I/O 设备管理层通过调用上图右侧的接口，即可实现对底层硬件的驱动操作。其底层实现还是调用了上图右侧的函数，也即 *设备控制块结构体中定义的哪一个函数*。所以，当我们自己创建一个新设备的时候，其底层的操作方法实现必不可少。

   在 rt_device_xxx() 函数内部，通过调用宏定义 device_xxx()，而实现底层调用 xxx() 的过程。

   <img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306121137398.png" style="zoom:65%;" />

   <!--底层 I/O 设备操作函数实现举例：-->

   <img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306121158787.png" style="zoom:65%;" />

#### 1-3.使用过程

参考源码：[D:\ProgramCode\RT_Program\HET-3SP_RT\drivers\my_drv_demo.c](其使用可见application\main.c)

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306100900731.png" style="zoom:85%;" />

### 2、UART设备--串口

博客参考：

[RT-Thread记录（十一、I/O 设备模型之UART设备 — 源码解析） - 掘金 (juejin.cn)](https://juejin.cn/post/7145447830478389285)

[RT-Thread记录（十二、I/O 设备模型之UART设备 — 使用测试） - 掘金 (juejin.cn)](https://juejin.cn/post/7146187976186265608#heading-8)

#### 2-1.初始化函数分析

![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306121548922.png)

#### 2-2.应用层与底层操作函数的关联

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306121601275.png" style="zoom:80%;" />

设备管理层使用 rt_device 通用设备控制块。

设备驱动框架层使用 rt_serial_device 控制块，其继承了`rt_device`的内容，同时还增加了 UART 设备特有的一些配置，操作，回调函数之类的内容。

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306131044702.png" style="zoom:80%;" />

#### 2-3.模式

串口数据接收和发送数据的模式分为 3 种：*中断模式、轮询模式、DMA 模式*。在使用的时候，这 3 种模式只能**选其一**，若串口的打开参数 oflags 没有指定使用中断模式或者 DMA 模式，则默认使用轮询模式（即流模式）。

DMA模式下，CPU被解放出来，串口的通信单独出来进行，所以使用 DMA 传输可以连续获取或发送一段信息而不占用中断或延时，在通信频繁或有大段信息要传输时非常有用。

**模式选择注意：**

*接收*：在我们正常的项目使用中，一般都是*中断接收*或者*DMA接收*，基本上不会使用轮询接收的方式（极大的浪费资源）；我们常用*信号量*或者*消息队列*来标志是否接收到串口数据，这样的好处是当没有数据的时候，会将数据处理线程挂机，让出CPU资源。

*发送*：*轮询模式发送*

```c
/*轮询方式发送，中断接收*/
rt_device_open(serial, RT_DEVICE_FLAG_INT_RX);
/*轮询方式发送，DMA接收*/
rt_device_open(serial, RT_DEVICE_FLAG_DMA_RX);
```

#### 2-4.实验例程

有关设备，通过 rt_device_t 句柄管理，使用串口，需要根据串口名查找设备；得到了设备，才能读写串口信息；

```c
serial = rt_device_find(uart_name);	//查找指定的串口设备
/* 以读写及中断接收方式打开串口设备 */
rt_device_open(serial, RT_DEVICE_OFLAG_RDWR | RT_DEVICE_FLAG_INT_RX);
/* 发送字符串 */
rt_device_write(serial, 0, str, (sizeof(str) - 1));
```

具体可以[RT-Thread API参考手册: pin_beep_sample.c](https://www.rt-thread.org/document/api/pin_beep_sample_8c-example.html))中查看 “uart_sample.c” 例程；

回车（'\r'）：把光标至于本行行首----0x0D；

换行（'\n'）：把光标置于下一行的同一列--0x0A；

windows下 enter 是 \n\r，unix 下是 \n，mac下是 \r

但是到我跑到小熊派上是：*\r*，代码参考：[D:\ProgramCode\RT_Program\BearPi01\my-sample\test03_sample.c]()

### 3、PIN设备--LED

PIN 设备的使用，同样是通过设备驱动框架层接口中转调用底层操作函数的。

![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306131126531.png)

#### 3-1. 获取 PIN 索引号方法

在 RT 中，针对芯片的 GPIO 进行顺序编号，比如 PA0-15：对应编号 0 ~ 15，PB0-15：16 ~ 31。有关编号，可以用户自己写上去<!--就是自己数一下-->

1. 自己数

   ```c
   /* KEY1按键的PIN脚编号，查看驱动文件/driver/drv_gpio.c确定 */
   #define KEY1_PIN_NUM        18  //PB2
   #define LED_PIN_NUM         45  //PC13
   ```

2. 使用函数`rt_pin_get()`

   ```c
   //获取索引号
   pin_number = rt_pin_get("PA.9"); // pin_number 就是索引号
   //设置GPIO模式
   rt_pin_mode(pin_number , PIN_MODE_INPUT_PULLUP);
   ```

3. 使用宏定义`GET_PIN`

   该函数需要引入头文件`<drv_common.h` 、`"drv_common.h"`：
   
   同时 `<rtdevice.h>` 头文件也不可少！
   
   <img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306051348175.png" style="zoom:80%;" />tsv
   
4. 电平翻转：

   ```c
   rt_uint8_t led_status;
   led_status = rt_pin_read(LED_PIN_NUM);  //读取LED当前电平
   rt_pin_write(LED_PIN_NUM, led_status^1);    //异或实现翻转电平, 0x00-->0x01, 0x01-->0x00
   ```

   

#### 3-2.按键

请参考：

[使用STM32CubeMX新建小熊派的STM32L431RCT6工程实现按键控制LED（循环查询&外部中断）](https://blog.csdn.net/qq_42754570/article/details/112668379)

[RT-Thread Studio与CubeMX联合编程](https://blog.csdn.net/qq_40824852/article/details/123067421)

### 4、ADC 和 DAC 设备

ADC 是模数转换器，是指将连续变化的模拟信号转换为离散的数字信号的器件。设备使用流程如下图：

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306201424793.png" style="zoom:67%;" />

DAC 指数模转换器。是指把二进制数字量形式的离散数字信号转换为连续变化的模拟信号的器件；设备使用流程如下图：

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306201427571.png" style="zoom:67%;" />

### 5、I2C总线设备

一般情况下都是 MCU 作为 I2C 主机，所以 RT 将 I2C 主机虚拟为 I2C 总线设备，I2C 从机通过接口和 I2C 总线通讯。

所以首先将需要根据 I2C 总线设备名称获取设备句柄——rt_device_t rt_device_find(const char* name);

然后便可以读写。不管读还是写，都使用 RT 的 `rt_i2c_transfer()` 进行数据传输。

有关`rt_i2c_transfer()` 函数 <!--注：此函数会调用 rt_mutex_take(), 不能在中断服务程序里面调用，会导致 assertion 报错-->

#### 5-1.软件 I2C

1. 首先需要在 RT-Thread Setting 中开启软件模拟 I2C；
2. 然后需要在 board.h 中设置 I2C 引脚：

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306051349928.png" style="zoom:65%;" />

3. *注意：*RT 把地址和读写位是分开的，底层发送的数据是将地址*左移1位*再或上读写位。所以当我们使用软件模拟 I2C 时，从机地址*首先右移一位*，之后 RT 底层会自动左移一位，这样便还原了。相关的解释网址如下：https://club.rt-thread.org/ask/question/8065.html

4. 示例代码：使用教学热电主板驱动教学霍尔上的ADC--MS1100，我使用的便是软件模拟 I2C 组件；

   我认为的基本步骤分为：

   * 查找设备：rt_i2c_bus_device_find("i2c1")
   * 读写寄存器：rt_i2c_transfer()

   但是比较离谱的一点是，上述两个函数好像不能一同调用，我的解决方案就是，把两个步骤单独放到一个函数中，一个接着一个的调用。

   而且存在的一个 Bug 是：我无法对 MS1100 进行 I2C 写操作，但是当我开启了 `sht3x probe i2c1`，然后我就可以成功写 MS1100 了，比较离谱！

   代码参考：[D:\ProgramCode\RT_Program\HET-3SP_RT\applications\i2c_ms11000.c]()

   <!--上述代码仅仅实现了对MS1100的一个读操作，也即查找到i2c设备后，MS1100处于默认配置-->

5. MS1100读接口：应吴超的要求，接口要求每次调用也就能返回 MS1100 转换电平

   代码参考：[D:\ProgramCode\RT_Program\HET-3SP_RT\applications\ms11000_ops.c]()

#### 5-2.硬件 I2C

RT官方是并没有写好移植的，找到一个博客，依照官方的软件模拟 I2C 来编写硬件驱动。

参考博客：[之前刚开始接触RTT的I2C驱动框架，说实在，感觉有点奇怪。RTT默认只给了软件模拟的I2C，没有硬件的I2C。后来接下来的时间里，都是暂时用着吧，反正能用，也 (yinxiang.com)](https://app.yinxiang.com/fx/505cee77-1e12-4cd3-8999-3a473cf4f0bf)

### 6、USB Host-DFS-FATFS

教学设备具备测量数据的导出至 U 盘的功能，单片机主板作为 USB 主机。官方的是使用 Env 工具来进行配置的，目前主流使用 Studio 的 Settings 来配置显然更加方便。

#### 6-1.基本步骤

1. 开启 DFS、Fatfs组件：

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306051357981.png" alt="image-20230605135744028" style="zoom:65%;" />

Fatfs配置项中，设置*要处理的最大扇区大小* 为 `4096`：

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306061737930.png" alt="image-20230606173728558" style="zoom:67%;" />

2. 单片机作为 USB 主机，开启配置：

   *切记开启使能驱动程序，设置挂载点*

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306061734905.png" alt="image-20230606173457793" style="zoom:67%;" />

​	以上开启后，工程中的变化有：

* rt_config.h：

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306061650350.png" alt="image-20230606165050615" style="zoom:67%;" />

3. CubeMX配置：

   <!--不建议通过RT项目内置的CubeMX进行配置，最好我们自己另外新建一个CubeMX工程进行配置，因为考虑到后续项目开发更换芯片的可能，所以必须尽量保证驱动代码的后期修改的方便。以下图片只是展示USB配置，前置的RCC、SYS、时钟配置依照芯片进行-->

   <img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306061646435.png" alt="image-20230606164612554"/>

   * CubeMX配置好后，需要将生成的 HAL_HCD_MspInit()、HAL_HCD_MspDeInit()两个函数代码，从CubeMX工程的 stm32xxxx_hal_msp.c 复制到 board.c 文件中；

   * 在 board.c 中配置时钟：

     <img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306061612614.png" alt="image-20230606161237343" style="zoom:67%;" />

     具体的配置值，在 board.h 头文件中进行：

     <img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306061757108.png" alt="image-20230606175756845" style="zoom:67%;" />

   * stm32f4xx_hal_config.h 中开启 `#define HAL_UART_MODULE_ENABLED`。改步骤没有的话，会报出句柄缺失的错误。


#### 6-2.使用

文件管理、目录管理，对文件或目录的打开、关闭、读取、写入操作，都有一套函数接口可供使用。

```c
/*官方是说只有这一个头文件，但是新版本的 RT，这个头文件不存在了*/
#include <dfs_posix.h> 

/*替换上方头文件，为以下两个头文件解决问题*/
#include <fcntl.h> //当需要使用文件操作时, 需要包含这个头文件
#include <unistd.h> //文件读写和文件关闭函数报出警告, 添加该头文件解决
```

#### 6-3.博客参考

* [STM32 上使用 USB Host 读写 U 盘](https://www.rt-thread.org/document/site/#/rt-thread-version/rt-thread-standard/application-note/driver/usb/an0046-rtthread-driver-usbh?id=stm32-上使用-usb-host-读写-u-盘)
* [基于RTT-Thread Studio STM32F407ZG的U盘挂载](https://blog.csdn.net/weixin_43745583/article/details/121928171)

  

#### 6-4.概念知识

DFS 是 RT 提供虚拟文件系统组件，FATFS 是一个文件系统、具体的存储设备包括 SD 卡、SPI FLASH

先进行挂载管理，然后就可以进行文件读写、目录创建删除等操作，具体的都有提供 API；

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202307071125561.png" style="zoom:67%;" />



---

> 需要注意的问题：

1. 删除目录：`rm -rf  目录名`  该命令会就目录删除，同时删除其中的所有内容；

2. 创建目录：例如 `mkdir /very/haha` 该命令并没有在根目录下创建一个 very 目录，也没有创建其子目录 haha；

   目录创建只能一级一级的、一个一个的创建，也即先创建 very，再创建 haha；

   所以只有 very 目录首先存在了，我们才可以通过 `mkdir /very/haha` 创建 haha。

3. U盘显示文件或目录损坏且无法读取（U盘提示无法访问）解决方法：

   * win+R+输入 cmd：打开 Windows 控制台命令窗口；
   * 敲入命令：chkdsk g:/f  <!--其中的g是指你U盘的盘符，比如我的U盘盘符为f，这里应该输入命令 chkdsk f:/f，要根据实际情况而改变-->


### 7、CAN设备



## 三、传感器驱动框架

Sensor 驱动框架的作用是：为上层提供统一的操作接口，提高上层代码的可重用性；简化底层驱动开发的难度，只要实现简单的 *ops*(operations: 操作命令) 就可以将传感器注册到系统上。

## 其他

#### 裸机系统与多任务系统

了解一下整个编码的发展过程，参考：[5. 裸机系统与多任务系统 — FreeRTOS内核实现与应用开发实战指南—基于STM32 文档 (embedfire.com)](https://doc.embedfire.com/rtos/freertos/zh/latest/zero_to_one/multi_task.html)

#### rt_kprintg输出浮点数

关于RT官方预留的打印功能rt_kprintf无法输出小数的问题，其解决办法如下：

![](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306051349261.png)

参考博客：[RT-thread rt_kprintf()函数格式化输出浮点数_rt_kprintf 浮点数_plokm789456的博客-CSDN博客](https://blog.csdn.net/plokm789456/article/details/107087502)

`注明`：上述输出浮点数的修改方法，会导致 msh（也即终端--串口）崩溃，所以慎用。麻烦一点的浮点数输出，便是输出整数位+小数位：

<img src="https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306051349396.png" style="zoom:80%;" />

---

```c
humidity---float类型变量
rt_kprintf("sht3x humidity   : %d.%d \n", (int)dev->humidity, (int)(dev->humidity * 10) % 10);
rt_kprintf("sht3x temperature: %d.%d \n", (int)dev->temperature, (int)(dev->temperature * 10) % 10);
```

#### 线程栈大小的设定

我习惯于使用线程句柄动态创建，其中线程*栈空间*建议设置为 `8192` 字节，*优先级*建议设置为`20以后`，*时间片*设置任意。 

#### board.c 与 board.h

在 board.c 中很重要的一个点是，这里面有*时钟配置*函数的调用，有了这个我们就不需要通过 CubeMX 去配置时钟。

![image-20230606161237343](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306061612614.png)

#### RT_EOK 与 -RT_ERROR

​	在官方的函数中，经常可以看到使用这两个宏作为函数返回值。通常情况下 `return RT_EOK` 表示执行成功，`return -RT_ERROR` 表示执行失败。也即，返回 `非0` 的一律认为执行错误。

![image-20230607145249564](https://vdadh-picture-bed.oss-cn-hangzhou.aliyuncs.com/img/202306071452194.png)

#### 日志

```c
LOG_E("错误级别日志");
LOG_W("警告级别日志");
LOG_I("提示级别日志");
LOG_D("调试级别日志");
LOG_RAW("输出raw日志")
```

#### rt_kprintf()打印问题

对于 `rt_uint8_t` 类型变量，通过`%u`格式符指定了要打印的变量`value`的格式为无符号十进制数。你可以根据需要选择其他格式符，如`%x`（十六进制）、`%d`（有符号十进制）等。

```c
rt_uint8_t value = 0x8C;
rt_kprintf("The value is 0x%02X\n", value);	//The value is 0x8C
```

`double` 类型变量：`%f`

#### 联合STM32CubeMX进行开发

[RT-Thread Studio联合STM32CubeMX进行开发 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/395106066)