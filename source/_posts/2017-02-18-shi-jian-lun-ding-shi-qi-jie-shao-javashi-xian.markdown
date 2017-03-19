---
layout: post
title: "时间轮定时器介绍-Java实现"
date: 2017-02-18 22:03:27 +0800
comments: true
categories: java Algorithm
tags: [arthurhoo arthurhoo's blog TimeWheel 时间轮]
keywords: arthurhoo arthurhoo's blog TimeWheel 时间轮 定时器
---

## 0. 背景

很多中间件内部需要一些定时任务。这类定时任务最简单的是依赖quartz等定时组件实现。依赖第三方包，在进行组件升级的时候会有不兼容的问题，比如spring4对quartz的依赖接口就
改变了，必须升级quartz。第三方包的api变化是一个很大问题。为了做到自包含，可以自己实现一个定时器。本文将详细介绍时间轮算法的思想和实现。<!-- more -->

时间轮定时结构（TimeWheel）是George Varghese和Anthony Lauck于1997年在论文[《Hashed and Hierarchical Timing Wheels:Efficient Data Structures for Implementing a Timer Facility》](https://pdfs.semanticscholar.org/e04e/548b287d48260def0ffe09692236d2a56ad3.pdf)中提出的。
目前在Linux内核、netty中都有应用。

下图可以很通俗清晰地阐述时间轮的机制：

![image](https://zos.alipayobjects.com/rmsportal/rWjuDLxWYmOUjbZxHMeL.gif)

TimeWheel结构原理很简单，类似钟表，有一个表盘，按照一定的时间间隔划分为多个槽（Slot)，每个槽标号，并且在每个槽上挂载一个列表。比如计时一分钟的时间轮，一般分为60个格子，每一个固定时间间隔（tick)走动一格。在进行初始化的时候，相对于当前指针延后相同时间间隔的任务挂载在同一个时刻
slot的链表上。当时间指针指向某个槽的时候，当前槽的任务集合中的所有任务被执行。

根据George的论文描述，时间轮的实现主要推荐两种方式，单表盘和分层表盘结构。

### 单表盘：

整个定时器只有一个表盘，用循环数组代表表盘，每个槽的时间间隔（tick)代表了定时器能够达到的定时精度，循环数组的大小代表了整个定时器可以定时的时间范围。比如说要一个可以最大定时一分钟的时间轮，其表盘分为60个槽，每隔1秒钟（tick)，指针转动一个槽.netty中就使用了这类结构实现时间轮。
单表盘时间轮的好处是实现简单，定时精确，但是也有个问题是所能代表的时间有限。设想一下如果要表示一个计时范围1小时，精度1秒的时间轮，那么循环数组的大小就需要3600，如果标识一天呢，就需要3600*24. 

### 分层时间轮：

分层时间轮模拟了时间进制的方式来组织数据结构。按照时间精度的等级采用多个表盘来表示不懂精度的时间轮，不同精度的时间轮设置不同的tick转动。比如一个可以计时范围1天，精度1秒的时间轮，会有3个表盘标识，分别是小时级表盘（HourDial)、分钟级表盘(MinuteDial)和秒级表盘(SecondDial)。HourDial每小时转动一次，MinuteDial每一分钟转动一次，SecondDial每一秒钟转动一次，各个表盘互不干扰。在这种结构下，要表示一天的时间跨度所需要的空间slot数目如下：

> 24(小时)+60(分钟)+60(秒) = 144

分层时间轮的特点是标识的时间范围大，对空间要求低，可以方便扩展精度和时间范围。缺点是实现较为复杂。


## 时间轮（TimeWheel）实现

综合考虑未来的可扩展性、精度和空间消耗的需求，采用分层时间轮的结构实现定时器。下面详细讲述一下定时器实现的几个关键要素和思路。

根据论文[《Hashed and Hierarchical Timing Wheels:Efficient Data Structures for Implementing a Timer Facility》](https://pdfs.semanticscholar.org/e04e/548b287d48260def0ffe09692236d2a56ad3.pdf)的描述，
TimeWheel主要包含四个：
* 初始化组件：包括时间表盘、指针初始化以及任务挂载等；
* 停止组件：停止计时，清理现场
* tick设定和初始化
* 任务的具体执行处理器：负责具体执行任务

上面几个组件是论文的理论划分，具体的实现中尽量按照这样切分功能块，但也没有必要完全实现每一个部分。

我们距离设计一个定时范围1个月，精度1秒的定时器。


#### 任务抽象
为了让定时器能够统一处理系统内部所有任务，我们需要对所有任务进行抽象，先抽象一个AbstractTask,所有任务只要继承自这个抽象类，都可以被时间轮挂载和执行。

```java
public abstract class AbstractTask implements Runable {

    protected Long    period;

    protected Integer secondPeriod = 0;

    protected Integer minutePeriod = 0;

    protected Integer hourPeriod   = 0;

    protected Integer dayPeriod    = 0;

    protected String  taskId; //任务的唯一标识

}
```

#### 表盘结构

TimeWheel定时范围1个月，精度1秒，所以我们需要4级时间表盘，分别为：

* 秒级表盘(dialInSeconds)---每秒转动1次
* 分钟级表盘(dialInMinutes)---每分钟转动1次
* 小时级表盘(dialInHours) ---- 每小时转动一次
* 天级表盘（dialInDays) --- 每24小时转动一次

由于表盘的主要作用是读取当前的刻度，所以在初始化之后只有读操作没有写，所以我们直接使用ArrayList表示。

每一个表盘需要划分多个槽，在槽结构上封装任务的挂载和删除、查询等相关操作。所以定义了Slot类：

```
/**
     * 时间轮槽数据结构 用于存储定时任务
     */
    private class Slot<T> {

        private Vector<T> expiredTask = new Vector<T>();

        /**
         * 添加任务
         * 
         * @param task
         */
        void insertTask(T task) {
            expiredTask.add(task);
        }

        /**
         * 判断是否有可执行任务
         * 
         * @return
         */
        boolean hasTask() {
            return !this.expiredTask.isEmpty();
        }

        /**
         * 获取制定索引的任务
         * 
         * @param index
         * @return
         */
        T getTask(int index) {
            if (index < 0) {
                throw new IllegalArgumentException(
                    "index of slot can not be less than zero.");
            }

            if (index >= this.expiredTask.size()) {
                throw new IllegalArgumentException(
                    "index of slot can not be more than the count which exiting in slot.");
            }
            return hasTask() ? this.expiredTask.get(index) : null;

        }
   
        void removeTask(AbstractTask task) {
            this.expiredTask.remove(task);
        }
    }

```

Slot是TimeWheel中的一个私有内部类。最后形成的表盘结构如下：
```
        dialInSeconds = new ArrayList<Slot<AbstractTask>>(60);
        dialInMinutes = new ArrayList<Slot<AbstractTask>>(60);
        dialInHours = new ArrayList<Slot<AbstractTask>>(24);
        dialInDays = new ArrayList<Slot<AbstractTask>>(30);
```

#### 定时驱动线程池
为了让四个表盘能够互不干涉的按指定的tick运行，我们设置了表盘驱动。

```
    private ScheduledExecutorService      timeSecondDriver;                                                        // 秒针驱动

    private ScheduledExecutorService      timeMinuteDriver;                                                        // 分钟驱动

    private ScheduledExecutorService      timeHourDriver;                                                          // 小时驱动

    private ScheduledExecutorService      timeDayDriver;                                                           // 天级驱动

```
初始化时设定不同的定时间隔时间。

```
timeSecondDriver.scheduleAtFixedRate(secondDriver, 0, 1000, TimeUnit.MILLISECONDS);
timeMinuteDriver.scheduleAtFixedRate(minuteDriver, 0, 60, TimeUnit.SECONDS);
timeHourDriver.scheduleAtFixedRate(hourDriver, 0, 60, TimeUnit.MINUTES);
timeDayDriver.scheduleAtFixedRate(dayDriver, 0, 24, TimeUnit.HOURS);
```

#### 表盘任务线程

由于每次表盘转动定时器要做的工作比较类似，所以抽象为一个统一的线程任务TimeDriverThread。
表盘转动任务主要做两件事情：

* 秒级表盘上对到期的任务进行执行
* 其他表盘上的到期任务进行降级处理

初始化挂载的时候就将任务挂载到最粗时间精度表盘；当粗粒度表盘到期后，将该Slot上的任务重新挂载到次粗粒度表盘。
比如一个任务taska是1小时15分30秒执行一次。初始化的时候将它挂载到dialInHours[1],当小时数到期后，对其根据分钟数降级，挂载到`分钟表盘当前时间后15个slot`的位置；15分钟后，再将该任务挂载到dialInSeconds表盘`距离当前指针位置30个slot`的地方。
这里我们采用求模运算，挂载的时间复杂度为O(1).秒级盘到期后，由任务处理器执行日志打印任务。

#### 任务执行处理器
ExpiredWorkProcessor是执行具体任务的处理器。他在秒级盘到期之后，执行具体任务的run方法。
```
final private class ExpiredWorkProcessor {

        // work threads
        private ThreadPoolExecutor workThreads;

        ExpiredWorkProcessor(Integer threadCount) {
            workThreads = (ThreadPoolExecutor) Executors.newFixedThreadPool(threadCount,
                new ThreadFactory() {

                    final AtomicLong count = new AtomicLong(0);

                    @Override
                    public Thread newThread(Runnable r) {
                        return new Thread(new ThreadGroup("TimeWheelTest"), r, "ExpiredWorkProcessor-"
                                                                          + count.getAndIncrement());
                    }
                });
        }
```
这是一个线程池，默认10个线程来执行具体任务。可以通过初始化的时候设定workThreads来修改线程池大小。

### 运行过程

#### 初始化过程

时间轮的初始化过程可以用下图标识：

```
st=>start: Start
e=>end
initialDial=>operation: 初始化表盘和驱动线程等
initialProcess=>operation: 初始化任务处理器
registerTask=>operation: 注册任务
io=>inputoutput: throw Error...|request
cond=>condition: 初始化成功 or 初始化失败?

st->initialDial->initialProcess->cond
cond(yes)->registerTask->e
cond(no)->io->e
```


![imgs](https://zos.alipayobjects.com/rmsportal/bDCelvYHvhmhsTFQyBEr.jpg)

如上图所示，时间路你启动过程大概经过如下过程：

1.  检查时间轮是否启动，如果没有启动，进行表盘和处理器的初始化；
2.  初始化成功，挂载任务。 挂载任务到对应的最粗粒度表盘。比如taska的dayPeriod>0,那么就将它挂载到天级表盘，如果dayPeriod=0，hourPeriod>0,则挂载到小时级表盘，以此类推。

#### 定时执行过程
 1.  定时驱动器按照各自表盘的时间间隔修改表盘的指针
 2.  转到特定slot时，如果slot上有任务，则进行以下两种处理：

    * 如果是非秒级盘到期，则将slot上的任务按照低一级的时间精度挂在到次粗粒度表盘，比如小时表盘的任务下放到分钟表盘

    * 如果是秒级则直接执行任务run方法

#### 重新挂载过程
定时任务都都是周期性的，所以在执行任务之后还需要重新挂载。在每个任务执行完成之后，会重新挂载任务到初始状态。


### 总结

本文讲述了全局定时器的时间轮算法实现方式，如有不妥欢迎大家指出。 

