# RT-Thread



# 内核

# **线程管理**

在多线程操作系统中，也同样需要开发人员把一个复杂的应用分解成多个小的、可调度的、序列化的程序单元，当合理地划分任务并正确地执行时，这种设计能够让系统满足实时系统的性能及时间的要求

![传感器数据接收任务与显示任务的切换执行](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/thread/figures/04Task_switching.png)

线程是实现任务的载体，是 RT-Thread 中最基本的调度单位，对任务设置优先级，轮流运行

当线程运行时，它会认为自己是以独占 CPU 的方式在运行。



### 线程管理的功能特点

主要功能是对线程进行管理和调度。

系统中总共存在两类线程，系统线程和用户线程，系统线程是由 RT-Thread 内核创建的线程，用户线程是由应用程序创建的线程。

![对象容器与线程对象](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/thread/figures/04Object_container.png)

线程调度器是抢占式的，主要的工作就是查找运行最高优先级线程



### 线程的工作机制

#### 线程控制块

在 RT-Thread 中，线程控制块由结构体 struct rt_thread 表示，线程控制块是操作系统用于管理线程的一个数据结构，它会存放线程的一些信息。

![image-20241105193052651](C:\Users\FFZNB\AppData\Roaming\Typora\typora-user-images\image-20241105193052651.png)

​	init_priority 是线程创建时指定的线程优先级，在线程运行过程当中是不会被改变的，除非用户执行线程控制函数进行手动调整线程优先级。

​	cleanup 会在线程退出时，被空闲线程回调一次以执行用户设置的清理现场等工作。

​	最后的一个成员 user_data 可由用户挂接一些数据信息到线程控制块中，以提供一种类似线程私有数据的实现方式。

#### 线程重要属性

##### 线程栈

当进行线程切换时，会将当前线程的上下文存在栈中，当线程要恢复运行时，再从栈中读取上下文信息，进行恢复。

线程栈存放函数中的局部变量：函数中的局部变量从线程栈空间中申请；函数中局部变量初始时从寄存器中分配（ARM 架构），当这个函数再调用另一个函数时，这些局部变量将放入栈中。

对于线程第一次运行，可以以手工的方式构造这个上下文，来设置一些初始的环境：入口函数（PC 寄存器）、入口参数（R0 寄存器）、返回位置（LR 寄存器）、当前机器运行状态（CPSR 寄存器）。

线程栈的增长方向是芯片构架密切相关的，对于 ARM Cortex-M 架构，线程栈可构造如下图所示。

![线程栈 (ARM)](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/thread/figures/04thread_stack.png)

线程栈大小设定：

对于资源相对较大的 MCU，可以适当设计较大的线程栈；

或初始时设置较大的栈，然后在 FinSH 中用 list_thread 命令查看线程运行的过程中线程所使用的栈的大小，通过此命令，能够看到从线程启动运行时，到当前时刻点，线程使用的最大栈深度，而后加上适当的余量形成最终的线程栈大小，最后对栈空间大小加以修改。

##### 线程状态

在 RT-Thread 中，线程包含五种状态，操作系统会自动根据它运行的情况来动态调整它的状态。

| 状态     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| 初始状态 | 当线程刚开始创建还没开始运行时就处于初始状态；在初始状态下，线程不参与调度。此状态在 RT-Thread 中的宏定义为 RT_THREAD_INIT |
| 就绪状态 | 在就绪状态下，线程按照优先级排队，等待被执行；一旦当前线程运行完毕让出处理器，操作系统会马上寻找最高优先级的就绪态线程运行。此状态在 RT-Thread 中的宏定义为 RT_THREAD_READY |
| 运行状态 | 线程当前正在运行。在单核系统中，只有 rt_thread_self() 函数返回的线程处于运行状态；在多核系统中，可能就不止这一个线程处于运行状态。此状态在 RT-Thread 中的宏定义为 RT_THREAD_RUNNING |
| 挂起状态 | 也称阻塞态。它可能因为资源不可用而挂起等待，或线程主动延时一段时间而挂起。在挂起状态下，线程不参与调度。此状态在 RT-Thread 中的宏定义为 RT_THREAD_SUSPEND |
| 关闭状态 | 当线程运行结束时将处于关闭状态。关闭状态的线程不参与线程的调度。此状态在 RT-Thread 中的宏定义为 RT_THREAD_CLOSE |

##### 线程优先级

表示线程被调度的优先程度。

##### 时间片

每个线程都有时间片这个参数，但时间片仅对优先级相同的就绪态线程有效。系统对优先级相同的就绪态线程采用时间片轮转的调度方式进行调度时，时间片起到约束线程单次运行时长的作用，其单位是一个系统节拍（OS Tick）。

![相同优先级时间片轮转](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/thread/figures/04time_slience.png)

##### 线程的入口函数

线程控制块中的 entry 是线程的入口函数，它是线程实现预期功能的函数。线程的入口函数由用户设计实现，一般有以下两种代码形式：

**无限循环模式**：

在实时系统中，线程通常是被动式的。这个是由实时系统的特性所决定的，实时系统通常总是等待外界事件的发生，而后进行相应的服务：

```c
void thread_entry(void* paramenter)
{
    while (1)
    {
    /* 等待事件的发生 */

    /* 对事件进行服务、进行处理 */
    }
}

```

**顺序执行或有限次循环模式**：

```
static void thread_entry(void* parameter)
{
    /* 处理事务 #1 */
    …
    /* 处理事务 #2 */
    …
    /* 处理事务 #3 */
}

```

##### 线程错误码

一个线程就是一个执行场景，错误码是与执行环境密切相关的，所以每个线程配备了一个变量用于保存错误码，线程的错误码有以下几种：

```c
#define RT_EOK           0 /* 无错误     */
#define RT_ERROR         1 /* 普通错误     */
#define RT_ETIMEOUT      2 /* 超时错误     */
#define RT_EFULL         3 /* 资源已满     */
#define RT_EEMPTY        4 /* 无资源     */
#define RT_ENOMEM        5 /* 无内存     */
#define RT_ENOSYS        6 /* 系统不支持     */
#define RT_EBUSY         7 /* 系统忙     */
#define RT_EIO           8 /* IO 错误       */
#define RT_EINTR         9 /* 中断系统调用   */
#define RT_EINVAL       10 /* 非法参数      */
```

#### 线程状态切换

RT-Thread 提供一系列的操作系统调用接口，使得线程的状态在这五个状态之间来回切换。

几种状态间的转换关系如下图所示：

![线程状态转换图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/thread/figures/04thread_sta.png)

线程通过调用函数 rt_thread_create/init() 进入到初始状态（RT_THREAD_INIT）；

初始状态的线程通过调用函数 rt_thread_startup() 进入到就绪状态（RT_THREAD_READY）；

就绪状态的线程被调度器调度后进入运行状态（RT_THREAD_RUNNING）；

当处于运行状态的线程调用 rt_thread_delay()，rt_sem_take()，rt_mutex_take()，rt_mb_recv() 等函数或者获取不到资源时，将进入到挂起状态（RT_THREAD_SUSPEND）；

处于挂起状态的线程，如果等待超时依然未能获得资源或由于其他线程释放了资源，那么它将返回到就绪状态。挂起状态的线程，如果调用 rt_thread_delete/detach() 函数，将更改为关闭状态（RT_THREAD_CLOSE）；

而运行状态的线程，如果运行结束，就会在线程的最后部分执行 rt_thread_exit() 函数，将状态更改为关闭状态。

#### 系统线程

系统线程是指由系统创建的线程，用户线程是由用户程序调用线程管理接口创建的线程，在 RT-Thread 内核中的系统线程有空闲线程和主线程。

##### 空闲线程

空闲线程（idle）是系统创建的最低优先级的线程，线程状态永远为就绪态。当系统中无其他就绪线程存在时，调度器将调度到空闲线程，它通常是一个死循环，且永远不能被挂起。另外，空闲线程在 RT-Thread 也有着它的特殊用途：

若某线程运行完毕，系统将自动删除线程：自动执行 rt_thread_exit() 函数，先将该线程从系统就绪队列中删除，再将该线程的状态更改为关闭状态，不再参与系统调度，然后挂入 rt_thread_defunct 僵尸队列（资源未回收、处于关闭状态的线程队列）中，最后空闲线程会回收被删除线程的资源。

空闲线程也提供了接口来运行用户设置的钩子函数，在空闲线程运行时会调用该钩子函数，适合处理功耗管理、看门狗喂狗等工作。空闲线程必须有得到执行的机会，即其他线程不允许一直while(1)死卡，必须调用具有阻塞性质的函数；否则例如线程删除、回收等操作将无法得到正确执行。

##### 主线程

在系统启动时，系统会创建 main 线程，它的入口函数为 main_thread_entry()，用户的应用入口函数 main() 就是从这里真正开始的，系统调度器启动后，main 线程就开始运行，过程如下图，用户可以在 main() 函数里添加自己的应用程序初始化代码。

![主线程调用过程](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/thread/figures/04main_thread.png)

### 线程的管理方式

​	线程的相关操作，包含：创建 / 初始化线程、启动线程、运行线程、删除 / 脱离线程。

​	可以使用 rt_thread_create() 创建一个动态线程，使用 rt_thread_init() 初始化一个静态线程。

​	动态线程与静态线程的区别是：动态线程是系统自动从动态内存堆上分配栈空间与线程句柄（初始化 heap 之后才能使用 create 创建动态线程），静态线程是由用户分配栈空间与线程句柄。

![线程相关操作](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/thread/figures/04thread_ops.png)

#### 创建和删除线程



一个线程要成为可执行的对象，就必须由操作系统的内核来为它创建一个线程。可以通过如下的接口创建一个动态线程：

```C
rt_thread_t rt_thread_create(const char* name,
                            void (*entry)(void* parameter),
                            void* parameter,
                            rt_uint32_t stack_size,
                            rt_uint8_t priority,
                            rt_uint32_t tick);

```

调用这个函数时，系统会从动态堆内存中分配一个线程句柄，按照参数中指定的栈大小从动态堆内存中分配相应的空间。

分配出来的栈空间是按照 rtconfig.h 中配置的 RT_ALIGN_SIZE 方式对齐。

线程创建 rt_thread_create() 的参数和返回值见下表：

| **参数**   | **描述**                                                     |
| ---------- | ------------------------------------------------------------ |
| name       | 线程的名称；线程名称的最大长度由 rtconfig.h 中的宏 RT_NAME_MAX 指定，多余部分会被自动截掉 |
| entry      | 线程入口函数                                                 |
| parameter  | 线程入口函数参数                                             |
| stack_size | 线程栈大小，单位是字节                                       |
| priority   | 线程的优先级。优先级范围根据系统配置情况（rtconfig.h 中的 RT_THREAD_PRIORITY_MAX 宏定义），如果支持的是 256 级优先级，那么范围是从 0~255，数值越小优先级越高，0 代表最高优先级 |
| tick       | 线程的时间片大小。时间片（tick）的单位是操作系统的时钟节拍。当系统中存在相同优先级线程时，这个参数指定线程一次调度能够运行的最大时间长度。这个时间片运行结束时，调度器自动选择下一个就绪态的同优先级线程进行运行 |
| **返回**   | ——                                                           |
| thread     | 线程创建成功，返回线程句柄                                   |
| RT_NULL    | 线程创建失败                                                 |



一些使用 rt_thread_create() 创建出来的线程，当不需要使用，或者运行出错时，我们可以使用下面的函数接口来从系统中把线程完全删除掉：

```c
rt_err_t rt_thread_delete(rt_thread_t thread);

```

调用该函数后，线程对象将会被移出线程队列并且从内核对象管理器中删除，线程占用的堆栈空间也会被释放，收回的空间将重新用于其他的内存分配。

实际上，用 rt_thread_delete() 函数删除线程接口，仅仅是把相应的线程状态更改为 RT_THREAD_CLOSE 状态，然后放入到 rt_thread_defunct 队列中；

而真正的删除动作（释放线程控制块和释放线程栈）需要到下一次执行空闲线程时，由空闲线程完成最后的线程删除动作。

线程删除 rt_thread_delete() 接口的参数和返回值见下表：

| **参数**  | **描述**         |
| --------- | ---------------- |
| thread    | 要删除的线程句柄 |
| **返回**  | ——               |
| RT_EOK    | 删除线程成功     |
| -RT_ERROR | 删除线程失败     |

<!--注：-->rt_thread_create() 和 rt_thread_delete() 函数仅在使能了系统动态堆时才有效（即 RT_USING_HEAP 宏定义已经定义了）。



#### 初始化和脱离线程

线程的初始化可以使用下面的函数接口完成，来初始化静态线程对象：

```c
rt_err_t rt_thread_init(struct rt_thread* thread,
                        const char* name,
                        void (*entry)(void* parameter), void* parameter,
                        void* stack_start, rt_uint32_t stack_size,
                        rt_uint8_t priority, rt_uint32_t tick);

```

静态线程的线程句柄（线程控制块指针）、线程栈由用户提供。

静态线程是指线程控制块、线程运行栈一般都设置为全局变量，在编译时就被确定、被分配处理，内核不负责动态分配内存空间。需要注意的是，用户提供的栈首地址需做系统对齐（例如 ARM 上需要做 4 字节对齐）。

线程初始化接口 rt_thread_init() 的参数和返回值见下表：

| **参数**    | **描述**                                                     |
| ----------- | ------------------------------------------------------------ |
| thread      | 线程句柄。线程句柄由用户提供出来，并指向对应的线程控制块内存地址 |
| name        | 线程的名称；线程名称的最大长度由 rtconfig.h 中定义的 RT_NAME_MAX 宏指定，多余部分会被自动截掉 |
| entry       | 线程入口函数                                                 |
| parameter   | 线程入口函数参数                                             |
| stack_start | 线程栈起始地址                                               |
| stack_size  | 线程栈大小，单位是字节。在大多数系统中需要做栈空间地址对齐（例如 ARM 体系结构中需要向 4 字节地址对齐） |
| priority    | 线程的优先级。优先级范围根据系统配置情况（rtconfig.h 中的 RT_THREAD_PRIORITY_MAX 宏定义），如果支持的是 256 级优先级，那么范围是从 0 ～ 255，数值越小优先级越高，0 代表最高优先级 |
| tick        | 线程的时间片大小。时间片（tick）的单位是操作系统的时钟节拍。当系统中存在相同优先级线程时，这个参数指定线程一次调度能够运行的最大时间长度。这个时间片运行结束时，调度器自动选择下一个就绪态的同优先级线程进行运行 |
| **返回**    | ——                                                           |
| RT_EOK      | 线程创建成功                                                 |
| -RT_ERROR   | 线程创建失败                                                 |



对于用 rt_thread_init() 初始化的线程，使用 rt_thread_detach() 将使线程对象在线程队列和内核对象管理器中被脱离。

线程脱离函数如下：

```c
rt_err_t rt_thread_detach (rt_thread_t thread);

```

线程脱离接口 rt_thread_detach() 的参数和返回值见下表：

| **参数**  | **描述**                                                   |
| --------- | ---------------------------------------------------------- |
| thread    | 线程句柄，它应该是由 rt_thread_init 进行初始化的线程句柄。 |
| **返回**  | ——                                                         |
| RT_EOK    | 线程脱离成功                                               |
| -RT_ERROR | 线程脱离失败                                               |

这个函数接口是和 rt_thread_delete() 函数相对应的，

rt_thread_delete() 函数操作的对象是 rt_thread_create() 创建的句柄，

而 rt_thread_detach() 函数操作的对象是使用 rt_thread_init() 函数初始化的线程控制块。

同样，线程本身不应调用这个接口脱离线程本身。



#### 启动线程

创建（初始化）的线程状态处于初始状态，并未进入就绪线程的调度队列，我们可以在线程初始化 / 创建成功后调用下面的函数接口让该线程进入就绪态：

```c
rt_err_t rt_thread_startup(rt_thread_t thread);
```

当调用这个函数时，将把线程的状态更改为就绪状态，并放到相应优先级队列中等待调度。如果新启动的线程优先级比当前线程优先级高，将立刻切换到这个线程。线程启动接口 rt_thread_startup() 的参数和返回值见下表：

| **参数**  | **描述**     |
| --------- | ------------ |
| thread    | 线程句柄     |
| **返回**  | ——           |
| RT_EOK    | 线程启动成功 |
| -RT_ERROR | 线程启动失败 |



#### 获得当前线程

在程序的运行过程中，相同的一段代码可能会被多个线程执行，在执行的时候可以通过下面的函数接口获得当前执行的线程句柄：

```c
rt_thread_t rt_thread_self(void);
```

该接口的返回值见下表：

| **返回** | **描述**             |
| -------- | -------------------- |
| thread   | 当前运行的线程句柄   |
| RT_NULL  | 失败，调度器还未启动 |



#### 使线程让出处理器资源

当前线程的时间片用完或者该线程主动要求让出处理器资源时，它将不再占有处理器，调度器会选择相同优先级的下一个线程执行。

线程调用这个接口后，这个线程仍然在就绪队列中。

线程让出处理器使用下面的函数接口：

```c
rt_err_t rt_thread_yield(void);
```

调用该函数后，当前线程首先把自己从它所在的就绪优先级线程队列中删除，

然后把自己挂到这个优先级队列链表的尾部，

然后激活调度器进行线程上下文切换（如果当前优先级只有这一个线程，则这个线程继续执行，不进行上下文切换动作）。



rt_thread_yield() 函数和 rt_schedule() 函数比较相像，但在有相同优先级的其他就绪态线程存在时，系统的行为却完全不一样。

执行 rt_thread_yield() 函数后，当前线程被换出，相同优先级的下一个就绪线程将被执行。

而执行 rt_schedule() 函数后，当前线程并不一定被换出，即使被换出，也不会被放到就绪线程链表的尾部，

而是在系统中选取就绪的优先级最高的线程执行（**如果系统中没有比当前线程优先级更高的线程存在，那么执行完 rt_schedule() 函数后，系统将继续执行当前线程**）。



#### 使线程睡眠

在实际应用中，我们有时需要让运行的当前线程延迟一段时间，在指定的时间到达后重新运行，这就叫做 “线程睡眠”。

线程睡眠可使用以下三个函数接口：

```c
rt_err_t rt_thread_sleep(rt_tick_t tick);
rt_err_t rt_thread_delay(rt_tick_t tick);
rt_err_t rt_thread_mdelay(rt_int32_t ms);
```

这三个函数接口的作用相同，调用它们可以使当前线程挂起一段指定的时间，当这个时间过后，线程会被唤醒并再次进入就绪状态。

这个函数接受一个参数，该参数指定了线程的休眠时间。

线程睡眠接口 rt_thread_sleep/delay/mdelay() 的参数和返回值见下表：

| **参数** | **描述**                                                     |
| -------- | ------------------------------------------------------------ |
| tick/ms  | 线程睡眠的时间： sleep/delay 的传入参数 tick 以 1 个 OS Tick 为单位 ； mdelay 的传入参数 ms 以 1ms 为单位； |
| **返回** | ——                                                           |
| RT_EOK   | 操作成功                                                     |



#### 挂起和恢复线程

当线程调用 rt_thread_delay() 时，线程将主动挂起；

当调用 rt_sem_take()，rt_mb_recv() 等函数时，资源不可使用也将导致线程挂起。

处于挂起状态的线程，如果其等待的资源超时（超过其设定的等待时间），那么该线程将不再等待这些资源，并返回到就绪状态；

或者，当其他线程释放掉该线程所等待的资源时，该线程也会返回到就绪状态。

线程挂起使用下面的函数接口：

```c
rt_err_t rt_thread_suspend (rt_thread_t thread);
```

线程挂起接口 rt_thread_suspend() 的参数和返回值见下表：

| **参数**  | **描述**                                     |
| --------- | -------------------------------------------- |
| thread    | 线程句柄                                     |
| **返回**  | ——                                           |
| RT_EOK    | 线程挂起成功                                 |
| -RT_ERROR | 线程挂起失败，因为该线程的状态并不是就绪状态 |

 注：一个线程尝试挂起另一个线程是一个非常危险的行为，因此RT-Thread对此函数有严格的使用限制：

​	该函数只能使用来挂起当前线程（即自己挂起自己），不可以在线程A中尝试挂起线程B。而且在挂起线程自己后，需要立刻调用 `rt_schedule()` 函数进行手动的线程上下文切换。这是因为A线程在尝试挂起B线程时，A线程并不清楚B线程正在运行什么程序，一旦B线程正在使用例如互斥量、信号量等影响、阻塞其他线程（如C线程）的内核对象，如果此时其他线程也在等待这个内核对象，那么A线程尝试挂起B线程的操作将会引发其他线程（如C线程）的饥饿，严重危及系统的实时性。

恢复线程就是让挂起的线程重新进入就绪状态，并将线程放入系统的就绪队列中；如果被恢复线程在所有就绪态线程中，位于最高优先级链表的第一位，那么系统将进行线程上下文的切换。

线程恢复使用下面的函数接口：

```c
rt_err_t rt_thread_resume (rt_thread_t thread);
```

线程恢复接口 rt_thread_resume() 的参数和返回值见下表：

| **参数**  | **描述**                                                     |
| --------- | ------------------------------------------------------------ |
| thread    | 线程句柄                                                     |
| **返回**  | ——                                                           |
| RT_EOK    | 线程恢复成功                                                 |
| -RT_ERROR | 线程恢复失败，因为该个线程的状态并不是 RT_THREAD_SUSPEND 状态 |

#### 控制线程

当需要对线程进行一些其他控制时，例如动态更改线程的优先级，可以调用如下函数接口：

```c
rt_err_t rt_thread_control(rt_thread_t thread, rt_uint8_t cmd, void* arg);
```

线程控制接口 rt_thread_control() 的参数和返回值见下表：

| **函数参数** | **描述**     |
| ------------ | ------------ |
| thread       | 线程句柄     |
| cmd          | 指示控制命令 |
| arg          | 控制参数     |
| **返回**     | ——           |
| RT_EOK       | 控制执行正确 |
| -RT_ERROR    | 失败         |

指示控制命令 cmd 当前支持的命令包括：

- RT_THREAD_CTRL_CHANGE_PRIORITY：动态更改线程的优先级；

- RT_THREAD_CTRL_STARTUP：开始运行一个线程，等同于 rt_thread_startup() 函数调用；

- RT_THREAD_CTRL_CLOSE：关闭一个线程，等同于 rt_thread_delete() 或 rt_thread_detach() 函数调用。

  

#### 设置和删除空闲钩子

空闲钩子函数是空闲线程的钩子函数，如果设置了空闲钩子函数，就可以**在系统执行空闲线程时，自动执行空闲钩子函数来做一些其他事情**，比如系统指示灯。

设置 / 删除空闲钩子的接口如下：

```c
rt_err_t rt_thread_idle_sethook(void (*hook)(void));
rt_err_t rt_thread_idle_delhook(void (*hook)(void));
```

设置空闲钩子函数 rt_thread_idle_sethook() 的输入参数和返回值如下表所示：

| **函数参数** | **描述**       |
| ------------ | -------------- |
| hook         | 设置的钩子函数 |
| **返回**     | ——             |
| RT_EOK       | 设置成功       |
| -RT_EFULL    | 设置失败       |

删除空闲钩子函数 rt_thread_idle_delhook() 的输入参数和返回值如下表所示：

| **函数参数** | **描述**       |
| ------------ | -------------- |
| hook         | 删除的钩子函数 |
| **返回**     | ——             |
| RT_EOK       | 删除成功       |
| -RT_ENOSYS   | 删除失败       |

注：

空闲线程是一个线程状态永远为就绪态的线程，因此设置的钩子函数必须保证空闲线程在**任何时刻都不会处于挂起状态**，

例如 rt_thread_delay()，rt_sem_take() 等可能会**导致线程挂起的函数都不能使用**。并且，由于 malloc、free 等内存相关的函数内部使用了信号量作为临界区保护，因此**在钩子函数内部也不允许调用此类函数**！



#### 设置调度器钩子

在整个系统的运行时，系统都处于线程运行、中断触发 - 响应中断、切换到其他线程，甚至是线程间的切换过程中，或者说系统的上下文切换是系统中最普遍的事件。

有时用户可能会想知道在一个时刻发生了什么样的线程切换，可以通过调用下面的函数接口设置一个相应的钩子函数。

在系统线程切换时，这个钩子函数将被调用：

```c
void rt_scheduler_sethook(void (*hook)(struct rt_thread* from, struct rt_thread* to));
```

设置调度器钩子函数的输入参数如下表所示：

| **函数参数** | **描述**                   |
| ------------ | -------------------------- |
| hook         | 表示用户定义的钩子函数指针 |

钩子函数 hook() 的声明如下：

```c
void hook(struct rt_thread* from, struct rt_thread* to);
```

调度器钩子函数 hook() 的输入参数如下表所示：

| **函数参数** | **描述**                               |
| ------------ | -------------------------------------- |
| from         | 表示系统所要切换**出**的线程控制块指针 |
| to           | 表示系统所要切换**到**的线程控制块指针 |

注：请仔细编写你的钩子函数，稍有不慎将很可能导致整个系统运行不正常（在这个钩子函数中，基本上不允许调用系统 API，更不应该导致当前运行的上下文挂起）。



# **时钟管理**

**时间是非常重要的概念**，和朋友出去游玩需要约定时间，完成任务也需要花费时间，生活离不开时间。

操作系统也一样，需要通过时间来规范其任务的执行，操作系统中最小的时间单位是时钟节拍 (OS Tick)。

这个主要介绍时钟节拍和基于时钟节拍的定时器，了解时钟节拍如何产生，并学会如何使用 RT-Thread 的定时器。



### 时钟节拍

任何操作系统都需要提供一个时钟节拍，以供系统处理所有和时间有关的事件，如线程的延时、线程的时间片轮转调度以及定时器超时等。

时钟节拍是特定的周期性中断，这个中断可以看做是系统心跳，中断之间的时间间隔取决于不同的应用，一般是 1ms–100ms，时钟节拍率越快，系统的实时响应越快，但是系统的额外开销就越大，从系统启动开始计数的时钟节拍数称为系统时间。

RT-Thread 中，时钟节拍的长度可以根据 RT_TICK_PER_SECOND 的定义来调整，等于 1/RT_TICK_PER_SECOND 秒。

#### 时钟节拍的实现方式

时钟节拍由配置为中断触发模式的硬件定时器产生，

当中断到来时，将调用一次：void rt_tick_increase(void)，通知操作系统已经过去一个系统时钟；

不同硬件定时器中断实现都不同，下面的中断函数以 STM32 定时器作为示例。

```c
void SysTick_Handler(void)
{
    /* 进入中断 */
    rt_interrupt_enter();
    ……
    rt_tick_increase();
    /* 退出中断 */
    rt_interrupt_leave();
}
```

在中断函数中调用 rt_tick_increase() 对全局变量 rt_tick 进行自加，代码如下所示：

```c
void rt_tick_increase(void)
{
    struct rt_thread *thread;

    /* 全局变量 rt_tick 自加 */
    ++ rt_tick;

    /* 检查时间片 */
    thread = rt_thread_self();

    -- thread->remaining_tick;
    if (thread->remaining_tick == 0)
    {
        /* 重新赋初值 */
        thread->remaining_tick = thread->init_tick;

        /* 线程挂起 */
        rt_thread_yield();
    }

    /* 检查定时器 */
    rt_timer_check();
}
```

可以看到全局变量 rt_tick 在每经过一个时钟节拍时，值就会加 1，rt_tick 的值表示了系统从启动开始总共经过的时钟节拍数，即系统时间。

此外，每经过一个时钟节拍时，都会检查当前线程的时间片是否用完，以及是否有定时器超时。

注：中断中的 rt_timer_check() 用于检查系统硬件定时器链表，如果有定时器超时，将调用相应的超时函数。且所有定时器在定时超时后都会从定时器链表中被移除，而周期性定时器会在它再次启动时被加入定时器链表。

#### 获取时钟节拍

由于全局变量 rt_tick 在每经过一个时钟节拍时，值就会加 1，通过调用 rt_tick_get 会返回当前 rt_tick 的值，即可以获取到当前的时钟节拍值。

此接口可用于记录系统的运行时间长短，或者测量某任务运行的时间。

接口函数如下：

```c
rt_tick_t rt_tick_get(void);
```

下表描述了 rt_tick_get() 函数的返回值：

| **返回** | **描述**       |
| -------- | -------------- |
| rt_tick  | 当前时钟节拍值 |



### 定时器管理

定时器，是指从指定的时刻开始，经过一定的指定时间后触发一个事件，例如定个时间提醒第二天能够按时起床。

定时器有硬件定时器和软件定时器之分：

1）**硬件定时器**是芯片本身提供的定时功能。一般是由外部晶振提供给芯片输入时钟，芯片向软件模块提供一组配置寄存器，接受控制输入，到达设定时间值后芯片中断控制器产生时钟中断。硬件定时器的精度一般很高，可以达到纳秒级别，并且是中断触发方式。

2）**软件定时器**是由操作系统提供的一类系统接口，它构建在硬件定时器基础之上，使系统能够提供不受数目限制的定时器服务。

RT-Thread 操作系统提供软件实现的定时器，以时钟节拍（OS Tick）的时间长度为单位，即定时数值必须是 OS Tick 的整数倍，例如一个 OS Tick 是 10ms，那么上层软件定时器只能是 10ms，20ms，100ms 等，而不能定时为 15ms。RT-Thread 的定时器也基于系统的节拍，提供了基于节拍整数倍的定时能力。



### RT-Thread 定时器介绍

RT-Thread 的定时器提供两类定时器机制：

第一类是单次触发定时器，这类定时器在启动后只会触发一次定时器事件，然后定时器自动停止。

第二类是周期触发定时器，这类定时器会周期性的触发定时器事件，直到用户手动的停止，否则将永远持续执行下去。

另外，根据超时函数执行时所处的上下文环境，RT-Thread 的定时器可以分为 HARD_TIMER 模式与 SOFT_TIMER 模式，如下图。

![定时器上下文环境](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/timer/figures/05timer_env.png)

#### HARD_TIMER 模式

HARD_TIMER 模式的定时器超时函数在中断上下文环境中执行，可以在初始化 / 创建定时器时使用参数 RT_TIMER_FLAG_HARD_TIMER 来指定。

在中断上下文环境中执行时，对于超时函数的要求与中断服务例程的要求相同：执行时间应该尽量短，执行时不应导致当前上下文挂起、等待。例如在中断上下文中执行的超时函数它不应该试图去申请动态内存、释放动态内存等。

RT-Thread 定时器默认的方式是 HARD_TIMER 模式，即定时器超时后，超时函数是在系统时钟中断的上下文环境中运行的。

在中断上下文中的执行方式决定了定时器的超时函数不应该调用任何会让当前上下文挂起的系统函数；也不能够执行非常长的时间，否则会导致其他中断的响应时间加长或抢占了其他线程执行的时间。

#### SOFT_TIMER 模式

SOFT_TIMER 模式可配置，通过宏定义 RT_USING_TIMER_SOFT 来决定是否启用该模式。

该模式被启用后，系统会在初始化时创建一个 timer 线程，然后 SOFT_TIMER 模式的定时器超时函数在都会在 timer 线程的上下文环境中执行。

可以在初始化 / 创建定时器时使用参数 RT_TIMER_FLAG_SOFT_TIMER 来指定设置 SOFT_TIMER 模式。

### 定时器工作机制

下面以一个例子来说明 RT-Thread 定时器的工作机制。在 RT-Thread 定时器模块中维护着两个重要的全局变量：

（1）当前系统经过的 tick 时间 rt_tick（当硬件定时器中断来临时，它将加 1）；

（2）定时器链表 rt_timer_list。系统新创建并激活的定时器都会按照以超时时间排序的方式插入到 rt_timer_list 链表中。

如下图所示，系统当前 tick 值为 20，在当前系统中已经创建并启动了三个定时器，分别是定时时间为 50 个 tick 的 Timer1、100 个 tick 的 Timer2 和 500 个 tick 的 Timer3，这三个定时器分别加上系统当前时间 rt_tick=20，从小到大排序链接在 rt_timer_list 链表中，形成如图所示的定时器链表结构。

![定时器链表示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/timer/figures/05timer_linked_list.png)

而 rt_tick 随着硬件定时器的触发一直在增长（每一次硬件定时器中断来临，rt_tick 变量会加 1），50 个 tick 以后，rt_tick 从 20 增长到 70，与 Timer1 的 timeout 值相等，这时会触发与 Timer1 定时器相关联的超时函数，

同时将 Timer1 从 rt_timer_list 链表上删除。同理，100 个 tick 和 500 个 tick 过去后，与 Timer2 和 Timer3 定时器相关联的超时函数会被触发，接着将 Timer2 和 Timer3 定时器从 rt_timer_list 链表中删除。

如果系统当前定时器状态在 10 个 tick 以后（rt_tick=30）有一个任务新创建了一个 tick 值为 300 的 Timer4 定时器，由于 Timer4 定时器的 timeout=rt_tick+300=330, 因此它将被插入到 Timer2 和 Timer3 定时器中间，形成如下图所示链表结构：

![定时器链表插入示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/timer/figures/05timer_linked_list2.png)

#### 定时器控制块

在 RT-Thread 操作系统中，定时器控制块由结构体 struct rt_timer 定义并形成定时器内核对象，再链接到内核对象容器中进行管理。

它是操作系统用于管理定时器的一个数据结构，会存储定时器的一些信息，例如初始节拍数，超时时的节拍数，也包含定时器与定时器之间连接用的链表结构，超时回调函数等。

```c
struct rt_timer
{
    struct rt_object parent;
    rt_list_t row[RT_TIMER_SKIP_LIST_LEVEL];  /* 定时器链表节点 */

    void (*timeout_func)(void *parameter);    /* 定时器超时调用的函数 */
    void      *parameter;                         /* 超时函数的参数 */
    rt_tick_t init_tick;                         /* 定时器初始超时节拍数 */
    rt_tick_t timeout_tick;                     /* 定时器实际超时时的节拍数 */
};
typedef struct rt_timer *rt_timer_t;
```

定时器控制块由 struct rt_timer 结构体定义并形成定时器内核对象，再链接到内核对象容器中进行管理，list 成员则用于把一个激活的（已经启动的）定时器链接到 rt_timer_list 链表中。

#### 定时器跳表 (Skip List) 算法

在前面介绍定时器的工作方式的时候说过，系统新创建并激活的定时器都会按照以超时时间排序的方式插入到 rt_timer_list 链表中，

也就是说 t_timer_list 链表是一个有序链表，RT-Thread 中使用了跳表算法来加快搜索链表元素的速度。

跳表是一种基于并联链表的数据结构，实现简单，插入、删除、查找的时间复杂度均为 O(log n)。

跳表是链表的一种，但它在链表的基础上增加了 “跳跃” 功能，正是这个功能，使得在查找元素时，跳表能够提供 O(log n)的时间复杂度，举例如下：

一个有序的链表，如下图所示，从该有序链表中搜索元素 {13, 39}，需要比较的次数分别为 {3, 5}，总共比较的次数为 3 + 5 = 8 次。

![有序链表示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/timer/figures/05timer_skip_list.png)

使用跳表算法后可以采用类似二叉搜索树的方法，把一些节点提取出来作为索引，得到如下图所示的结构：

![有序链表索引示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/timer/figures/05timer_skip_list2.png)

在这个结构里把 {3, 18,77} 提取出来作为一级索引，这样搜索的时候就可以减少比较次数了, 例如在搜索 39 时仅比较了 3 次（通过比较 3，18，39)。当然我们还可以再从一级索引提取一些元素出来，作为二级索引，这样更能加快元素搜索。

![三层跳表示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/timer/figures/05timer_skip_list3.png)

所以，定时器跳表可以通过上层的索引，在搜索的时候就减少比较次数，提升查找的效率，这是一种通过 “空间来换取时间” 的算法，

在 RT-Thread 中通过宏定义 RT_TIMER_SKIP_LIST_LEVEL 来配置跳表的层数，默认为 1，表示采用一级有序链表图的有序链表算法，每增加一，表示在原链表基础上增加一级索引。

### 定时器的管理方式

前面介绍了 RT-Thread 定时器并对定时器的工作机制进行了概念上的讲解，本节将深入到定时器的各个接口，帮助读者在代码层次上理解 RT-Thread 定时器。

在系统启动时需要初始化定时器管理系统。可以通过下面的函数接口完成：

```c
void rt_system_timer_init(void);
```

如果需要使用 **SOFT_TIMER**，则系统初始化时，应该调用下面这个函数接口：

```c
void rt_system_timer_thread_init(void);
```

定时器控制块中含有定时器相关的重要参数，在定时器各种状态间起到纽带的作用。

定时器的相关操作如下图所示，对定时器的操作包含：创建 / 初始化定时器、启动定时器、运行定时器、删除 / 脱离定时 器，

所有定时器在定时超时后都会从定时器链表中被移除，而周期性定时器会在它再次启动时被加入定时器链表，这与定时器参数设置相关。在每次的操作系统时钟中断发生时，都会对已经超时的定时器状态参数做改变。

![定时器相关操作](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/timer/figures/05timer_ops.png)

#### 创建和删除定时器

当动态创建一个定时器时，可使用下面的函数接口：

```c
rt_timer_t rt_timer_create(const char* name,
                           void (*timeout)(void* parameter),
                           void* parameter,
                           rt_tick_t time,
                           rt_uint8_t flag);
```

调用该函数接口后，内核首先从动态内存堆中分配一个定时器控制块，然后对该控制块进行基本的初始化。其中的各参数和返回值说明详见下表：

rt_timer_create() 的输入参数和返回值

| **参数**                        | **描述**                                                     |
| ------------------------------- | ------------------------------------------------------------ |
| name                            | 定时器的名称                                                 |
| void (timeout) (void parameter) | 定时器超时函数指针（当定时器超时时，系统会调用这个函数）     |
| parameter                       | 定时器超时函数的入口参数（当定时器超时时，调用超时回调函数会把这个参数做为入口参数传递给超时函数） |
| time                            | 定时器的超时时间，单位是时钟节拍                             |
| flag                            | 定时器创建时的参数，支持的值包括单次定时、周期定时、硬件定时器、软件定时器等（可以用 “或” 关系取多个值） |
| **返回**                        | ——                                                           |
| RT_NULL                         | 创建失败（通常会由于系统内存不够用而返回 RT_NULL）           |
| 定时器的句柄                    | 定时器创建成功                                               |

include/rtdef.h 中定义了一些定时器相关的宏，如下：

```c
#define RT_TIMER_FLAG_ONE_SHOT      0x0     /* 单次定时     */
#define RT_TIMER_FLAG_PERIODIC      0x2     /* 周期定时     */

#define RT_TIMER_FLAG_HARD_TIMER    0x0     /* 硬件定时器   */
#define RT_TIMER_FLAG_SOFT_TIMER    0x4     /* 软件定时器   */
```

上面 2 组值可以以 “或” 逻辑的方式赋给 flag。

当指定的 flag 为 RT_TIMER_FLAG_HARD_TIMER 时，如果定时器超时，定时器的回调函数将在**时钟中断的服务例程**上下文中被调用；

当指定的 flag 为 RT_TIMER_FLAG_SOFT_TIMER 时，如果定时器超时，定时器的回调函数将在**系统时钟 timer 线程的**上下文中被调用。



系统不再使用动态定时器时，可使用下面的函数接口（删除定时器）：

```c
rt_err_t rt_timer_delete(rt_timer_t timer);
```

调用这个函数接口后，系统会把这个定时器从 rt_timer_list 链表中删除，然后释放相应的定时器控制块占有的内存，

其中的各参数和返回值说明详见下表：

rt_timer_delete() 的输入参数和返回值

| **参数** | **描述**                                                     |
| -------- | ------------------------------------------------------------ |
| timer    | 定时器句柄，**指向要删除的定时器**                           |
| **返回** | ——                                                           |
| RT_EOK   | 删除成功（如果参数 timer 句柄是一个 RT_NULL，将会导致一个 ASSERT 断言） |

#### 初始化和脱离定时器

当选择静态创建定时器时，可利用 rt_timer_init 接口来初始化该定时器，函数接口如下：

```c
void rt_timer_init(rt_timer_t timer,
                   const char* name,
                   void (*timeout)(void* parameter),
                   void* parameter,
                   rt_tick_t time, rt_uint8_t flag);
```

使用该函数接口时会初始化相应的定时器控制块，初始化相应的定时器名称，定时器超时函数等等，

其中的各参数和返回值说明详见下表：

rt_timer_init() 的输入参数和返回值

| **参数**                        | **描述**                                                     |
| ------------------------------- | ------------------------------------------------------------ |
| timer                           | 定时器句柄，指向要初始化的定时器控制块                       |
| name                            | 定时器的名称                                                 |
| void (timeout) (void parameter) | 定时器超时函数指针（当定时器超时时，系统会调用这个函数）     |
| parameter                       | 定时器超时函数的入口参数（当定时器超时时，调用超时回调函数会把这个参数做为入口参数传递给超时函数） |
| time                            | 定时器的超时时间，单位是时钟节拍                             |
| flag                            | 定时器创建时的参数，支持的值包括单次定时、周期定时、硬件定时器、软件定时器（可以用 “或” 关系取多个值），详见创建定时器小节 |



当一个静态定时器不需要再使用时，可以使用下面的函数接口：

```c
rt_err_t rt_timer_detach(rt_timer_t timer);
```

脱离定时器时，系统会把定时器对象从内核对象容器中脱离，但是定时器对象所占有的内存不会被释放，

其中的各参数和返回值说明详见表下表：

rt_timer_detach() 的输入参数和返回值

| **参数** | **描述**                             |
| -------- | ------------------------------------ |
| timer    | 定时器句柄，指向要脱离的定时器控制块 |
| **返回** | ——                                   |
| RT_EOK   | 脱离成功                             |

#### 启动和停止定时器

当定时器被创建或者初始化以后，并不会被立即启动，必须在调用启动定时器函数接口后，才开始工作，

启动定时器函数接口如下：

```c
rt_err_t rt_timer_start(rt_timer_t timer);
```

调用定时器启动函数接口后，定时器的状态将更改为激活状态（**RT_TIMER_FLAG_ACTIVATED**），

并按照超时顺序插入到 rt_timer_list 队列链表中，其中的各参数和返回值说明详见下表：

rt_timer_start() 的输入参数和返回值

| **参数** | **描述**                             |
| -------- | ------------------------------------ |
| timer    | 定时器句柄，指向要启动的定时器控制块 |
| **返回** | ——                                   |
| RT_EOK   | 启动成功                             |

启动定时器的例子请参考后面的示例代码。



启动定时器以后，若想使它停止，可以使用下面的函数接口：

```c
rt_err_t rt_timer_stop(rt_timer_t timer);
```

调用定时器停止函数接口后，定时器状态将更改为停止状态，并从 rt_timer_list 链表中脱离出来不参与定时器超时检查。

当一个（周期性）定时器超时时，也可以调用这个函数接口停止这个（周期性）定时器本身，其中的各参数和返回值说明详见下表：

rt_timer_stop() 的输入参数和返回值

| **参数**   | **描述**                             |
| ---------- | ------------------------------------ |
| timer      | 定时器句柄，指向要停止的定时器控制块 |
| **返回**   | ——                                   |
| RT_EOK     | 成功停止定时器                       |
| - RT_ERROR | timer 已经处于停止状态               |

#### 控制定时器

除了上述提供的一些编程接口，RT-Thread 也额外提供了定时器控制函数接口，以获取或设置更多定时器的信息。

控制定时器函数接口如下：

```c
rt_err_t rt_timer_control(rt_timer_t timer, rt_uint8_t cmd, void* arg);
```

控制定时器函数接口可根据命令类型参数，来查看或改变定时器的设置，其中的各参数和返回值说明详见下表：

rt_timer_control() 的输入参数和返回值

| **参数** | **描述**                                                     |
| -------- | ------------------------------------------------------------ |
| timer    | 定时器句柄，指向要控制的定时器控制块                         |
| cmd      | 用于控制定时器的命令，当前支持四个命令，分别是设置定时时间，查看定时时间，设置单次触发，设置周期触发 |
| arg      | 与 cmd 相对应的控制命令参数 比如，cmd 为设定超时时间时，就可以将超时时间参数通过 arg 进行设定 |
| **返回** | ——                                                           |
| RT_EOK   | 成功                                                         |

函数参数 cmd 支持的命令：

```c
#define RT_TIMER_CTRL_SET_TIME      0x0     /* 设置定时器超时时间       */
#define RT_TIMER_CTRL_GET_TIME      0x1     /* 获得定时器超时时间       */
#define RT_TIMER_CTRL_SET_ONESHOT   0x2     /* 设置定时器为单次定时器   */
#define RT_TIMER_CTRL_SET_PERIODIC  0x3     /* 设置定时器为周期型定时器 */
```

使用定时器控制接口的代码请见动态定时器例程。



# 线程间同步

在多线程实时系统中，一项的完成往往可以通过多个线程协调的方式共同来完成，

例如，一项工作中的两个线程：

一个线程从传感器中接收数据并且将数据写到共享内存中，同时另一个线程周期性的从共享内存中读取数据并发送去显示，下图描述了两个线程间的数据传递：

![线程间数据传递示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc1/figures/06inter_ths_commu1.png)

如果对共享内存的访问不是排他性的，那么各个线程间可能同时访问它，这将引起数据一致性的问题。例如，在显示线程试图显示数据之前，接收线程还未完成数据的写入，那么显示将包含不同时间采样的数据，造成显示数据的错乱。

将传感器数据写入到共享内存块的接收线程 #1 和将传感器数据从共享内存块中读出的线程 #2 都会访问同一块内存。为了防止出现数据的差错，两个线程访问的动作必须是**互斥进行**的，应该是在一个线程对共享内存块操作完成后，才允许另一个线程去操作，这样，接收线程 #1 与显示线程 #2 才能正常配合，使此项工作正确地执行。

**同步**是指按预定的先后次序进行运行，线程同步是指多个线程通过特定的机制（如互斥量，事件对象，临界区）来控制线程之间的执行顺序，也可以说是在线程之间通过同步建立起执行顺序的关系，如果没有同步，那线程之间将是无序的。

多个线程操作 / 访问同一块区域（代码），这块代码就称为**临界区**，上述例子中的共享内存块就是临界区。

**线程互斥**是指对于临界区资源访问的排它性。当多个线程都要使用临界区资源时，任何时刻最多只允许一个线程去使用，其它要使用该资源的线程必须等待，直到占用资源者释放该资源。线程互斥可以看成是一种特殊的线程同步。

线程的同步方式有很多种，其核心思想都是：**在访问临界区的时候只允许一个 (或一类) 线程运行。**

进入 / 退出临界区的方式有很多种：

1）调用 rt_hw_interrupt_disable() 进入临界区，调用 rt_hw_interrupt_enable() 退出临界区；详见《中断管理》的全局中断开关内容。

2）调用 rt_enter_critical() 进入临界区，调用 rt_exit_critical() 退出临界区。

本章将介绍多种同步方式：**信号量**（semaphore）、**互斥量**（mutex）、和**事件集**（event）。





## 信号量

以生活中的停车场为例来理解信号量的概念：

①当停车场空的时候，停车场的管理员发现有很多空车位，此时会让外面的车陆续进入停车场获得停车位；

②当停车场的车位满的时候，管理员发现已经没有空车位，将禁止外面的车进入停车场，车辆在外排队等候；

③当停车场内有车离开时，管理员发现有空的车位让出，允许外面的车进入停车场；待空车位填满后，又禁止外部车辆进入。

在此例子中，管理员就相当于信号量，管理员手中空车位的个数就是信号量的值（非负数，动态变化）；停车位相当于公共资源（临界区），车辆相当于线程。

车辆通过获得管理员的允许取得停车位，就类似于线程通过获得信号量访问公共资源。

#### 信号量工作机制

信号量是一种轻型的用于解决线程间同步问题的内核对象，线程可以获取或释放它，从而达到同步或互斥的目的。

信号量工作示意图如下图所示，每个信号量对象都有一个信号量值和一个线程等待队列，信号量的值对应了信号量对象的实例数目、资源数目，

假如信号量值为 5，则表示共有 5 个信号量实例（资源）可以被使用，当信号量实例数目为零时，再申请该信号量的线程就会被挂起在该信号量的等待队列上，等待可用的信号量实例（资源）。

![信号量工作示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc1/figures/06sem_work.png)

### 信号量控制块

在 RT-Thread 中，信号量控制块是操作系统用于管理信号量的一个数据结构，由结构体 struct rt_semaphore 表示。

另外一种 C 表达方式 rt_sem_t，表示的是信号量的句柄，在 C 语言中的实现是指向信号量控制块的**指针**。

信号量控制块结构的详细定义如下：

```c
struct rt_semaphore
{
   struct rt_ipc_object parent;  /* 继承自 ipc_object 类 */
   rt_uint16_t value;            /* 信号量的值 */
};
/* rt_sem_t 是指向 semaphore 结构体的指针类型 */

typedef struct rt_semaphore* rt_sem_t;
```

rt_semaphore 对象从 **rt_ipc_object 中派生**，由 **IPC 容器**所管理，信号量的最大值是 **65535**。

### 信号量的管理方式

信号量控制块中含有信号量相关的重要参数，在信号量各种状态间起到纽带的作用。信号量相关接口如下图所示，对一个信号量的操作包含：创建 / 初始化信号量、获取信号量、释放信号量、删除 / 脱离信号量。

![信号量相关接口](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc1/figures/06sem_ops.png)

#### 创建和删除信号量

当创建一个信号量时，内核首先创建一个信号量控制块，然后对该控制块进行基本的初始化工作，

创建信号量使用下面的函数接口：

```c
 rt_sem_t rt_sem_create(const char *name,
                        rt_uint32_t value,
                        rt_uint8_t flag);
```

当调用这个函数时，系统将先从对象管理器中分配一个 semaphore 对象，并初始化这个对象，然后初始化**父类 IPC 对象**以及与 semaphore 相关的部分。

在创建信号量指定的参数中，信号量标志参数决定了当信号量不可用时，多个线程等待的排队方式。

当选择 RT_IPC_FLAG_FIFO（先进先出）方式时，那么等待线程队列将按照**先进先出**的方式排队，**先进入的**线程将**先获得**等待的信号量；

当选择 RT_IPC_FLAG_PRIO（优先级等待）方式时，等待线程队列将按照**优先级**进行排队，**优先级高的**等待线程将先获得等待的信号量。



注：RT_IPC_FLAG_FIFO 属于非实时调度方式，除非应用程序非常在意先来后到，并且你清楚地明白所有涉及到该信号量的线程都将会变为非实时线程，方可使用 RT_IPC_FLAG_FIFO，否则建议采用 RT_IPC_FLAG_PRIO，即确保线程的实时性。

下表描述了该函数的输入参数与返回值：

| **参数**           | **描述**                                                     |
| ------------------ | ------------------------------------------------------------ |
| name               | 信号量名称                                                   |
| value              | 信号量初始值                                                 |
| flag               | 信号量标志，它可以取如下数值： RT_IPC_FLAG_FIFO 或 RT_IPC_FLAG_PRIO |
| **返回**           | ——                                                           |
| RT_NULL            | 创建失败                                                     |
| 信号量的控制块指针 | 创建成功                                                     |

系统不再使用信号量时，可通过删除信号量以释放系统资源，适用于动态创建的信号量。

删除信号量使用下面的函数接口：

```c
rt_err_t rt_sem_delete(rt_sem_t sem);
```

调用这个函数时，系统将删除这个信号量。

如果删除该信号量时，有线程正在等待该信号量，那么删除操作会先**唤醒**等待在该信号量上的线程（等待线程的返回值是 - RT_ERROR），然后再释放信号量的内存资源。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**                         |
| -------- | -------------------------------- |
| sem      | rt_sem_create() 创建的信号量对象 |
| **返回** | ——                               |
| RT_EOK   | 删除成功                         |

#### 初始化和脱离信号量

对于静态信号量对象，**它的内存空间在编译时期就被编译器分配出来**，放在读写数据段或未初始化数据段上，此时使用信号量就不再需要使用 rt_sem_create 接口来创建它，而只需在使用前对它进行初始化即可。

初始化信号量对象可使用下面的函数接口：

```c
rt_err_t rt_sem_init(rt_sem_t       sem,
                    const char     *name,
                    rt_uint32_t    value,
                    rt_uint8_t     flag)
```

当调用这个函数时，系统将对这个 semaphore 对象进行初始化，然后初始化 IPC 对象以及与 semaphore 相关的部分。

信号量标志可用上面创建信号量函数里提到的标志。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**                                                     |
| -------- | ------------------------------------------------------------ |
| sem      | 信号量对象的句柄                                             |
| name     | 信号量名称                                                   |
| value    | 信号量初始值                                                 |
| flag     | 信号量标志，它可以取如下数值： RT_IPC_FLAG_FIFO 或 RT_IPC_FLAG_PRIO |
| **返回** | ——                                                           |
| RT_EOK   | 初始化成功                                                   |

脱离信号量就是让信号量对象从内核对象管理器中脱离，适用于静态初始化的信号量。

脱离信号量使用下面的函数接口：

```c
rt_err_t rt_sem_detach(rt_sem_t sem);
```

使用该函数后，内核先唤醒所有挂在该信号量等待队列上的线程，然后将该信号量从内核对象管理器中脱离。原来挂起在信号量上的等待线程将获得 - RT_ERROR 的返回值。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**         |
| -------- | ---------------- |
| sem      | 信号量对象的句柄 |
| **返回** | ——               |
| RT_EOK   | 脱离成功         |

#### 获取信号量

线程通过获取信号量来获得信号量资源实例，当信号量值大于零时，线程将获得信号量，并且相应的信号量值会减 1，获取信号量使用下面的函数接口：

```c
rt_err_t rt_sem_take (rt_sem_t sem, rt_int32_t time);
```

在调用这个函数时，如果信号量的值等于零，那么说明当前信号量资源实例不可用，申请该信号量的线程将根据 time 参数的情况选择**直接返回、或挂起等待一段时间、或永久等待**，直到其他线程或中断释放该信号量。

如果在参数 time 指定的时间内依然得不到信号量，线程将**超时返回**，返回值是 - RT_ETIMEOUT。

下表描述了该函数的输入参数与返回值：

| **参数**     | **描述**                                          |
| ------------ | ------------------------------------------------- |
| sem          | 信号量对象的句柄                                  |
| time         | 指定的等待时间，单位是操作系统时钟节拍（OS Tick） |
| **返回**     | ——                                                |
| RT_EOK       | 成功获得信号量                                    |
| -RT_ETIMEOUT | 超时依然未获得信号量                              |
| -RT_ERROR    | 其他错误                                          |

#### 无等待获取信号量

当用户不想在申请的信号量上挂起线程进行等待时，可以使用**无等待方式**获取信号量，

无等待获取信号量使用下面的函数接口：

```c
rt_err_t rt_sem_trytake(rt_sem_t sem);
```

这个函数与 `rt_sem_take(sem, RT_WAITING_NO)` 的作用相同，即当线程申请的信号量资源实例不可用的时候，它不会等待在该信号量上，而是直接返回 - RT_ETIMEOUT。

下表描述了该函数的输入参数与返回值：

| **参数**     | **描述**         |
| ------------ | ---------------- |
| sem          | 信号量对象的句柄 |
| **返回**     | ——               |
| RT_EOK       | 成功获得信号量   |
| -RT_ETIMEOUT | 获取失败         |

#### 释放信号量

释放信号量可以**唤醒**挂起在该信号量上的线程。

释放信号量使用下面的函数接口：

```c
rt_err_t rt_sem_release(rt_sem_t sem);
```

例如当信号量的值等于零时，并且有线程等待这个信号量时，释放信号量将唤醒等待在该信号量线程队列中的第一个线程，由它获取信号量；否则将把信号量的值加 1。下表描述了该函数的输入参数与返回值：

| **参数** | **描述**         |
| -------- | ---------------- |
| sem      | 信号量对象的句柄 |
| **返回** | ——               |
| RT_EOK   | 成功释放信号量   |

### **生产者消费者例程**

信号量的个应用例程如下所示，例程使用 2 个线程、3 个信号量实现生产者与消费者的例子。

其中：

3 个信号量分别为：

①lock：信号量锁的作用，因为 2 个线程都会对同一个数组 array 进行操作，所以该数组是一个共享资源，锁用来保护这个共享资源。②empty：空位个数，初始化为 5 个空位。

③full：满位个数，初始化为 0 个满位。

2 个线程分别为：

①生产者线程：获取到空位后，产生一个数字，循环放入数组中，然后释放一个满位。

②消费者线程：获取到满位后，读取数组内容并相加，然后释放一个空位。

```c
#include <rtthread.h>

#define THREAD_PRIORITY       6
#define THREAD_STACK_SIZE     512
#define THREAD_TIMESLICE      5

/* 定义最大 5 个元素能够被产生 */
#define MAXSEM 5

/* 用于放置生产的整数数组 */
rt_uint32_t array[MAXSEM];

/* 指向生产者、消费者在 array 数组中的读写位置 */
static rt_uint32_t set, get;

/* 指向线程控制块的指针 */
static rt_thread_t producer_tid = RT_NULL;
static rt_thread_t consumer_tid = RT_NULL;

struct rt_semaphore sem_lock;
struct rt_semaphore sem_empty, sem_full;

/* 生产者线程入口 */
void producer_thread_entry(void *parameter)
{
    int cnt = 0;

    /* 运行 10 次 */
    while (cnt < 10)
    {
        /* 获取一个空位 */
        rt_sem_take(&sem_empty, RT_WAITING_FOREVER);

        /* 修改 array 内容，上锁 */
        rt_sem_take(&sem_lock, RT_WAITING_FOREVER);
        array[set % MAXSEM] = cnt + 1;
        rt_kprintf("the producer generates a number: %d\n", array[set % MAXSEM]);
        set++;
        rt_sem_release(&sem_lock);

        /* 发布一个满位 */
        rt_sem_release(&sem_full);
        cnt++;

        /* 暂停一段时间 */
        rt_thread_mdelay(20);
    }

    rt_kprintf("the producer exit!\n");
    cnt = 0;
}

/* 消费者线程入口 */
void consumer_thread_entry(void *parameter)
{
    rt_uint32_t sum = 0;

    while (1)
    {
        /* 获取一个满位 */
        rt_sem_take(&sem_full, RT_WAITING_FOREVER);

        /* 临界区，上锁进行操作 */
        rt_sem_take(&sem_lock, RT_WAITING_FOREVER);
        sum += array[get % MAXSEM];
        rt_kprintf("the consumer[%d] get a number: %d\n", (get % MAXSEM), array[get % MAXSEM]);
        get++;
        rt_sem_release(&sem_lock);

        /* 释放一个空位 */
        rt_sem_release(&sem_empty);

        /* 生产者生产到 10 个数目，停止，消费者线程相应停止 */
        if (get == 10) break;

        /* 暂停一小会时间 */
        rt_thread_mdelay(50);
    }

    rt_kprintf("the consumer sum is: %d\n", sum);
    rt_kprintf("the consumer exit!\n");
    rt_sem_detach(&sem_lock);
    rt_sem_detach(&sem_empty);
    rt_sem_detach(&sem_full);
    sum = 0;
}

int producer_consumer(void)
{
    set = 0;
    get = 0;

    /* 初始化 3 个信号量 */
    rt_sem_init(&sem_lock, "lock",     1,      RT_IPC_FLAG_PRIO);
    rt_sem_init(&sem_empty, "empty",   MAXSEM, RT_IPC_FLAG_PRIO);
    rt_sem_init(&sem_full, "full",     0,      RT_IPC_FLAG_PRIO);

    /* 创建生产者线程 */
    producer_tid = rt_thread_create("producer",
                                    producer_thread_entry, RT_NULL,
                                    THREAD_STACK_SIZE,
                                    THREAD_PRIORITY - 1,
                                    THREAD_TIMESLICE);
    if (producer_tid != RT_NULL)
    {
        rt_thread_startup(producer_tid);
    }
    else
    {
        rt_kprintf("create thread producer failed");
        return -1;
    }

    /* 创建消费者线程 */
    consumer_tid = rt_thread_create("consumer",
                                    consumer_thread_entry, RT_NULL,
                                    THREAD_STACK_SIZE,
                                    THREAD_PRIORITY + 1,
                                    THREAD_TIMESLICE);
    if (consumer_tid != RT_NULL)
    {
        rt_thread_startup(consumer_tid);
    }
    else
    {
        rt_kprintf("create thread consumer failed");
        return -1;
    }

    return 0;
}

/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(producer_consumer, producer_consumer sample);复制错误复制成功
```



该例程的仿真结果如下：



```shell
 \ | /
- RT -     Thread Operating System
 / | \     4.1.1 build Sep  2 2024 18:24:30
 2006 - 2022 Copyright by RT-Thread team
msh >producer_consumer
the producer generates a number: 1
the consumer[0] get a number: 1
msh >the producer generates a number: 2
the producer generates a number: 3
the consumer[1] get a number: 2
the producer generates a number: 4
the producer generates a number: 5
the consumer[2] get a number: 3
the producer generates a number: 6
the producer generates a number: 7
the producer generates a number: 8
the consumer[3] get a number: 4
the producer generates a number: 9
the consumer[4] get a number: 5
the producer generates a number: 10
the producer exit!
the consumer[0] get a number: 6
the consumer[1] get a number: 7
the consumer[2] get a number: 8
the consumer[3] get a number: 9
the consumer[4] get a number: 10
the consumer sum is: 55
the consumer exit!

msh >producer_consumer
the producer generates a number: 1
the consumer[0] get a number: 1
msh >the producer generates a number: 2
the producer generates a number: 3
the consumer[1] get a number: 2
the producer generates a number: 4
the producer generates a number: 5
the consumer[2] get a number: 3
the producer generates a number: 6
the producer generates a number: 7
the producer generates a number: 8
the consumer[3] get a number: 4
the producer generates a number: 9
the consumer[4] get a number: 5
the producer generates a number: 10
the producer exit!
the consumer[0] get a number: 6
the consumer[1] get a number: 7
the consumer[2] get a number: 8
the consumer[3] get a number: 9
the consumer[4] get a number: 10
the consumer sum is: 55
the consumer exit!复制错误复制成功
```

本例程可以理解为生产者生产产品放入仓库，消费者从仓库中取走产品。

（1）生产者线程：

1）获取 1 个空位（放产品 number），此时空位减 1；

2）上锁保护；本次的产生的 number 值为 cnt+1，把值循环存入数组 array 中；再开锁；

3）释放 1 个满位（给仓库中放置一个产品，仓库就多一个满位），满位加 1；

（2）消费者线程：

1）获取 1 个满位（取产品 number），此时满位减 1；

2）上锁保护；将本次生产者生产的 number 值从 array 中读出来，并与上次的 number 值相加；再开锁；

3）释放 1 个空位（从仓库上取走一个产品，仓库就多一个空位），空位加 1。



生产者依次产生 10 个 number，消费者依次取走，并将 10 个 number 的值求和。信号量锁 lock 保护 array 临界区资源：保证了消费者每次取 number 值的排他性，实现了线程间同步。

### 信号量的使用场合

信号量是一种非常灵活的同步方式，可以运用在多种场合中。

形成锁、同步、资源计数等关系，也能方便的用于线程与线程、中断与线程间的同步中。



#### 线程同步



线程同步是信号量最简单的一类应用。

例如，使用信号量进行两个线程之间的同步，信号量的值初始化成 0，表示具备 0 个信号量资源实例；而尝试获得该信号量的线程，将直接在这个信号量上进行等待。



当持有信号量的线程完成它处理的工作时，释放这个信号量，可以把等待在这个信号量上的线程唤醒，让它执行下一部分工作。

这类场合也可以看成把信号量用于工作完成标志：持有信号量的线程完成它自己的工作，然后通知等待该信号量的线程继续下一部分工作。

#### 中断与线程的同步

信号量也能够方便地应用于中断与线程间的同步，例如一个中断触发，中断服务例程需要通知线程进行相应的数据处理。

这个时候可以设置信号量的初始值是 0，线程在试图持有这个信号量时，由于信号量的初始值是 0，线程直接在这个信号量上挂起直到信号量被释放。当中断触发时，先进行与硬件相关的动作，例如从硬件的 I/O 口中读取相应的数据，并确认中断以清除中断源，而后释放一个信号量来唤醒相应的线程以做后续的数据处理。

例如 FinSH 线程的处理方式，如下图所示。

![FinSH 的中断、线程间同步示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc1/figures/06inter_ths_commu2.png)

信号量的值初始为 0，当 FinSH 线程试图取得信号量时，因为信号量值是 0，所以它会被挂起。当 console 设备有数据输入时，产生中断，从而进入中断服务例程。在中断服务例程中，它会读取 console 设备的数据，并把读得的数据放入 UART buffer 中进行缓冲，而后释放信号量，释放信号量的操作将唤醒 shell 线程。在中断服务例程运行完毕后，如果系统中没有比 shell 线程优先级更高的就绪线程存在时，shell 线程将持有信号量并运行，从 UART buffer 缓冲区中获取输入的数据。

Note

注：中断与线程间的互斥不能采用信号量（锁）的方式，而应采用开关中断的方式。

#### 资源计数

信号量也可以认为是一个递增或递减的计数器，需要注意的是信号量的值非负。例如：初始化一个信号量的值为 5，则这个信号量可最大连续减少 5 次，直到计数器减为 0。

资源计数适合于线程间工作处理速度不匹配的场合，这个时候信号量可以做为前一线程工作完成个数的计数，而当调度到后一线程时，它也可以以一种连续的方式一次处理多个事件。例如，生产者与消费者问题中，生产者可以对信号量进行多次释放，而后消费者被调度到时能够一次处理多个信号量资源。



注：一般资源计数类型多是混合方式的线程间同步，因为对于单个的资源处理依然存在线程的多重访问，这就需要对一个单独的资源进行访问、处理，并进行锁方式的互斥操作。

#### 锁

锁，单一的锁常应用于多个线程间对同一共享资源（即临界区）的访问。

信号量在作为锁来使用时，通常应将信号量资源实例初始化成 1，代表系统默认有一个资源可用，因为信号量的值始终在 1 和 0 之间变动，所以这类锁也叫做二值信号量。

如下图所示，当线程需要访问共享资源时，它需要先获得这个资源锁。当这个线程成功获得资源锁时，其他打算访问共享资源的线程会由于获取不到资源而挂起，这是因为其他线程在试图获取这个锁时，这个锁已经被锁上（信号量值是 0）。当获得信号量的线程处理完毕，退出临界区时，它将会释放信号量并把锁解开，而挂起在锁上的第一个等待线程将被唤醒从而获得临界区的访问权。

![锁](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc1/figures/06sem_lock.png)



注：在计算机操作系统发展历史上，人们早期使用二值信号量来保护临界区，但是在1990年，研究人员发现了使用信号量保护临界区会导致无界优先级反转的问题，因此提出了互斥量的概念。如今，我们已经不使用二值信号量来保护临界区，互斥量取而代之。



## 互斥量

互斥量又叫相互排斥的信号量，是一种特殊的二值信号量。

互斥量类似于只有一个车位的停车场：当有一辆车进入的时候，将停车场大门锁住，其他车辆在外面等候。当里面的车出来时，将停车场大门打开，下一辆车才可以进入。

#### 互斥量工作机制

互斥量和信号量不同的是：拥有互斥量的线程拥有互斥量的所有权，互斥量支持递归访问且能防止线程优先级翻转；并且互斥量只能由持有线程释放，而信号量则可以由任何线程释放。

互斥量的状态只有两种，开锁或闭锁（两种状态值）。

当有线程持有它时，互斥量处于闭锁状态，由这个线程获得它的所有权。相反，当这个线程释放它时，将对互斥量进行开锁，失去它的所有权。

当一个线程持有互斥量时，其他线程将不能够对它进行开锁或持有它，持有该互斥量的线程也能够再次获得这个锁而不被挂起，如下图时所示。这个特性与一般的二值信号量有很大的不同：在信号量中，因为已经不存在实例，线程递归持有会发生主动挂起（最终形成死锁）。

![互斥量工作示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc1/figures/06mutex_work.png)

使用信号量会导致的另一个潜在问题是线程优先级翻转问题。

所谓优先级翻转，即当一个高优先级线程试图通过信号量机制访问共享资源时，如果该信号量已被一低优先级线程持有，而这个低优先级线程在运行过程中可能又被其它一些中等优先级的线程抢占，因此造成高优先级线程被许多具有较低优先级的线程阻塞，实时性难以得到保证。

如下图所示：有优先级为 A、B 和 C 的三个线程，优先级 A> B > C。线程 A，B 处于挂起状态，等待某一事件触发，线程 C 正在运行，此时线程 C 开始使用某一共享资源 M。在使用过程中，线程 A 等待的事件到来，线程 A 转为就绪态，因为它比线程 C 优先级高，所以立即执行。但是当线程 A 要使用共享资源 M 时，由于其正在被线程 C 使用，因此线程 A 被挂起切换到线程 C 运行。如果此时线程 B 等待的事件到来，则线程 B 转为就绪态。由于线程 B 的优先级比线程 C 高，且线程B没有用到共享资源 M ，因此线程 B 开始运行，直到其运行完毕，线程 C 才开始运行。只有当线程 C 释放共享资源 M 后，线程 A 才得以执行。在这种情况下，优先级发生了翻转：线程 B 先于线程 A 运行。这样便不能保证高优先级线程的响应时间。

![优先级反转 (M 为信号量)](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc1/figures/06priority_inversion.png)

在 RT-Thread 操作系统中，互斥量可以解决优先级翻转问题，实现的是优先级继承协议 (Sha, 1990)。

优先级继承是通过在线程 A 尝试获取共享资源而被挂起的期间内，将线程 C 的优先级提升到线程 A 的优先级别，从而解决优先级翻转引起的问题。这样能够防止 C（间接地防止 A）被 B 抢占，如下图所示。优先级继承是指，提高某个占有某种资源的低优先级线程的优先级，使之与所有等待该资源的线程中优先级最高的那个线程的优先级相等，然后执行，而当这个低优先级线程释放该资源时，优先级重新回到初始设定。因此，继承优先级的线程避免了系统资源被任何中间优先级的线程抢占。

![优先级继承 (M 为互斥量)](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc1/figures/06priority_inherit.png)



注：在获得互斥量后，请尽快释放互斥量，并且在持有互斥量的过程中，不得再行更改持有互斥量线程的优先级，否则可能人为引入无界优先级反转的问题。

### 互斥量控制块

在 RT-Thread 中，**互斥量控制块**是操作系统用于管理互斥量的一个数据结构，由**结构体 struct rt_mutex** 表示。

另外一种 C 表达方式 **rt_mutex_t**，表示的是**互斥量的句柄**，在 C 语言中的实现是指互斥量控制块的指针。

互斥量控制块结构的详细定义请见以下代码：

```c
struct rt_mutex
    {
        struct rt_ipc_object parent;                /* 继承自 ipc_object 类 */

        rt_uint16_t          value;                   /* 互斥量的值 */
        rt_uint8_t           original_priority;     /* 持有线程的原始优先级 */
        rt_uint8_t           hold;                     /* 持有线程的持有次数   */
        struct rt_thread    *owner;                 /* 当前拥有互斥量的线程 */
    };
    /* rt_mutext_t 为指向互斥量结构体的指针类型  */
    typedef struct rt_mutex* rt_mutex_t;
```

rt_mutex 对象从 rt_ipc_object 中派生，由 IPC 容器所管理。

### 互斥量的管理方式

互斥量控制块中含有互斥相关的重要参数，在互斥量功能的实现中起到重要的作用。

互斥量相关接口如下图所示，对一个互斥量的操作包含：创建 / 初始化互斥量、获取互斥量、释放互斥量、删除 / 脱离互斥量。

![互斥量相关接口](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc1/figures/06mutex_ops.png)

#### 创建和删除互斥量

创建一个互斥量时，内核首先创建一个互斥量控制块，然后完成对该控制块的初始化工作。

创建互斥量使用下面的函数接口：

```c
rt_mutex_t rt_mutex_create (const char* name, rt_uint8_t flag);
```

可以调用 rt_mutex_create 函数创建一个互斥量，它的名字由 name 所指定。

当调用这个函数时，系统将先从对象管理器中分配一个 mutex 对象，并初始化这个对象，然后初始化父类 IPC 对象以及与 mutex 相关的部分。

互斥量的 flag 标志已经作废，无论用户选择 RT_IPC_FLAG_PRIO 还是 RT_IPC_FLAG_FIFO，内核均按照 RT_IPC_FLAG_PRIO 处理。

下表描述了该函数的输入参数与返回值：

| **参数**   | **描述**                                                     |
| ---------- | ------------------------------------------------------------ |
| name       | 互斥量的名称                                                 |
| flag       | 该标志已经作废，无论用户选择 RT_IPC_FLAG_PRIO 还是 RT_IPC_FLAG_FIFO，内核均按照 RT_IPC_FLAG_PRIO 处理 |
| **返回**   | ——                                                           |
| 互斥量句柄 | 创建成功                                                     |
| RT_NULL    | 创建失败                                                     |

当不再使用互斥量时，通过删除互斥量以释放系统资源，适用于动态创建的互斥量。

删除互斥量使用下面的函数接口：

```c
rt_err_t rt_mutex_delete (rt_mutex_t mutex);
```

当删除一个互斥量时，所有等待此互斥量的线程都将被唤醒，等待线程获得的返回值是 - RT_ERROR。

然后系统将该互斥量从内核对象管理器链表中删除并释放互斥量占用的内存空间。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**         |
| -------- | ---------------- |
| mutex    | 互斥量对象的句柄 |
| **返回** | ——               |
| RT_EOK   | 删除成功         |

#### 初始化和脱离互斥量

静态互斥量对象的内存是在系统编译时由编译器分配的，一般放于读写数据段或未初始化数据段中。

在使用这类静态互斥量对象前，需要先进行初始化。

初始化互斥量使用下面的函数接口：

```c
rt_err_t rt_mutex_init (rt_mutex_t mutex, const char* name, rt_uint8_t flag);
```

使用该函数接口时，需指定互斥量对象的句柄（即指向互斥量控制块的指针），互斥量名称以及互斥量标志。

互斥量标志可用上面创建互斥量函数里提到的标志。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**                                                     |
| -------- | ------------------------------------------------------------ |
| mutex    | 互斥量对象的句柄，它由用户提供，并指向互斥量对象的内存块     |
| name     | 互斥量的名称                                                 |
| flag     | 该标志已经作废，无论用户选择 RT_IPC_FLAG_PRIO 还是 RT_IPC_FLAG_FIFO，内核均按照 RT_IPC_FLAG_PRIO 处理 |
| **返回** | ——                                                           |
| RT_EOK   | 初始化成功                                                   |



脱离互斥量将把互斥量对象从内核对象管理器中脱离，适用于**静态**初始化的互斥量。

脱离互斥量使用下面的函数接口：

```c
rt_err_t rt_mutex_detach (rt_mutex_t mutex);
```

使用该函数接口后，内核先唤醒所有挂在该互斥量上的线程（线程的返回值是 -RT_ERROR），然后系统将该互斥量从内核对象管理器中脱离。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**         |
| -------- | ---------------- |
| mutex    | 互斥量对象的句柄 |
| **返回** | ——               |
| RT_EOK   | 成功             |

#### 获取互斥量

线程获取了互斥量，那么线程就有了对该互斥量的所有权，即某一个时刻一个互斥量只能被一个线程持有。

获取互斥量使用下面的函数接口：

```c
rt_err_t rt_mutex_take (rt_mutex_t mutex, rt_int32_t time);
```

如果互斥量没有被其他线程控制，那么申请该互斥量的线程将成功获得该互斥量。

如果互斥量已经被当前线程线程控制，则该互斥量的**持有计数加 1**，当前线程也不会挂起等待。

如果互斥量已经被其他线程占有，则当前线程在该互斥量上挂起等待，直到其他线程释放它或者等待时间超过指定的超时时间。

下表描述了该函数的输入参数与返回值：

| **参数**     | **描述**         |
| ------------ | ---------------- |
| mutex        | 互斥量对象的句柄 |
| time         | 指定等待的时间   |
| **返回**     | ——               |
| RT_EOK       | 成功获得互斥量   |
| -RT_ETIMEOUT | 超时             |
| -RT_ERROR    | 获取失败         |

#### 无等待获取互斥量

当用户不想在申请的互斥量上挂起线程进行等待时，可以使用无等待方式获取互斥量，无等待获取互斥量使用下面的函数接口：

```c
rt_err_t rt_mutex_trytake(rt_mutex_t mutex);
```

这个函数与 `rt_mutex_take(mutex, RT_WAITING_NO)` 的作用相同，

即当线程申请的互斥量资源实例不可用的时候，它不会等待在该互斥量上，而是直接返回 - RT_ETIMEOUT。

下表描述了该函数的输入参数与返回值：

| **参数**     | **描述**         |
| ------------ | ---------------- |
| mutex        | 互斥量对象的句柄 |
| **返回**     | ——               |
| RT_EOK       | 成功获得互斥量   |
| -RT_ETIMEOUT | 获取失败         |

####  释放互斥量

当线程完成互斥资源的访问后，应尽快释放它占据的互斥量，使得其他线程能及时获取该互斥量。

释放互斥量使用下面的函数接口：

```c
rt_err_t rt_mutex_release(rt_mutex_t mutex);
```

使用该函数接口时，只有已经拥有互斥量控制权的线程才能释放它，每释放一次该互斥量，它的持有计数就减 1。

当该互斥量的持有计数为零时（即持有线程已经释放所有的持有操作），它变为可用，等待在该互斥量上的线程将被唤醒。

如果线程的运行优先级被互斥量提升，那么当互斥量被释放后，线程恢复为持有互斥量前的优先级。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**         |
| -------- | ---------------- |
| mutex    | 互斥量对象的句柄 |
| **返回** | ——               |
| RT_EOK   | 成功             |



### 互斥量应用示例

这是一个互斥量的应用例程，互斥锁是一种保护共享资源的方法。当一个线程拥有互斥锁的时候，可以保护共享资源不被其他线程破坏。

下面用一个例子来说明，有两个线程：

线程 1 和线程 2，线程 1 对 2 个 number 分别进行加 1 操作；

线程 2 也对 2 个 number 分别进行加 1 操作，使用互斥量保证线程改变 2 个 number 值的操作不被打断。

如下代码所示：

**互斥量例程**

```c
#include <rtthread.h>

#define THREAD_PRIORITY 8
#define THREAD_TIMESLICE 5

/* 指向互斥量的指针 */
static rt_mutex_t dynamic_mutex = RT_NULL;
static rt_uint8_t number1, number2 = 0;
/* 线程退出标志 */
static rt_bool_t thread_exit_flag = RT_FALSE;

ALIGN(RT_ALIGN_SIZE)
static char thread1_stack[1024];
static struct rt_thread thread1;

static void rt_thread_entry1(void *parameter)
{
    while (1)
    {
        /* 线程 1 在获取互斥量前检查它是否存在 */
        if (dynamic_mutex == RT_NULL || thread_exit_flag)
        {
            number1 = 0;
            number2 = 0;

            /* 重置退出标志 */
            thread_exit_flag = RT_FALSE;
            break; /* 退出线程 */
        }

        /* 获取互斥量并进行操作 */
        if (rt_mutex_take(dynamic_mutex, RT_WAITING_FOREVER) == RT_EOK)
        {
            number1++;
            number2++;
            rt_kprintf("thread1 mutex protect, number1 = number2 is %d\n", number1);
            rt_mutex_release(dynamic_mutex);
            rt_thread_mdelay(10);
        }
    }
}

ALIGN(RT_ALIGN_SIZE)
static char thread2_stack[1024];
static struct rt_thread thread2;
static void rt_thread_entry2(void *parameter)
{
    while (1)
    {
        /* 获取互斥量 */
        if (rt_mutex_take(dynamic_mutex, RT_WAITING_FOREVER) == RT_EOK)
        {
            if (number1 != number2)
            {
                rt_kprintf("not protect. number1 = %d, number2 = %d\n", number1, number2);
            }
            else
            {
                rt_kprintf("mutex protect, number1 = number2 is %d\n", number1);
            }

            number1++;
            number2++;
            rt_mutex_release(dynamic_mutex);

            /* 判断是否达到退出条件 */
            if (number1 >= 50)
            {
                thread_exit_flag = RT_TRUE;

                /* 删除互斥量 */
                rt_mutex_delete(dynamic_mutex);
                dynamic_mutex = RT_NULL;

                break; /* 退出线程 */
            }
        }
    }
}

/* 互斥量示例的初始化 */
int mutex_sample(void)
{
    /* 创建一个动态互斥量 */
    dynamic_mutex = rt_mutex_create("dmutex", RT_IPC_FLAG_PRIO);
    if (dynamic_mutex == RT_NULL)
    {
        rt_kprintf("create dynamic mutex failed.\n");
        return -1;
    }

    rt_thread_init(&thread1,
                   "thread1",
                   rt_thread_entry1,
                   RT_NULL,
                   &thread1_stack[0],
                   sizeof(thread1_stack),
                   THREAD_PRIORITY, THREAD_TIMESLICE);
    rt_thread_startup(&thread1);

    rt_thread_init(&thread2,
                   "thread2",
                   rt_thread_entry2,
                   RT_NULL,
                   &thread2_stack[0],
                   sizeof(thread2_stack),
                   THREAD_PRIORITY - 1, THREAD_TIMESLICE);
    rt_thread_startup(&thread2);

    return 0;
}

/* 导出到 MSH 命令列表中 */
MSH_CMD_EXPORT(mutex_sample, mutex sample);复制错误复制成功
```

线程 1 与线程 2 中均使用互斥量保护对 2 个 number 的操作（倘若将线程 1 中的获取、释放互斥量语句注释掉，线程 1 将对 number 不再做保护）

仿真运行结果如下：

```shell
\ | /
- RT -     Thread Operating System
 / | \     4.1.1 build Sep  2 2024 19:21:00
 2006 - 2022 Copyright by RT-Thread team
msh >mutex_sample
msh >mutex protect, number1 = number2 is 1
mutex protect, number1 = number2 is 2
mutex protect, number1 = number2 is 3
mutex protect, number1 = number2 is 4
mutex protect, number1 = number2 is 5
mutex protect, number1 = number2 is 6
mutex protect, number1 = number2 is 7
mutex protect, number1 = number2 is 8
mutex protect, number1 = number2 is 9
mutex protect, number1 = number2 is 10
mutex protect, number1 = number2 is 11
mutex protect, number1 = number2 is 12
mutex protect, number1 = number2 is 13
mutex protect, number1 = number2 is 14
mutex protect, number1 = number2 is 15
mutex protect, number1 = number2 is 16
mutex protect, number1 = number2 is 17
mutex protect, number1 = number2 is 18
mutex protect, number1 = number2 is 19
mutex protect, number1 = number2 is 20
mutex protect, number1 = number2 is 21
mutex protect, number1 = number2 is 22
mutex protect, number1 = number2 is 23
mutex protect, number1 = number2 is 24
mutex protect, number1 = number2 is 25
mutex protect, number1 = number2 is 26
mutex protect, number1 = number2 is 27
mutex protect, number1 = number2 is 28
mutex protect, number1 = number2 is 29
mutex protect, number1 = number2 is 30
mutex protect, number1 = number2 is 31
mutex protect, number1 = number2 is 32
mutex protect, number1 = number2 is 33
mutex protect, number1 = number2 is 34
mutex protect, number1 = number2 is 35
mutex protect, number1 = number2 is 36
mutex protect, number1 = number2 is 37
mutex protect, number1 = number2 is 38
mutex protect, number1 = number2 is 39
mutex protect, number1 = number2 is 40
mutex protect, number1 = number2 is 41
mutex protect, number1 = number2 is 42
mutex protect, number1 = number2 is 43
mutex protect, number1 = number2 is 44
mutex protect, number1 = number2 is 45
mutex protect, number1 = number2 is 46
mutex protect, number1 = number2 is 47
mutex protect, number1 = number2 is 48
mutex protect, number1 = number2 is 49
msh >mutex_sample
msh >mutex protect, number1 = number2 is 1
mutex protect, number1 = number2 is 2
mutex protect, number1 = number2 is 3
mutex protect, number1 = number2 is 4
mutex protect, number1 = number2 is 5
mutex protect, number1 = number2 is 6
mutex protect, number1 = number2 is 7
mutex protect, number1 = number2 is 8
mutex protect, number1 = number2 is 9
mutex protect, number1 = number2 is 10
mutex protect, number1 = number2 is 11
mutex protect, number1 = number2 is 12
mutex protect, number1 = number2 is 13
mutex protect, number1 = number2 is 14
mutex protect, number1 = number2 is 15
mutex protect, number1 = number2 is 16
mutex protect, number1 = number2 is 17
mutex protect, number1 = number2 is 18
mutex protect, number1 = number2 is 19
mutex protect, number1 = number2 is 20
mutex protect, number1 = number2 is 21
mutex protect, number1 = number2 is 22
mutex protect, number1 = number2 is 23
mutex protect, number1 = number2 is 24
mutex protect, number1 = number2 is 25
mutex protect, number1 = number2 is 26
mutex protect, number1 = number2 is 27
mutex protect, number1 = number2 is 28
mutex protect, number1 = number2 is 29
mutex protect, number1 = number2 is 30
mutex protect, number1 = number2 is 31
mutex protect, number1 = number2 is 32
mutex protect, number1 = number2 is 33
mutex protect, number1 = number2 is 34
mutex protect, number1 = number2 is 35
mutex protect, number1 = number2 is 36
mutex protect, number1 = number2 is 37
mutex protect, number1 = number2 is 38
mutex protect, number1 = number2 is 39
mutex protect, number1 = number2 is 40
mutex protect, number1 = number2 is 41
mutex protect, number1 = number2 is 42
mutex protect, number1 = number2 is 43
mutex protect, number1 = number2 is 44
mutex protect, number1 = number2 is 45
mutex protect, number1 = number2 is 46
mutex protect, number1 = number2 is 47
mutex protect, number1 = number2 is 48
mutex protect, number1 = number2 is 49
```

线程使用互斥量保护对两个 number 的操作，使 number 值保持一致。



这个例子将创建 3 个动态线程以检查持有互斥量时，持有的线程优先级是否被调整到等待线程优先级中的最高优先级。

**防止优先级翻转特性例程**

```c
#include <rtthread.h>

/* 指向线程控制块的指针 */
static rt_thread_t tid1 = RT_NULL;
static rt_thread_t tid2 = RT_NULL;
static rt_thread_t tid3 = RT_NULL;
static rt_mutex_t mutex = RT_NULL;


#define THREAD_PRIORITY       10
#define THREAD_STACK_SIZE     512
#define THREAD_TIMESLICE    5

/* 线程 1 入口 */
static void thread1_entry(void *parameter)
{
    /* 先让低优先级线程运行 */
    rt_thread_mdelay(100);

    /* 此时 thread3 持有 mutex，并且 thread2 等待持有 mutex */

    /* 检查 thread2 与 thread3 的优先级情况 */
    if (tid2->current_priority != tid3->current_priority)
    {
        /* 优先级不相同，测试失败 */
        rt_kprintf("the priority of thread2 is: %d\n", tid2->current_priority);
        rt_kprintf("the priority of thread3 is: %d\n", tid3->current_priority);
        rt_kprintf("test failed.\n");
        return;
    }
    else
    {
        rt_kprintf("the priority of thread2 is: %d\n", tid2->current_priority);
        rt_kprintf("the priority of thread3 is: %d\n", tid3->current_priority);
        rt_kprintf("test OK.\n");
    }
}

/* 线程 2 入口 */
static void thread2_entry(void *parameter)
{
    rt_err_t result;

    rt_kprintf("the priority of thread2 is: %d\n", tid2->current_priority);

    /* 先让低优先级线程运行 */
    rt_thread_mdelay(50);

    /*
     * 试图持有互斥锁，此时 thread3 持有，应把 thread3 的优先级提升
     * 到 thread2 相同的优先级
     */
    result = rt_mutex_take(mutex, RT_WAITING_FOREVER);

    if (result == RT_EOK)
    {
        /* 释放互斥锁 */
        rt_mutex_release(mutex);
    }
}

/* 线程 3 入口 */
static void thread3_entry(void *parameter)
{
    rt_tick_t tick;
    rt_err_t result;

    rt_kprintf("the priority of thread3 is: %d\n", tid3->current_priority);

    result = rt_mutex_take(mutex, RT_WAITING_FOREVER);
    if (result != RT_EOK)
    {
        rt_kprintf("thread3 take a mutex, failed.\n");
    }

    /* 做一个长时间的循环，500ms */
    tick = rt_tick_get();
    while (rt_tick_get() - tick < (RT_TICK_PER_SECOND / 2)) ;

    rt_mutex_release(mutex);
}

int pri_inversion(void)
{
    /* 创建互斥锁 */
    mutex = rt_mutex_create("mutex", RT_IPC_FLAG_PRIO);
    if (mutex == RT_NULL)
    {
        rt_kprintf("create dynamic mutex failed.\n");
        return -1;
    }

    /* 创建线程 1 */
    tid1 = rt_thread_create("thread1",
                            thread1_entry,
                            RT_NULL,
                            THREAD_STACK_SIZE,
                            THREAD_PRIORITY - 1, THREAD_TIMESLICE);
    if (tid1 != RT_NULL)
         rt_thread_startup(tid1);

    /* 创建线程 2 */
    tid2 = rt_thread_create("thread2",
                            thread2_entry,
                            RT_NULL,
                            THREAD_STACK_SIZE,
                            THREAD_PRIORITY, THREAD_TIMESLICE);
    if (tid2 != RT_NULL)
        rt_thread_startup(tid2);

    /* 创建线程 3 */
    tid3 = rt_thread_create("thread3",
                            thread3_entry,
                            RT_NULL,
                            THREAD_STACK_SIZE,
                            THREAD_PRIORITY + 1, THREAD_TIMESLICE);
    if (tid3 != RT_NULL)
        rt_thread_startup(tid3);

    return 0;
}

/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(pri_inversion, prio_inversion sample);
```

仿真运行结果如下：

```shell
 \ | /
- RT -     Thread Operating System
 / | \     3.1.0 build Aug 27 2018
 2006 - 2018 Copyright by rt-thread team
msh >pri_inversion
the priority of thread2 is: 10
the priority of thread3 is: 11
the priority of thread2 is: 10
the priority of thread3 is: 10
test OK.
```

例程演示了互斥量的使用方法。

线程 3 先持有互斥量，而后线程 2 试图持有互斥量，此时线程 3 的优先级被提升为和线程 2 的优先级相同。

注：需要切记的是互斥量不能在中断服务例程中使用。



### 互斥量的使用场合

互斥量的使用比较单一，因为它是信号量的一种，并且它是以锁的形式存在。在初始化的时候，互斥量永远都处于开锁的状态，而被线程持有的时候则立刻转为闭锁的状态。

互斥量更适合于：

（1）线程多次持有互斥量的情况下。这样可以避免同一线程多次递归持有而造成死锁的问题。

（2）可能会由于多线程同步而造成优先级翻转的情况。



## 事件集

事件集也是线程间同步的机制之一，一个事件集可以包含多个事件，利用事件集可以完成一对多，多对多的线程间同步。

下面以坐公交为例说明事件，在公交站等公交时可能有以下几种情况：

①P1 坐公交去某地，只有一种公交可以到达目的地，等到此公交即可出发。

②P1 坐公交去某地，有 3 种公交都可以到达目的地，等到其中任意一辆即可出发。

③P1 约另一人 P2 一起去某地，则 P1 必须要等到 “同伴 P2 到达公交站” 与“公交到达公交站”两个条件都满足后，才能出发。

这里，可以将 P1 去某地视为线程，将 “公交到达公交站”、“同伴 P2 到达公交站” 视为事件的发生，

情况①是特定事件唤醒线程；

情况②是任意单个事件唤醒线程；

情况③是多个事件同时发生才唤醒线程。



#### 事件集工作机制

事件集主要用于线程间的同步，与信号量不同，它的特点是可以实现一对多，多对多的同步。

即一个线程与多个事件的关系可设置为：其中任意一个事件唤醒线程，或几个事件都到达后才唤醒线程进行后续的处理；

同样，事件也可以是多个线程同步多个事件。这种多个事件的集合可以用一个 32 位无符号整型变量来表示，变量的每一位代表一个事件，线程通过 “逻辑与” 或“逻辑或”将一个或多个事件关联起来，形成事件组合。

事件的 “逻辑或” 也称为是独立型同步，指的是线程与任何事件之一发生同步；事件 “逻辑与” 也称为是关联型同步，指的是线程与若干事件都发生同步。

RT-Thread 定义的事件集有以下特点：

1）事件只与线程相关，事件间相互独立：每个线程可拥有 32 个事件标志，采用一个 32 bit 无符号整型数进行记录，每一个 bit 代表一个事件；

2）事件仅用于同步，不提供数据传输功能；

3）事件无排队性，即多次向线程发送同一事件 (如果线程还未来得及读走)，其效果等同于**只发送一次**。

在 RT-Thread 中，每个线程都拥有一个事件信息标记，它有三个属性，分别是 RT_EVENT_FLAG_AND(逻辑与)，RT_EVENT_FLAG_OR(逻辑或）以及 RT_EVENT_FLAG_CLEAR(清除标记）。

当线程等待事件同步时，可以通过 32 个事件标志和这个事件信息标记来判断当前接收的事件是否满足同步条件。

![事件集工作示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc1/figures/06event_work.png)

如上图所示，线程 #1 的事件标志中第 1 位和第 30 位被置位，

如果事件信息标记位设为逻辑与，则表示线程 #1 只有在事件 1 和事件 30 都发生以后才会被触发唤醒，

如果事件信息标记位设为逻辑或，则事件 1 或事件 30 中的任意一个发生都会触发唤醒线程 #1。

如果信息标记同时设置了清除标记位，则当线程 #1 唤醒后将主动把事件 1 和事件 30 清为零，否则事件标志将依然存在（即置 1）。

### 事件集控制块

在 RT-Thread 中，事件集控制块是操作系统用于管理事件的一个数据结构，由结构体 struct rt_event 表示。

另外一种 C 表达方式 rt_event_t，表示的是事件集的句柄，在 C 语言中的实现是事件集控制块的指针。

事件集控制块结构的详细定义请见以下代码：

```c
struct rt_event
{
    struct rt_ipc_object parent;    /* 继承自 ipc_object 类 */

    /* 事件集合，每一 bit 表示 1 个事件，bit 位的值可以标记某事件是否发生 */
    rt_uint32_t set;
};
/* rt_event_t 是指向事件结构体的指针类型  */
typedef struct rt_event* rt_event_t;
```

rt_event 对象从 rt_ipc_object 中派生，由 IPC 容器所管理。

### 事件集的管理方式

事件集控制块中含有与事件集相关的重要参数，在事件集功能的实现中起重要的作用。

事件集相关接口如下图所示，对一个事件集的操作包含：

创建 / 初始化事件集、发送事件、接收事件、删除 / 脱离事件集。



![事件相关接口](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc1/figures/06event_ops.png)

#### 创建和删除事件集

当创建一个事件集时，内核首先创建一个事件集控制块，然后对该事件集控制块进行基本的初始化，创建事件集使用下面的函数接口：

```c
rt_event_t rt_event_create(const char* name, rt_uint8_t flag);
```

调用该函数接口时，系统会从对象管理器中分配事件集对象，并初始化这个对象，然后初始化父类 IPC 对象。

下表描述了该函数的输入参数与返回值：

| **参数**       | **描述**                                                     |
| -------------- | ------------------------------------------------------------ |
| name           | 事件集的名称                                                 |
| flag           | 事件集的标志，它可以取如下数值： RT_IPC_FLAG_FIFO 或 RT_IPC_FLAG_PRIO |
| **返回**       | ——                                                           |
| RT_NULL        | 创建失败                                                     |
| 事件对象的句柄 | 创建成功                                                     |



注：RT_IPC_FLAG_FIFO 属于非实时调度方式，除非应用程序非常在意先来后到，并且你清楚地明白所有涉及到该事件集的线程都将会变为非实时线程，方可使用 RT_IPC_FLAG_FIFO，否则建议采用 RT_IPC_FLAG_PRIO，即确保线程的实时性。



系统不再使用 rt_event_create() 创建的事件集对象时，通过删除事件集对象控制块来释放系统资源。

删除事件集可以使用下面的函数接口：

```c
rt_err_t rt_event_delete(rt_event_t event);
```

在调用 rt_event_delete 函数删除一个事件集对象时，应该确保该事件集不再被使用。

在删除前会唤醒所有挂起在该事件集上的线程（线程的返回值是 - RT_ERROR），然后释放事件集对象占用的内存块。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**         |
| -------- | ---------------- |
| event    | 事件集对象的句柄 |
| **返回** | ——               |
| RT_EOK   | 成功             |

#### 初始化和脱离事件集

静态事件集对象的内存是在系统编译时由编译器分配的，一般放于读写数据段或未初始化数据段中。

在使用静态事件集对象前，需要先行对它进行初始化操作。初始化事件集使用下面的函数接口：

```c
rt_err_t rt_event_init(rt_event_t event, const char* name, rt_uint8_t flag);
```

调用该接口时，需指定静态事件集对象的句柄（即指向事件集控制块的指针），然后系统会初始化事件集对象，并加入到系统对象容器中进行管理。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**                                                     |
| -------- | ------------------------------------------------------------ |
| event    | 事件集对象的句柄                                             |
| name     | 事件集的名称                                                 |
| flag     | 事件集的标志，它可以取如下数值： RT_IPC_FLAG_FIFO 或 RT_IPC_FLAG_PRIO |
| **返回** | ——                                                           |
| RT_EOK   | 成功                                                         |

系统不再使用 rt_event_init() 初始化的事件集对象时，通过脱离事件集对象控制块来释放系统资源。

脱离事件集是将事件集对象从内核对象管理器中脱离。

脱离事件集使用下面的函数接口：

```c
rt_err_t rt_event_detach(rt_event_t event);
```

用户调用这个函数时，系统首先**唤醒所有挂在该事件集等待队列上的线程**（线程的返回值是 - RT_ERROR），然后将该事件集从内核对象管理器中脱离。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**         |
| -------- | ---------------- |
| event    | 事件集对象的句柄 |
| **返回** | ——               |
| RT_EOK   | 成功             |

#### 发送事件

发送事件函数可以发送事件集中的一个或多个事件，如下：

```c
rt_err_t rt_event_send(rt_event_t event, rt_uint32_t set);
```

使用该函数接口时，通过参数 set 指定的事件标志来设定 event 事件集对象的事件标志值，然后遍历等待在 event 事件集对象上的等待线程链表，判断是否有线程的事件激活要求与当前 event 对象事件标志值匹配，**如果有，则唤醒该线程**。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**                     |
| -------- | ---------------------------- |
| event    | 事件集对象的句柄             |
| set      | 发送的一个或多个事件的标志值 |
| **返回** | ——                           |
| RT_EOK   | 成功                         |

#### 接收事件

内核使用 32 位的无符号整数来标识事件集，它的每一位代表一个事件，因此一个事件集对象可同时等待接收 32 个事件，内核可以通过指定选择参数 “逻辑与” 或“逻辑或”来选择如何激活线程，使用 “逻辑与” 参数表示只有当所有等待的事件都发生时才激活线程，而使用 “逻辑或” 参数则表示只要有一个等待的事件发生就激活线程。

接收事件使用下面的函数接口：

```c
rt_err_t rt_event_recv(rt_event_t event,
                           rt_uint32_t set,
                           rt_uint8_t option,
                           rt_int32_t timeout,
                           rt_uint32_t* recved);
```

当用户调用这个接口时，系统首先根据 set 参数和接收选项 option 来判断它要接收的事件是否发生，如果已经发生，则根据参数 option 上是否设置有 RT_EVENT_FLAG_CLEAR 来决定是否重置事件的相应标志位，然后返回（其中 recved 参数返回接收到的事件）；

如果没有发生，则把等待的 set 和 option 参数填入线程本身的结构中，然后把线程挂起在此事件上，直到其等待的事件满足条件或等待时间超过指定的超时时间。如果超时时间设置为零，则表示当线程要接受的事件没有满足其要求时就不等待，而直接返回 - RT_ETIMEOUT。

下表描述了该函数的输入参数与返回值：

| **参数**     | **描述**             |
| ------------ | -------------------- |
| event        | 事件集对象的句柄     |
| set          | 接收线程感兴趣的事件 |
| option       | 接收选项             |
| timeout      | 指定超时时间         |
| recved       | 指向接收到的事件     |
| **返回**     | ——                   |
| RT_EOK       | 成功                 |
| -RT_ETIMEOUT | 超时                 |
| -RT_ERROR    | 错误                 |

option 的值可取：

```c
/* 选择 逻辑与 或 逻辑或 的方式接收事件 */
RT_EVENT_FLAG_OR
RT_EVENT_FLAG_AND

/* 选择清除重置事件标志位 */
RT_EVENT_FLAG_CLEAR
```



### 事件集应用示例

这是事件集的应用例程，例子中初始化了一个事件集，两个线程。

一个线程等待自己关心的事件发生，另外一个线程发送事件，如以下代码所示：

**事件集的使用例程**

```c
#include <rtthread.h>

#define THREAD_PRIORITY      9
#define THREAD_TIMESLICE     5

#define EVENT_FLAG3 (1 << 3)
#define EVENT_FLAG5 (1 << 5)

/* 事件控制块 */
static struct rt_event event;

ALIGN(RT_ALIGN_SIZE)
static char thread1_stack[1024];
static struct rt_thread thread1;

/* 线程 1 入口函数 */
static void thread1_recv_event(void *param)
{
    rt_uint32_t e;

    /* 第一次接收事件，事件 3 或事件 5 任意一个可以触发线程 1，接收完后清除事件标志 */
    if (rt_event_recv(&event, (EVENT_FLAG3 | EVENT_FLAG5),
                      RT_EVENT_FLAG_OR | RT_EVENT_FLAG_CLEAR,
                      RT_WAITING_FOREVER, &e) == RT_EOK)
    {
        rt_kprintf("thread1: OR recv event 0x%x\n", e);
    }

    rt_kprintf("thread1: delay 1s to prepare the second event\n");
    rt_thread_mdelay(1000);

    /* 第二次接收事件，事件 3 和事件 5 均发生时才可以触发线程 1，接收完后清除事件标志 */
    if (rt_event_recv(&event, (EVENT_FLAG3 | EVENT_FLAG5),
                      RT_EVENT_FLAG_AND | RT_EVENT_FLAG_CLEAR,
                      RT_WAITING_FOREVER, &e) == RT_EOK)
    {
        rt_kprintf("thread1: AND recv event 0x%x\n", e);
    }
    /* 执行完该事件集后进行事件集的脱离，事件集重复初始化会导致再次运行时，出现重复初始化的问题 */
    rt_event_detach(&event);
    rt_kprintf("thread1 leave.\n");
}


ALIGN(RT_ALIGN_SIZE)
static char thread2_stack[1024];
static struct rt_thread thread2;

/* 线程 2 入口 */
static void thread2_send_event(void *param)
{
    rt_kprintf("thread2: send event3\n");
    rt_event_send(&event, EVENT_FLAG3);
    rt_thread_mdelay(200);

    rt_kprintf("thread2: send event5\n");
    rt_event_send(&event, EVENT_FLAG5);
    rt_thread_mdelay(200);

    rt_kprintf("thread2: send event3\n");
    rt_event_send(&event, EVENT_FLAG3);
    rt_kprintf("thread2 leave.\n");
}

int event_sample(void)
{
    rt_err_t result;

    /* 初始化事件对象 */
    result = rt_event_init(&event, "event", RT_IPC_FLAG_PRIO);
    if (result != RT_EOK)
    {
        rt_kprintf("init event failed.\n");
        return -1;
    }

    rt_thread_init(&thread1,
                   "thread1",
                   thread1_recv_event,
                   RT_NULL,
                   &thread1_stack[0],
                   sizeof(thread1_stack),
                   THREAD_PRIORITY - 1, THREAD_TIMESLICE);
    rt_thread_startup(&thread1);

    rt_thread_init(&thread2,
                   "thread2",
                   thread2_send_event,
                   RT_NULL,
                   &thread2_stack[0],
                   sizeof(thread2_stack),
                   THREAD_PRIORITY, THREAD_TIMESLICE);
    rt_thread_startup(&thread2);

    return 0;
}

/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(event_sample, event sample);
```

仿真运行结果如下：

```c
 \ | /
- RT -     Thread Operating System
 / | \     4.1.1 build Sep  5 2024 15:53:21
 2006 - 2022 Copyright by RT-Thread team
msh >event_sample
thread2: send event3
thread1: OR recv event 0x8
thread1: delay 1s to prepare the second event
msh >thread2: send event5
thread2: send event3
thread2 leave.
thread1: AND recv event 0x28
thread1 leave.

msh >event_sample
thread2: send event3
thread1: OR recv event 0x8
thread1: delay 1s to prepare the second event
msh >thread2: send event5
thread2: send event3
thread2 leave.
thread1: AND recv event 0x28
thread1 leave.
```

例程演示了事件集的使用方法。

线程 1 前后两次接收事件，分别使用了 “逻辑或” 与“逻辑与”的方法。

### 事件集的使用场合

事件集可使用于多种场合，它能够在一定程度上替代信号量，用于线程间同步。

一个线程或中断服务例程发送一个事件给事件集对象，而后等待的线程被唤醒并对相应的事件进行处理。但是它与信号量不同的是，事件的发送操作在事件未清除前，是不可累计的，而信号量的释放动作是累计的。

事件的另一个特性是，接收线程可等待多种事件，即多个事件对应一个线程或多个线程。同时按照线程等待的参数，可选择是 “逻辑或” 触发还是 “逻辑与” 触发。这个特性也是信号量等所不具备的，信号量只能识别单一的释放动作，而不能同时等待多种类型的释放。

如下图所示为多事件接收示意图：

![多事件接收示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc1/figures/06event_use.png)

一个事件集中包含 32 个事件，特定线程只等待、接收它关注的事件。

可以是一个线程等待多个事件的到来（线程 1、2 均等待多个事件，事件间可以使用 “与” 或者 “或” 逻辑触发线程），也可以是多个线程等待一个事件的到来（事件 25）。

当有它们关注的事件发生时，线程将被唤醒并进行后续的处理动作。





# 线程间通信

在裸机编程中，经常会使用全局变量进行功能间的通信，如某些功能可能由于一些操作而改变全局变量的值，另一个功能对此全局变量进行读取，根据读取到的全局变量值执行相应的动作，达到通信协作的目的。

RT-Thread 中则提供了更多的工具帮助在不同的线程中间传递信息，本章会详细介绍这些工具。

本章，大家将学会如何将邮箱、消息队列、信号用于线程间的通信。

## 邮箱

邮箱服务是实时操作系统中一种典型的线程间通信方法。举一个简单的例子，有两个线程，线程 1 检测按键状态并发送，线程 2 读取按键状态并根据按键的状态相应地改变 LED 的亮灭。这里就可以使用邮箱的方式进行通信，线程 1 将按键的状态作为邮件发送到邮箱，线程 2 在邮箱中读取邮件获得按键状态并对 LED 执行亮灭操作。

这里的线程 1 也可以扩展为多个线程。例如，共有三个线程，线程 1 检测并发送按键状态，线程 2 检测并发送 ADC 采样信息，线程 3 则根据接收的信息类型不同，执行不同的操作。

### 邮箱的工作机制

RT-Thread 操作系统的邮箱用于线程间通信，特点是开销比较低，效率较高。

邮箱中的每一封邮件只能容纳固定的 4 字节内容（针对 32 位处理系统，指针的大小即为 4 个字节，所以一封邮件恰好能够容纳一个指针）。

典型的邮箱也称作交换消息，如下图所示，线程或中断服务例程把一封 4 字节长度的邮件发送到邮箱中，而一个或多个线程可以从邮箱中接收这些邮件并进行处理。

![邮箱工作示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc2/figures/07mb_work.png)

非阻塞方式的邮件发送过程能够安全的应用于中断服务中，是线程、中断服务、定时器向线程发送消息的有效手段。

通常来说，邮件收取过程可能是阻塞的，这取决于邮箱中是否有邮件，以及收取邮件时设置的超时时间。当邮箱中不存在邮件且超时时间不为 0 时，邮件收取过程将变成阻塞方式。在这类情况下，只能由线程进行邮件的收取。

当一个线程向邮箱发送邮件时，如果邮箱没满，将把邮件复制到邮箱中。如果邮箱已经满了，发送线程可以设置超时时间，选择等待挂起或直接返回 - RT_EFULL。如果发送线程选择挂起等待，那么当邮箱中的邮件被收取而空出空间来时，等待挂起的发送线程将被唤醒继续发送。

当一个线程从邮箱中接收邮件时，如果邮箱是空的，接收线程可以选择是否等待挂起直到收到新的邮件而唤醒，或可以设置超时时间。当达到设置的超时时间，邮箱依然未收到邮件时，这个选择超时等待的线程将被唤醒并返回 - RT_ETIMEOUT。如果邮箱中存在邮件，那么接收线程将复制邮箱中的 4 个字节邮件到接收缓存中。

### 邮箱控制块

在 RT-Thread 中，邮箱控制块是操作系统用于管理邮箱的一个数据结构，由结构体 **struct rt_mailbox** 表示。另外一种 C 表达方式 **rt_mailbox_t**，表示的是邮箱的句柄，在 C 语言中的实现是邮箱控制块的指针。

邮箱控制块结构的详细定义请见以下代码：

```c
struct rt_mailbox
{
    struct rt_ipc_object parent;

    rt_uint32_t* msg_pool;                /* 邮箱缓冲区的开始地址 */
    rt_uint16_t size;                     /* 邮箱缓冲区的大小     */

    rt_uint16_t entry;                    /* 邮箱中邮件的数目     */
    rt_uint16_t in_offset, out_offset;    /* 邮箱缓冲的进出指针   */
    rt_list_t suspend_sender_thread;      /* 发送线程的挂起等待队列 */
};
typedef struct rt_mailbox* rt_mailbox_t;
```

rt_mailbox 对象从 rt_ipc_object 中派生，由 IPC 容器所管理。

### 邮箱的管理方式

邮箱控制块是一个结构体，其中含有事件相关的重要参数，在邮箱的功能实现中起重要的作用。

邮箱的相关接口如下图所示，对一个邮箱的操作包含：创建 / 初始化邮箱、发送邮件、接收邮件、删除 / 脱离邮箱。

![邮箱相关接口](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc2/figures/07mb_ops.png)

#### 创建和删除邮箱

动态创建一个邮箱对象可以调用如下的函数接口：

```c
rt_mailbox_t rt_mb_create (const char* name, rt_size_t size, rt_uint8_t flag);
```

创建邮箱对象时会先从对象管理器中分配一个邮箱对象，然后给邮箱动态分配一块内存空间用来存放邮件，这块内存的大小等于邮件大小（4 字节）与邮箱容量的乘积，接着初始化接收邮件数目和发送邮件在邮箱中的偏移量。

下表描述了该函数的输入参数与返回值：

| **参数**       | **描述**                                                     |
| -------------- | ------------------------------------------------------------ |
| name           | 邮箱名称                                                     |
| size           | 邮箱容量                                                     |
| flag           | 邮箱标志，它可以取如下数值： RT_IPC_FLAG_FIFO 或 RT_IPC_FLAG_PRIO |
| **返回**       | ——                                                           |
| RT_NULL        | 创建失败                                                     |
| 邮箱对象的句柄 | 创建成功                                                     |



注：RT_IPC_FLAG_FIFO 属于非实时调度方式，除非应用程序非常在意先来后到，并且你清楚地明白所有涉及到该邮箱的线程都将会变为非实时线程，方可使用 RT_IPC_FLAG_FIFO，否则建议采用 RT_IPC_FLAG_PRIO，即确保线程的实时性。



当用 rt_mb_create() 创建的邮箱不再被使用时，应该删除它来释放相应的系统资源，一旦操作完成，邮箱将被永久性的删除。

删除邮箱的函数接口如下：

```c
rt_err_t rt_mb_delete (rt_mailbox_t mb);
```

删除邮箱时，如果有线程被挂起在该邮箱对象上，内核先唤醒挂起在该邮箱上的所有线程（线程返回值是 -RT_ERROR），然后再释放邮箱使用的内存，最后删除邮箱对象。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**       |
| -------- | -------------- |
| mb       | 邮箱对象的句柄 |
| **返回** | ——             |
| RT_EOK   | 成功           |



#### 初始化和脱离邮箱

初始化邮箱跟创建邮箱类似，只是初始化邮箱用于静态邮箱对象的初始化。

与创建邮箱不同的是，静态邮箱对象的内存是在系统编译时由**编译器**分配的，一般放于读写数据段或未初始化数据段中，其余的初始化工作与创建邮箱时相同。

函数接口如下：

```c
  rt_err_t rt_mb_init(rt_mailbox_t mb,
                    const char* name,
                    void* msgpool,
                    rt_size_t size,
                    rt_uint8_t flag)
```

初始化邮箱时，该函数接口需要获得用户已经申请获得的邮箱对象控制块，缓冲区的指针，以及邮箱名称和邮箱容量（能够存储的邮件数）。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**                                                     |
| -------- | ------------------------------------------------------------ |
| mb       | 邮箱对象的句柄                                               |
| name     | 邮箱名称                                                     |
| msgpool  | 缓冲区指针                                                   |
| size     | 邮箱容量                                                     |
| flag     | 邮箱标志，它可以取如下数值： RT_IPC_FLAG_FIFO 或 RT_IPC_FLAG_PRIO |
| **返回** | ——                                                           |
| RT_EOK   | 成功                                                         |

这里的 size 参数指定的是邮箱的容量，即如果 msgpool 指向的缓冲区的**字节数是 N**，那么邮箱**容量**应该是 **N/4**。



脱离邮箱将把静态初始化的邮箱对象从内核对象管理器中脱离。

脱离邮箱使用下面的接口：

```c
rt_err_t rt_mb_detach(rt_mailbox_t mb);
```

使用该函数接口后，内核先唤醒所有挂在该邮箱上的线程（线程获得返回值是 - RT_ERROR），然后将该邮箱对象从内核对象管理器中脱离。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**       |
| -------- | -------------- |
| mb       | 邮箱对象的句柄 |
| **返回** | ——             |
| RT_EOK   | 成功           |

#### 发送邮件

线程或者中断服务程序可以通过邮箱给其他线程发送邮件，发送邮件函数接口如下：

```c
rt_err_t rt_mb_send (rt_mailbox_t mb, rt_uint32_t value);
```

发送的邮件可以是 **32 位任意格式的数据**，一个**整型值**或者**一个指向缓冲区的指针**。

当邮箱中的邮件已经满时，发送邮件的线程或者中断程序会**收到 -RT_EFULL 的返回值**。

下表描述了该函数的输入参数与返回值：

| **参数**  | **描述**       |
| --------- | -------------- |
| mb        | 邮箱对象的句柄 |
| value     | 邮件内容       |
| **返回**  | ——             |
| RT_EOK    | 发送成功       |
| -RT_EFULL | 邮箱已经满了   |

#### 等待方式发送邮件

用户也可以通过如下的函数接口向指定邮箱发送邮件：

```c
rt_err_t rt_mb_send_wait (rt_mailbox_t mb,
                      rt_uint32_t value,
                      rt_int32_t timeout);
```

rt_mb_send_wait() 与 rt_mb_send() 的区别在于有等待时间，如果邮箱已经满了，那么发送线程将根据设定的 timeout 参数等待邮箱中因为收取邮件而空出空间。

如果设置的超时时间到达依然没有空出空间，这时发送线程将被唤醒并返回错误码。

下表描述了该函数的输入参数与返回值：

| **参数**     | **描述**       |
| ------------ | -------------- |
| mb           | 邮箱对象的句柄 |
| value        | 邮件内容       |
| timeout      | 超时时间       |
| **返回**     | ——             |
| RT_EOK       | 发送成功       |
| -RT_ETIMEOUT | 超时           |
| -RT_ERROR    | 失败，返回错误 |

#### 发送紧急邮件

发送紧急邮件的过程与发送邮件几乎一样，唯一的不同是，当发送紧急邮件时，邮件被**直接插队**放入了邮件**队首**，这样，接收者就能够优先接收到紧急邮件，从而及时进行处理。

发送紧急邮件的函数接口如下：

```c
rt_err_t rt_mb_urgent (rt_mailbox_t mb, rt_ubase_t value);
```

下表描述了该函数的输入参数与返回值：

| **参数**  | **描述**       |
| --------- | -------------- |
| mb        | 邮箱对象的句柄 |
| value     | 邮件内容       |
| **返回**  | ——             |
| RT_EOK    | 发送成功       |
| -RT_EFULL | 邮箱已满       |

#### 接收邮件

只有当接收者接收的邮箱中有邮件时，接收者才能立即取到邮件并返回 RT_EOK 的返回值，否则接收线程会根据超时时间设置，或挂起在邮箱的等待线程队列上，或直接返回。

接收邮件函数接口如下：

```c
rt_err_t rt_mb_recv (rt_mailbox_t mb, rt_uint32_t* value, rt_int32_t timeout);
```

接收邮件时，接收者需指定接收邮件的邮箱句柄，并指定接收到的邮件存放位置以及最多能够等待的超时时间。

如果接收时设定了超时，当指定的时间内依然未收到邮件时，将返回 - RT_ETIMEOUT。

下表描述了该函数的输入参数与返回值：

| **参数**     | **描述**       |
| ------------ | -------------- |
| mb           | 邮箱对象的句柄 |
| value        | 邮件内容       |
| timeout      | 超时时间       |
| **返回**     | ——             |
| RT_EOK       | 接收成功       |
| -RT_ETIMEOUT | 超时           |
| -RT_ERROR    | 失败，返回错误 |

### 邮箱使用示例

这是一个邮箱的应用例程，初始化 2 个静态线程，一个静态的邮箱对象，其中一个线程往邮箱中发送邮件，一个线程往邮箱中收取邮件。

如下代码所示：

**邮箱的使用例程**

> 注意：RT-Thread 5.0 及更高的版本将 `ALIGN` 关键字改成了 `rt_align`，使用时注意修改。

```c
#include <rtthread.h>

#define THREAD_PRIORITY      10
#define THREAD_TIMESLICE     5

/* 邮箱控制块 */
static struct rt_mailbox mb;
/* 用于放邮件的内存池 */
static char mb_pool[128];

static char mb_str1[] = "I'm a mail!";
static char mb_str2[] = "this is another mail!";
static char mb_str3[] = "over";

ALIGN(RT_ALIGN_SIZE)
static char thread1_stack[1024];
static struct rt_thread thread1;

/* 线程 1 入口 */
static void thread1_entry(void *parameter)
{
    char *str;

    while (1)
    {
        rt_kprintf("thread1: try to recv a mail\n");

        /* 从邮箱中收取邮件 */
        if (rt_mb_recv(&mb, (rt_uint32_t *)&str, RT_WAITING_FOREVER) == RT_EOK)
        {
            rt_kprintf("thread1: get a mail from mailbox, the content:%s\n", str);
            if (str == mb_str3)
                break;

            /* 延时 100ms */
            rt_thread_mdelay(100);
        }
    }
    /* 执行邮箱对象脱离 */
    rt_mb_detach(&mb);
}

ALIGN(RT_ALIGN_SIZE)
static char thread2_stack[1024];
static struct rt_thread thread2;

/* 线程 2 入口 */
static void thread2_entry(void *parameter)
{
    rt_uint8_t count;

    count = 0;
    while (count < 10)
    {
        count ++;
        if (count & 0x1)
        {
            /* 发送 mb_str1 地址到邮箱中 */
            rt_mb_send(&mb, (rt_uint32_t)&mb_str1);
        }
        else
        {
            /* 发送 mb_str2 地址到邮箱中 */
            rt_mb_send(&mb, (rt_uint32_t)&mb_str2);
        }

        /* 延时 200ms */
        rt_thread_mdelay(200);
    }

    /* 发送邮件告诉线程 1，线程 2 已经运行结束 */
    rt_mb_send(&mb, (rt_uint32_t)&mb_str3);
}

int mailbox_sample(void)
{
    rt_err_t result;

    /* 初始化一个 mailbox */
    result = rt_mb_init(&mb,
                        "mbt",                      /* 名称是 mbt */
                        &mb_pool[0],                /* 邮箱用到的内存池是 mb_pool */
                        sizeof(mb_pool) / 4,        /* 邮箱中的邮件数目，因为一封邮件占 4 字节 */
                        RT_IPC_FLAG_FIFO);          /* 采用 FIFO 方式进行线程等待 */
    if (result != RT_EOK)
    {
        rt_kprintf("init mailbox failed.\n");
        return -1;
    }

    rt_thread_init(&thread1,
                   "thread1",
                   thread1_entry,
                   RT_NULL,
                   &thread1_stack[0],
                   sizeof(thread1_stack),
                   THREAD_PRIORITY, THREAD_TIMESLICE);
    rt_thread_startup(&thread1);

    rt_thread_init(&thread2,
                   "thread2",
                   thread2_entry,
                   RT_NULL,
                   &thread2_stack[0],
                   sizeof(thread2_stack),
                   THREAD_PRIORITY, THREAD_TIMESLICE);
    rt_thread_startup(&thread2);
    return 0;
}

/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(mailbox_sample, mailbox sample);
```

仿真运行结果如下：

```
 \ | /
- RT -     Thread Operating System
 / | \     3.1.0 build Aug 27 2018
 2006 - 2018 Copyright by rt-thread team
msh >mailbox_sample
thread1: try to recv a mail
thread1: get a mail from mailbox, the content:I'm a mail!
msh >thread1: try to recv a mail
thread1: get a mail from mailbox, the content:this is another mail!
…
thread1: try to recv a mail
thread1: get a mail from mailbox, the content:this is another mail!
thread1: try to recv a mail
thread1: get a mail from mailbox, the content:over
```

例程演示了邮箱的使用方法。

线程 2 发送邮件，共发送 11 次；线

程 1 接收邮件，共接收到 11 封邮件，将邮件内容打印出来，并判断结束。

### 邮箱的使用场合

邮箱是一种简单的线程间消息传递方式，特点是开销比较低，效率较高。

在 RT-Thread 操作系统的实现中能够一次传递一个 4 字节大小的邮件，并且邮箱具备一定的存储功能，能够缓存一定数量的邮件数 (邮件数由创建、初始化邮箱时指定的容量决定)。邮箱中一封邮件的最大长度是 4 字节，所以邮箱能够用于不超过 4 字节的消息传递。由于在 32 系统上 4 字节的内容恰好可以放置一个指针，因此当需要在线程间**传递比较大**的消息时，可以把**指向一个缓冲区的指针**作为邮件发送到邮箱中，即**邮箱也可以传递指针**，例如：

```c
struct msg
{
    rt_uint8_t *data_ptr;
    rt_uint32_t data_size;
};
```

对于这样一个消息结构体，其中包含了指向数据的指针 data_ptr 和数据块长度的变量 data_size。

当一个线程需要把这个消息发送给另外一个线程时，可以采用如下的操作：

```c
struct msg* msg_ptr;

msg_ptr = (struct msg*)rt_malloc(sizeof(struct msg));
msg_ptr->data_ptr = ...; /* 指向相应的数据块地址 */
msg_ptr->data_size = len; /* 数据块的长度 */
/* 发送这个消息指针给 mb 邮箱 */
rt_mb_send(mb, (rt_uint32_t)msg_ptr);
```

而在接收线程中，因为收取过来的是指针，而 msg_ptr 是一个新分配出来的内存块，所以在接收线程处理完毕后，需要释放相应的内存块：

```c
struct msg* msg_ptr;
if (rt_mb_recv(mb, (rt_uint32_t*)&msg_ptr) == RT_EOK)
{
    /* 在接收线程处理完毕后，需要释放相应的内存块 */
    rt_free(msg_ptr);
}
```



## 消息队列

消息队列是另一种常用的线程间通讯方式，是邮箱的扩展。

可以应用在多种场合：**线程间的消息交换**、**使用串口接收不定长数据**等。

### 消息队列的工作机制

消息队列能够**接收**来自线程或中断服务例程中**不固定长度**的消息，并把消息缓存在自己的内存空间中。

**其他线程**也能够从消息队列中**读取**相应的消息，而当消息队列是空的时候，可以挂起读取线程。当有新的消息到达时，挂起的线程将被唤醒以接收并处理消息。

消息队列是一种**异步的**通信方式。

如下图所示，线程或中断服务例程可以将一条或多条消息放入消息队列中。同样，一个或多个线程也可以从消息队列中获得消息。当有多个消息发送到消息队列时，通常将先进入消息队列的消息先传给线程，也就是说，线程先得到的是最先进入消息队列的消息，即**先进先出**原则 (FIFO)。

![消息队列工作示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc2/figures/07msg_work.png)

RT-Thread 操作系统的消息队列对象由多个元素组成，当消息队列被创建时，它就被分配了消息队列控制块：消息队列名称、内存缓冲区、消息大小以及队列长度等。

同时每个消息队列对象中包含着多个消息框，每个消息框可以存放一条消息；

消息队列中的第一个和最后一个消息框被分别称为**消息链表头**和**消息链表尾**，对应于消息队列控制块中的 **msg_queue_head** 和 **msg_queue_tail**；

有些消息框可能是空的，它们通过 msg_queue_free 形成一个空闲消息框链表。

所有消息队列中的消息框总数即是消息队列的长度，这个长度可在消息队列创建时指定。

### 消息队列控制块

在 RT-Thread 中，消息队列控制块是操作系统用于管理消息队列的一个数据结构，由结构体 struct rt_messagequeue 表示。另外一种 C 表达方式 rt_mq_t，表示的是消息队列的句柄，在 C 语言中的实现是消息队列控制块的指针。

消息队列控制块结构的详细定义请见以下代码：

```c
struct rt_messagequeue
{
    struct rt_ipc_object parent;

    void* msg_pool;                     /* 指向存放消息的缓冲区的指针 */

    rt_uint16_t msg_size;               /* 每个消息的长度 */
    rt_uint16_t max_msgs;               /* 最大能够容纳的消息数 */

    rt_uint16_t entry;                  /* 队列中已有的消息数 */

    void* msg_queue_head;               /* 消息链表头 */
    void* msg_queue_tail;               /* 消息链表尾 */
    void* msg_queue_free;               /* 空闲消息链表 */

    rt_list_t suspend_sender_thread;    /* 发送线程的挂起等待队列 */
};
typedef struct rt_messagequeue* rt_mq_t;
```

rt_messagequeue 对象从 rt_ipc_object 中派生，由 IPC 容器所管理。

### 消息队列的管理方式

消息队列控制块是一个结构体，其中含有消息队列相关的重要参数，在消息队列的功能实现中起重要的作用。消息队列的相关接口如下图所示，对一个消息队列的操作包含：创建消息队列 - 发送消息 - 接收消息 - 删除消息队列。

![消息队列相关接口](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc2/figures/07msg_ops.png)

#### 创建和删除消息队列

消息队列在使用前，应该被创建出来，或对已有的静态消息队列对象进行初始化，创建消息队列的函数接口如下所示：

```c
rt_mq_t rt_mq_create(const char* name, rt_size_t msg_size,
            rt_size_t max_msgs, rt_uint8_t flag);
```

创建消息队列时先从对象管理器中分配一个消息队列对象，然后给消息队列对象分配一块内存空间，组织成空闲消息链表，这块内存的大小 =[消息大小 + 消息头（用于链表连接）的大小]X 消息队列最大个数，接着再初始化消息队列，此时消息队列为空。

下表描述了该函数的输入参数与返回值：

| **参数**           | **描述**                                                     |
| ------------------ | ------------------------------------------------------------ |
| name               | 消息队列的名称                                               |
| msg_size           | 消息队列中一条消息的最大长度，单位字节                       |
| max_msgs           | 消息队列的最大个数                                           |
| flag               | 消息队列采用的等待方式，它可以取如下数值： RT_IPC_FLAG_FIFO 或 RT_IPC_FLAG_PRIO |
| **返回**           | ——                                                           |
| RT_EOK             | 发送成功                                                     |
| 消息队列对象的句柄 | 成功                                                         |
| RT_NULL            | 失败                                                         |



注：RT_IPC_FLAG_FIFO 属于非实时调度方式，除非应用程序非常在意先来后到，并且你清楚地明白所有涉及到该消息队列的线程都将会变为非实时线程，方可使用 RT_IPC_FLAG_FIFO，否则建议采用 RT_IPC_FLAG_PRIO，即确保线程的实时性。



当消息队列不再被使用时，应该删除它以释放系统资源，一旦操作完成，消息队列将被永久性地删除。

删除消息队列的函数接口如下：

```c
rt_err_t rt_mq_delete(rt_mq_t mq);
```

删除消息队列时，如果有线程被挂起在该消息队列等待队列上，则内核先唤醒挂起在该消息等待队列上的所有线程（线程返回值是 - RT_ERROR），然后再释放消息队列使用的内存，最后删除消息队列对象。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**           |
| -------- | ------------------ |
| mq       | 消息队列对象的句柄 |
| **返回** | ——                 |
| RT_EOK   | 成功               |

#### 初始化和脱离消息队列

初始化静态消息队列对象跟创建消息队列对象类似，只是静态消息队列对象的内存是在系统编译时由编译器分配的，一般放于读数据段或未初始化数据段中。在使用这类静态消息队列对象前，需要进行初始化。

初始化消息队列对象的函数接口如下：

```c
rt_err_t rt_mq_init(rt_mq_t mq, const char* name,
                        void *msgpool, rt_size_t msg_size,
                        rt_size_t pool_size, rt_uint8_t flag);
```

初始化消息队列时，该接口需要用户已经申请获得的消息队列对象的句柄（即指向消息队列对象控制块的指针）、消息队列名、消息缓冲区指针、消息大小以及消息队列缓冲区大小。

消息队列初始化后所有消息都挂在空闲消息链表上，消息队列为空。

下表描述了该函数的输入参数与返回值：

| **参数**  | **描述**                                                     |
| --------- | ------------------------------------------------------------ |
| mq        | 消息队列对象的句柄                                           |
| name      | 消息队列的名称                                               |
| msgpool   | 指向存放消息的缓冲区的指针                                   |
| msg_size  | 消息队列中一条消息的最大长度，单位字节                       |
| pool_size | 存放消息的缓冲区大小                                         |
| flag      | 消息队列采用的等待方式，它可以取如下数值： RT_IPC_FLAG_FIFO 或 RT_IPC_FLAG_PRIO |
| **返回**  | ——                                                           |
| RT_EOK    | 成功                                                         |



脱离消息队列将使消息队列对象被从内核对象管理器中脱离。

脱离消息队列使用下面的接口：

```c
rt_err_t rt_mq_detach(rt_mq_t mq);
```

使用该函数接口后，内核先唤醒所有挂在该消息等待队列对象上的线程（线程返回值是 -RT_ERROR），然后将该消息队列对象从内核对象管理器中脱离。

下表描述了该函数的输入参数与返回值：

| **参数** | **描述**           |
| -------- | ------------------ |
| mq       | 消息队列对象的句柄 |
| **返回** | ——                 |
| RT_EOK   | 成功               |

#### 发送消息

线程或者中断服务程序都可以给消息队列发送消息。

当发送消息时，消息队列对象先从空闲消息链表上取下一个空闲消息块，把线程或者中断服务程序发送的消息内容复制到消息块上，然后把该消息块挂到消息队列的尾部。当且仅当空闲消息链表上有可用的空闲消息块时，发送者才能成功发送消息；当空闲消息链表上无可用消息块，说明消息队列已满，此时，发送消息的的线程或者中断程序会收到一个错误码（-RT_EFULL）。

发送消息的函数接口如下：

```c
rt_err_t rt_mq_send (rt_mq_t mq, void* buffer, rt_size_t size);
```

发送消息时，发送者需指定发送的消息队列的对象句柄（即指向消息队列控制块的指针），并且指定发送的消息内容以及消息大小。在发送一个普通消息之后，空闲消息链表上的队首消息被转移到了消息队列尾。

下表描述了该函数的输入参数与返回值：

| **参数**  | **描述**                                             |
| --------- | ---------------------------------------------------- |
| mq        | 消息队列对象的句柄                                   |
| buffer    | 消息内容                                             |
| size      | 消息大小                                             |
| **返回**  | ——                                                   |
| RT_EOK    | 成功                                                 |
| -RT_EFULL | 消息队列已满                                         |
| -RT_ERROR | 失败，表示发送的消息长度大于消息队列中消息的最大长度 |

#### 等待方式发送消息

用户也可以通过如下的函数接口向指定的消息队列中发送消息：

```c
rt_err_t rt_mq_send_wait(rt_mq_t     mq,
                         const void *buffer,
                         rt_size_t   size,
                         rt_int32_t  timeout);
```

rt_mq_send_wait() 与 rt_mq_send() 的区别在于有等待时间，如果消息队列已经满了，那么发送线程将根据设定的 timeout 参数进行等待。

如果设置的超时时间到达依然没有空出空间，这时发送线程将被唤醒并返回错误码。

下表描述了该函数的输入参数与返回值：

| **参数**  | **描述**                                             |
| --------- | ---------------------------------------------------- |
| mq        | 消息队列对象的句柄                                   |
| buffer    | 消息内容                                             |
| size      | 消息大小                                             |
| timeout   | 超时时间                                             |
| **返回**  | ——                                                   |
| RT_EOK    | 成功                                                 |
| -RT_EFULL | 消息队列已满                                         |
| -RT_ERROR | 失败，表示发送的消息长度大于消息队列中消息的最大长度 |

#### 发送紧急消息

发送紧急消息的过程与发送消息几乎一样，唯一的不同是，当发送紧急消息时，从空闲消息链表上取下来的消息块不是挂到消息队列的队尾，而是挂到队首，这样，接收者就能够优先接收到紧急消息，从而及时进行消息处理。

发送紧急消息的函数接口如下：

```c
rt_err_t rt_mq_urgent(rt_mq_t mq, void* buffer, rt_size_t size);
```

下表描述了该函数的输入参数与返回值：

| **参数**  | **描述**           |
| --------- | ------------------ |
| mq        | 消息队列对象的句柄 |
| buffer    | 消息内容           |
| size      | 消息大小           |
| **返回**  | ——                 |
| RT_EOK    | 成功               |
| -RT_EFULL | 消息队列已满       |
| -RT_ERROR | 失败               |

#### 接收消息

当消息队列中有消息时，接收者才能接收消息，否则接收者会根据超时时间设置，或挂起在消息队列的等待线程队列上，或直接返回。

接收消息函数接口如下：

```c
rt_ssize_t rt_mq_recv (rt_mq_t mq, void* buffer,
                    rt_size_t size, rt_int32_t timeout);
```

接收消息时，接收者需指定存储消息的消息队列对象句柄，并且指定一个内存缓冲区，接收到的消息内容将被复制到该缓冲区里。此外，还需指定未能及时取到消息时的超时时间。

下表描述了该函数的输入参数与返回值：

| **参数**         | **描述**           |
| ---------------- | ------------------ |
| mq               | 消息队列对象的句柄 |
| buffer           | 消息内容           |
| size             | 消息大小           |
| timeout          | 指定的超时时间     |
| **返回**         | ——                 |
| 接收到消息的长度 | 成功收到           |
| -RT_ETIMEOUT     | 超时               |
| -RT_ERROR        | 失败，返回错误     |

### 消息队列应用示例

这是一个消息队列的应用例程，例程中初始化了 2 个静态线程，一个线程会从消息队列中收取消息；另一个线程会定时给消息队列发送普通消息和紧急消息，如下代码所示：

**消息队列的使用例程**

```c
#include <rtthread.h>

#define THREAD1_PRIORITY         25
#define THREAD1_STACK_SIZE       512
#define THREAD1_TIMESLICE        5

#define THREAD2_PRIORITY         25
#define THREAD2_STACK_SIZE       512
#define THREAD2_TIMESLICE        5

/* 消息队列控制块 */
static struct rt_messagequeue mq;
/* 消息队列中用到的放置消息的内存池 */
static rt_uint8_t msg_pool[2048];

static rt_thread_t tid1 = RT_NULL;
/* 线程 1 入口函数 */
static void thread1_entry(void *parameter)
{
    char buf = 0;
    rt_uint8_t cnt = 0;

    while (1)
    {
        /* 从消息队列中接收消息 */
#if defined(RT_VERSION_CHECK) && (RTTHREAD_VERSION >= RT_VERSION_CHECK(5, 0, 1))
        if (rt_mq_recv(&mq, &buf, sizeof(buf), RT_WAITING_FOREVER) > 0)
#else
        if (rt_mq_recv(&mq, &buf, sizeof(buf), RT_WAITING_FOREVER) == RT_EOK)
#endif  
        {
            rt_kprintf("thread1: recv msg from msg queue, the content:%c\n", buf);
            if (cnt == 19)
            {
                break;
            }
        }
        /* 延时 50ms */
        cnt++;
        rt_thread_mdelay(50);
    }
    rt_kprintf("thread1: detach mq \n");
    rt_mq_detach(&mq);
}


static rt_thread_t tid2 = RT_NULL;
/* 线程 2 入口 */
static void thread2_entry(void *parameter)
{
    int result;
    char buf = 'A';
    rt_uint8_t cnt = 0;

    while (1)
    {
        if (cnt == 8)
        {
            /* 发送紧急消息到消息队列中 */
            result = rt_mq_urgent(&mq, &buf, 1);
            if (result != RT_EOK)
            {
                rt_kprintf("rt_mq_urgent ERR\n");
            }
            else
            {
                rt_kprintf("thread2: send urgent message - %c\n", buf);
            }
        }
        else if (cnt>= 20)/* 发送 20 次消息之后退出 */
        {
            rt_kprintf("message queue stop send, thread2 quit\n");
            break;
        }
        else
        {
            /* 发送消息到消息队列中 */
            result = rt_mq_send(&mq, &buf, 1);
            if (result != RT_EOK)
            {
                rt_kprintf("rt_mq_send ERR\n");
            }

            rt_kprintf("thread2: send message - %c\n", buf);
        }
        buf++;
        cnt++;
        /* 延时 5ms */
        rt_thread_mdelay(5);
    }
}

/* 消息队列示例的初始化 */
int msgq_sample(void)
{
    rt_err_t result;

    /* 初始化消息队列 */
    result = rt_mq_init(&mq,
                        "mqt",
                        &msg_pool[0],             /* 内存池指向 msg_pool */
                        1,                          /* 每个消息的大小是 1 字节 */
                        sizeof(msg_pool),        /* 内存池的大小是 msg_pool 的大小 */
                        RT_IPC_FLAG_PRIO);       /* 如果有多个线程等待，优先级大小的方法分配消息 */

    if (result != RT_EOK)
    {
        rt_kprintf("init message queue failed.\n");
        return -1;
    }

    /* 动态创建线程1 */
    tid1 = rt_thread_create("thread1",
                   thread1_entry, RT_NULL,
                   THREAD1_STACK_SIZE,
                   THREAD1_PRIORITY, THREAD1_TIMESLICE);

    if (tid1 != RT_NULL)
        rt_thread_startup(tid1);

    /* 动态创建线程2 */
    tid2 = rt_thread_create("thread2",
                   thread2_entry, RT_NULL,
                   THREAD2_STACK_SIZE,
                   THREAD2_PRIORITY, THREAD2_TIMESLICE);

    if (tid2 != RT_NULL)
        rt_thread_startup(tid2);

    return 0;
}

/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(msgq_sample, msgq sample);复制错误复制成功
```

仿真运行结果如下：

```
\ | /
- RT -     Thread Operating System
 / | \     3.1.0 build Aug 24 2018
 2006 - 2018 Copyright by rt-thread team
msh > msgq_sample
msh >thread2: send message - A
thread1: recv msg from msg queue, the content:A
thread2: send message - B
thread2: send message - C
thread2: send message - D
thread2: send message - E
thread1: recv msg from msg queue, the content:B
thread2: send message - F
thread2: send message - G
thread2: send message - H
thread2: send urgent message - I
thread2: send message - J
thread1: recv msg from msg queue, the content:I
thread2: send message - K
thread2: send message - L
thread2: send message - M
thread2: send message - N
thread2: send message - O
thread1: recv msg from msg queue, the content:C
thread2: send message - P
thread2: send message - Q
thread2: send message - R
thread2: send message - S
thread2: send message - T
thread1: recv msg from msg queue, the content:D
message queue stop send, thread2 quit
thread1: recv msg from msg queue, the content:E
thread1: recv msg from msg queue, the content:F
thread1: recv msg from msg queue, the content:G
…
thread1: recv msg from msg queue, the content:T
thread1: detach mq
```

例程演示了消息队列的使用方法。

线程 1 会从消息队列中收取消息；线程 2 定时给消息队列发送普通消息和紧急消息。

由于线程 2 发送消息 “I” 是紧急消息，会直接插入消息队列的队首，所以线程 1 在接收到消息 “B” 后，接收的是该紧急消息，之后才接收消息“C”。

### 消息队列的使用场合

消息队列可以应用于发送不定长消息的场合，包括**线程与线程间的消息交换**，以及**中断服务例程中给线程发送消息**（中断服务例程不能接收消息）。

下面分发送消息和同步消息两部分来介绍消息队列的使用。

#### 发送消息

消息队列和邮箱的明显不同是**消息的长度并不限定在 4 个字节以内**；

另外，消息队列也包括了一个**发送紧急消息**的函数接口。

但是当创建的是一个所有消息的最大长度是 4 字节的消息队列时，消息队列对象将蜕化成邮箱。

这个**不限定长度**的消息，也及时的反应到了代码编写的场合上，同样是类似邮箱的代码：

```c
struct msg
{
    rt_uint8_t *data_ptr;    /* 数据块首地址 */
    rt_uint32_t data_size;   /* 数据块大小   */
};
```

和邮箱例子相同的消息结构定义，假设依然需要发送这样一个消息给接收线程。

在邮箱例子中，这个**结构只能够发送指向这个结构的指针**（在函数指针被发送过去后，接收线程能够正确的访问指向这个地址的内容，通常这块数据需要留给接收线程来释放）。

而使用消息队列的方式则大不相同：

```c
void send_op(void *data, rt_size_t length)
{
    struct msg msg_ptr;

    msg_ptr.data_ptr = data;  /* 指向相应的数据块地址 */
    msg_ptr.data_size = length; /* 数据块的长度 */

    /* 发送这个消息指针给 mq 消息队列 */
    rt_mq_send(mq, (void*)&msg_ptr, sizeof(struct msg));
}
```

注意，上面的代码中，是把一个局部变量的数据内容发送到了消息队列中。

在接收线程中，同样也采用局部变量进行消息接收的结构体：

```c
void message_handler()
{
    struct msg msg_ptr; /* 用于放置消息的局部变量 */

    /* 从消息队列中接收消息到 msg_ptr 中 */
    if (rt_mq_recv(mq, (void*)&msg_ptr, sizeof(struct msg), RT_WAITING_FOREVER) > 0)
    {
        /* 成功接收到消息，进行相应的数据处理 */
    }
}
```

因为消息队列是直接的数据内容复制，所以在上面的例子中，都采用了局部变量的方式保存消息结构体，这样也就免去动态内存分配的烦恼了（也就不用担心，接收线程在接收到消息时，消息内存空间已经被释放）。

#### 同步消息

在一般的系统设计中会经常遇到要**发送同步消息**的问题，这个时候就可以根据当时状态的不同选择相应的实现：两个线程间可以采用 **[消息队列 + 信号量或邮箱]** 的形式实现。发送线程通过消息发送的形式发送相应的消息给消息队列，发送完毕后希望获得接收线程的收到确认，工作示意图如下图所示：

![同步消息示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc2/figures/07msg_syn.png)

根据消息确认的不同，可以把消息结构体定义成：

```c
struct msg
{
    /* 消息结构其他成员 */
    struct rt_mailbox ack;
};
/* 或者 */
struct msg
{
    /* 消息结构其他成员 */
    struct rt_semaphore ack;
};
```

第一种类型的消息使用了邮箱来作为确认标志，而第二种类型的消息采用了信号量来作为确认标志。

邮箱作为确认标志，代表着接收线程**能够通知一些状态值**给发送线程；而信号量作为确认标志**只能够单一的通知**发送线程，消息已经确认接收。



## 信号

信号（又称为**软中断**信号），在软件层次上是**对中断机制的一种模拟**，在原理上，一个线程收到一个信号与处理器收到一个中断请求可以说是类似的。

### 信号的工作机制

信号在 RT-Thread 中用作异步通信，POSIX 标准定义了 sigset_t 类型来定义一个信号集，然而 sigset_t 类型在不同的系统可能有不同的定义方式，在 RT-Thread 中，将 sigset_t 定义成了 unsigned long 型，并命名为 rt_sigset_t，应用程序能够使用的信号为 SIGUSR1（10）和 SIGUSR2（12）。

信号本质是软中断，**用来通知线程发生了异步事件**，**用做线程之间的异常通知**、**应急处理**。

一个线程不必通过任何操作来等待信号的到达，事实上，线程也不知道信号到底什么时候到达，线程之间可以互相通过调用 rt_thread_kill() 发送软中断信号。

收到信号的线程对各种信号有不同的处理方法，处理方法可以分为三类：

第一种是类似中断的处理程序，对于需要处理的信号，线程可以指定处理函数，由该函数来处理。

第二种方法是，忽略某个信号，对该信号不做任何处理，就像未发生过一样。

第三种方法是，对该信号的处理保留系统的默认值。

如下图所示，假设线程 1 需要对信号进行处理，首先线程 1 安装一个信号并解除阻塞，并在安装的同时设定了对信号的异常处理方式；然后其他线程可以给线程 1 发送信号，触发线程 1 对该信号的处理。

![信号工作机制](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc2/figures/07signal_work.png)

当信号被传递给线程 1 时，如果它正处于挂起状态，那会把状态改为就绪状态去处理对应的信号。如果它正处于运行状态，那么会在它当前的线程栈基础上建立新栈帧空间去处理对应的信号，需要注意的是使用的线程栈大小也会相应增加。

### 信号的管理方式

对于信号的操作，有以下几种：安装信号、阻塞信号、阻塞解除、信号发送、信号等待。

信号的接口详见下图：

![信号相关接口](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc2/figures/07signal_ops.png)

#### 安装信号

如果线程要处理某一信号，那么就要在线程中安装该信号。

安装信号主要用来**确定信号值**及**线程针对该信号值的动作之间的映射关系**，即线程将要处理哪个信号，该信号被传递给线程时，将执行何种操作。

详细定义请见以下代码：

```c
rt_sighandler_t rt_signal_install(int signo, rt_sighandler_t[] handler);
```

其中 rt_sighandler_t 是定义信号处理函数的函数指针类型。

下表描述了该函数的输入参数与返回值：

| **参数**                | **描述**                                                   |
| ----------------------- | ---------------------------------------------------------- |
| signo                   | 信号值（只有 SIGUSR1 和 SIGUSR2 是开放给用户使用的，下同） |
| handler                 | 设置对信号值的处理方式                                     |
| **返回**                | ——                                                         |
| SIG_ERR                 | 错误的信号                                                 |
| 安装信号前的 handler 值 | 成功                                                       |

在信号安装时设定 handler 参数，决定了该信号的不同的处理方法。

处理方法可以分为三种：

1）类似中断的处理方式，参数指向当信号发生时用户自定义的处理函数，由该函数来处理。

2）参数设为 SIG_IGN，忽略某个信号，对该信号不做任何处理，就像未发生过一样。

3）参数设为 SIG_DFL，系统会调用默认的处理函数_signal_default_handler()。



#### 阻塞信号

信号阻塞，也可以理解为屏蔽信号。如果该信号被阻塞，则该信号将不会递达给安装此信号的线程，也不会引发软中断处理。

调 rt_signal_mask() 可以使信号阻塞：

```c
void rt_signal_mask(int signo);
```

下表描述了该函数的输入参数：

| **参数** | **描述** |
| -------- | -------- |
| signo    | 信号值   |



#### 解除信号阻塞

线程中可以安装好几个信号，使用此函数可以对其中一些信号给予 “关注”，那么发送这些信号都会引发该线程的软中断。

调用 rt_signal_unmask() 可以用来解除信号阻塞：

```c
void rt_signal_unmask(int signo);
```

下表描述了该函数的输入参数：

rt_signal_unmask() 函数参数

| **参数** | **描述** |
| -------- | -------- |
| signo    | 信号值   |



#### 发送信号

当需要进行异常处理时，可以给设定了处理异常的线程发送信号，调用 rt_thread_kill() 可以用来向任何线程发送信号：

```c
int rt_thread_kill(rt_thread_t tid, int sig);
```

下表描述了该函数的输入参数与返回值：

| **参数**   | **描述**       |
| ---------- | -------------- |
| tid        | 接收信号的线程 |
| sig        | 信号值         |
| **返回**   | ——             |
| RT_EOK     | 发送成功       |
| -RT_EINVAL | 参数错误       |

#### 等待信号

等待 set 信号的到来，如果没有等到这个信号，则将线程挂起，直到等到这个信号或者等待时间超过指定的超时时间 timeout。

如果等到了该信号，则将指向该信号体的指针存入 si，如下是等待信号的函数。

```c
int rt_signal_wait(const rt_sigset_t *set,
                        rt_siginfo_t[] *si, rt_int32_t timeout);
```

其中 rt_siginfo_t 是定义信号信息的数据类型，下表描述了该函数的输入参数与返回值：

| **参数**     | **描述**                   |
| ------------ | -------------------------- |
| set          | 指定等待的信号             |
| si           | 指向存储等到信号信息的指针 |
| timeout      | 指定的等待时间             |
| **返回**     | ——                         |
| RT_EOK       | 等到信号                   |
| -RT_ETIMEOUT | 超时                       |
| -RT_EINVAL   | 参数错误                   |

### 信号应用示例

这是一个信号的应用例程，如下代码所示。此例程创建了 1 个线程，在安装信号时，信号处理方式设为自定义处理，定义的信号的处理函数为 thread1_signal_handler()。待此线程运行起来安装好信号之后，给此线程发送信号。此线程将接收到信号，并打印信息。



> 注意：**使用信号前需要先在menuconfig中使能**

![信号量使能](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/ipc2/figures/07signal_enable.png)

**信号使用例程**

```c
#include <rtthread.h>

#define THREAD_PRIORITY         25
#define THREAD_STACK_SIZE       512
#define THREAD_TIMESLICE        5

static rt_thread_t tid1 = RT_NULL;

/* 线程 1 的信号处理函数 */
void thread1_signal_handler(int sig)
{
    rt_kprintf("thread1 received signal %d\n", sig);
}

/* 线程 1 的入口函数 */
static void thread1_entry(void *parameter)
{
    int cnt = 0;

    /* 安装信号 */
    rt_signal_install(SIGUSR1, thread1_signal_handler);
    rt_signal_unmask(SIGUSR1);

    /* 运行 10 次 */
    while (cnt < 10)
    {
        /* 线程 1 采用低优先级运行，一直打印计数值 */
        rt_kprintf("thread1 count : %d\n", cnt);

        cnt++;
        rt_thread_mdelay(100);
    }
}

/* 信号示例的初始化 */
int signal_sample(void)
{
    /* 创建线程 1 */
    tid1 = rt_thread_create("thread1",
                            thread1_entry, RT_NULL,
                            THREAD_STACK_SIZE,
                            THREAD_PRIORITY, THREAD_TIMESLICE);

    if (tid1 != RT_NULL)
        rt_thread_startup(tid1);

    rt_thread_mdelay(300);

    /* 发送信号 SIGUSR1 给线程 1 */
    rt_thread_kill(tid1, SIGUSR1);

    return 0;
}

/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(signal_sample, signal sample);
```

仿真运行结果如下：

```
 \ | /
- RT -     Thread Operating System
 / | \     3.1.0 build Aug 24 2018
 2006 - 2018 Copyright by rt-thread team
msh >signal_sample
thread1 count : 0
thread1 count : 1
thread1 count : 2
msh >thread1 received signal 10
thread1 count : 3
thread1 count : 4
thread1 count : 5
thread1 count : 6
thread1 count : 7
thread1 count : 8
thread1 count : 9
```

例程中，首先线程安装信号并解除阻塞，然后发送信号给线程。

线程接收到信号并打印出了接收到的信号：**SIGUSR1（10）**。





# 内存管理

在计算系统中，通常存储空间可以分为两种：内部存储空间和外部存储空间。

内部存储空间通常访问速度比较快，能够按照变量地址随机地访问，也就是我们通常所说的 **RAM**（随机存储器），可以把它理解为电脑的内存；

而外部存储空间内所保存的内容相对来说比较固定，即使掉电后数据也不会丢失，这就是通常所讲的 **ROM**（只读存储器），可以把它理解为电脑的硬盘。

计算机系统中，变量、中间数据一般存放在 RAM 中，只有在实际使用时才将它们从 RAM 调入到 CPU 中进行运算。一些数据需要的内存大小需要在程序运行过程中根据实际情况确定，这就要求系统具有对内存空间进行动态管理的能力，在用户需要一段内存空间时，向系统申请，系统选择一段合适的内存空间分配给用户，用户使用完毕后，再释放回系统，以便系统将该段内存空间回收再利用。

这章主要介绍 RT-Thread 中的两种内存管理方式，动态内存堆管理和静态内存池管理，学完本章会了解 RT-Thread 的内存管理原理及使用方式。

## 内存管理的功能特点

由于实时系统中对时间的要求非常严格，内存管理往往要比通用操作系统要求苛刻得多：

1）**分配内存的时间必须是确定的**。一般内存管理算法是根据需要存储的数据的长度在内存中去寻找一个与这段数据相适应的空闲内存块，然后将数据存储在里面。而寻找这样一个空闲内存块所耗费的时间是不确定的，因此对于实时系统来说，这就是不可接受的，实时系统必须要保证内存块的分配过程在可预测的确定时间内完成，否则实时任务对外部事件的响应也将变得不可确定。

2）**随着内存不断被分配和释放，整个内存区域会产生越来越多的碎片**（因为在使用过程中，申请了一些内存，其中一些释放了，导致内存空间中存在一些小的内存块，它们地址不连续，不能够作为一整块的大内存分配出去），系**统中还有足够的空闲内存，但因为它们地址并非连续，不能组成一块连续的完整内存块，会使得程序不能申请到大的内存**。对于通用系统而言，这种不恰当的内存分配算法可以通过重新启动系统来解决 (每个月或者数个月进行一次)，但是对于那些需要常年不间断地工作于野外的嵌入式系统来说，就变得让人无法接受了。

3）嵌入式系统的资源环境也是不尽相同，有些系统的资源比较紧张，只有数十 KB 的内存可供分配，而有些系统则存在数 MB 的内存，如何为这些不同的系统，选择适合它们的高效率的内存分配算法，就将变得复杂化。

RT-Thread 操作系统在内存管理上，根据上层应用及系统资源的不同，有针对性地提供了不同的内存分配管理算法。

总体上可分为两类：内存堆管理与内存池管理，而内存堆管理又根据具体内存设备划分为三种情况：

第一种是针对小内存块的分配管理（小内存管理算法）；

第二种是针对大内存块的分配管理（slab 管理算法）；

第三种是针对多内存堆的分配情况（memheap 管理算法）

## 内存堆管理

内存堆管理用于管理一段连续的内存空间，在参考文档中 [《内核基础》](https://www.rt-thread.org/document/site/#/rt-thread-version/rt-thread-standard/programming-manual/basic/basic) 章节有介绍过 RT-Thread 的内存分布情况，

如下图所示，RT-Thread 将 **“ZI 段结尾处” 到内存尾部的空间**用作内存堆。

![RT-Thread 内存分布](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/memory/figures/08Memory_distribution.png)

内存堆可以在当前资源满足的情况下，根据用户的需求分配任意大小的内存块。

而当用户不需要再使用这些内存块时，又可以释放回堆中供其他应用分配使用。

RT-Thread 系统为了满足不同的需求，提供了不同的内存管理算法，分别是小内存管理算法、slab 管理算法和 memheap 管理算法。

小内存管理算法主要针对系统资源比较少，一般用于**小于 2MB** 内存空间的系统；

而 slab 内存管理算法则主要是在系统资源比较丰富时，提供了一种**近似多内存池管理算法**的快速算法。

除上述之外，RT-Thread 还有一种针对多内存堆的管理算法，即 memheap 管理算法。memheap 方法适用于系统存在多个内存堆的情况，它可以**将多个内存 “粘贴” 在一起，形成一个大的内存堆**，用户使用起来会非常方便。

这几类内存堆管理算法在系统运行时只能选择其中之一或者完全不使用内存堆管理器，他们提供给应用程序的 API 接口完全相同。



注：因为内存堆管理器要满足多线程情况下的安全分配，会考虑多线程间的互斥问题，所以请不要在中断服务例程中分配或释放动态内存块。因为它可能会引起当前上下文被挂起等待。



### 小内存管理算法

小内存管理算法是一个简单的内存分配算法。

初始时，它是一块大的内存。当需要分配内存块时，将从这个大的内存块上分割出相匹配的内存块，然后把分割出来的空闲内存块还回给堆管理系统中。每个内存块都包含一个管理用的数据头，通过这个头把使用块与空闲块用双向链表的方式链接起来。

**在4.1.0之前使用此实现**

如下图所示：

![小内存管理工作机制图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/memory/figures/08smem_work.png)

每个内存块（不管是已分配的内存块还是空闲的内存块）都包含一个数据头，其中包括：

**1）magic**：变数（或称为幻数），它会被初始化成 **0x1ea0**（即英文单词 heap），用于标记这个内存块是一个内存管理用的内存数据块；变数不仅仅用于标识这个数据块是一个内存管理用的内存数据块，实质也是一个**内存保护字**：如果这个区域被改写，那么也就意味着这块内存块被非法改写（正常情况下只有内存管理器才会去碰这块内存）。

**2）used**：指示出当前内存块是否已经分配。

内存管理的表现主要体现在**内存的分配与释放**上，小型内存管理算法可以用以下例子体现出来。

**4.1.0及以后版本使用此实现**

如下图所示：

![img](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/memory/figures/08smem_work4.png)

**堆开始**:堆起始地址存储**内存使用信息**，**堆起始地址指针**，**堆结束地址指针**，**最小堆空闲地址指针和内存大小**。

无论是已分配的**内存块**还是空闲的内存块都包含一个数据头，包括:

**pool_ptr**:**小内存对象地址**。如果内存块的最后一位为**1**，则标记为使用。如果内存块的最后一位为**0**，则标记为未使用。可以通过该地址计算快速获得小内存算法结构体成员。

如下图所示的内存分配情况，空闲链表指针 lfree 初始指向 32 字节的内存块。

当用户线程要再分配一个 64 字节的内存块时，但此 lfree 指针指向的内存块只有 32 字节并不能满足要求，内存管理器会继续寻找下一内存块，当找到再下一块内存块，128 字节时，它满足分配的要求。因为这个内存块比较大，分配器将把此内存块进行拆分，余下的内存块（52 字节）继续留在 lfree 链表中，如下图分配 64 字节后的链表结构所示。

![小内存管理算法链表结构示意图 1](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/memory/figures/08smem_work2.png)

![小内存管理算法链表结构示意图 2](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/memory/figures/08smem_work3.png)

另外，在每次分配内存块前，都会**留出 12 字节**数据头用于 magic、used 信息及链表节点使用。返回给应用的地址实际上是**这块内存块 12 字节以后的地址**，前面的 12 字节数据头是用户永远不应该碰的部分（注：12 字节数据头长度会与系统对齐差异而有所不同）。

释放时则是相反的过程，但分配器会查看前后相邻的内存块是否空闲，如果空闲则**合并成一个大的空闲内存块**。

### slab 管理算法

RT-Thread 的 slab 分配器是在 DragonFly BSD 创始人 Matthew Dillon 实现的 slab 分配器基础上，针对嵌入式系统优化的内存分配算法。最原始的 slab 算法是 Jeff Bonwick 为 Solaris 操作系统而引入的一种高效内核内存分配算法。

RT-Thread 的 slab 分配器实现主要是去掉了其中的对象构造及析构过程，只保留了**纯粹的缓冲型**的内存池算法。

slab 分配器会根据对象的大小分成多个区（zone），也可以看成每类对象有一个内存池，如下图所示：

![slab 内存分配结构图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/memory/figures/08slab.png)

一个 zone 的大小在 32K 到 128K 字节之间，分配器会在堆初始化时根据堆的大小自动调整。

系统中的 zone 最多包括 **72 种对象**，一次最大能够分配 **16K** 的内存空间，如果超出了 16K 那么直接从页分配器中分配。

每个 zone 上分配的内存块大小是固定的，能够分配相同大小内存块的 zone 会链接在一个链表中，而 72 种对象的 zone 链表则放在一个数组（zone_array[]）中统一管理。

下面是内存分配器主要的两种操作：

**（1）内存分配**

假设分配一个 32 字节的内存，slab 内存分配器会先按照 32 字节的值，从 zone array 链表表头数组中找到相应的 zone 链表。如果这个链表是空的，则向页分配器分配一个新的 zone，然后从 zone 中返回第一个空闲内存块。如果链表非空，则这个 zone 链表中的第一个 zone 节点必然有空闲块存在（否则它就不应该放在这个链表中），那么就取相应的空闲块。

如果分配完成后，zone 中所有空闲内存块都使用完毕，那么分配器需要把这个 zone 节点从链表中删除。

**（2）内存释放**

分配器需要找到内存块所在的 zone 节点，然后把内存块链接到 zone 的空闲内存块链表中。如果此时 zone 的空闲链表指示出 zone 的所有内存块都已经释放，即 zone 是完全空闲的，那么当 zone 链表中全空闲 zone 达到一定数目后，系统就会把**这个全空闲的 zone** 释放到页面分配器中去。

### memheap 管理算法

memheap 管理算法适用于**系统含有多个地址可不连续的内存堆**。

使用 memheap 内存管理可以简化系统存在多个内存堆时的使用：当系统中存在多个内存堆的时候，用户只需要在系统初始化时将多个所需的 memheap 初始化，并开启 memheap 功能就可以很方便地把多个 memheap（地址可不连续）粘合起来用于系统的 heap 分配。



注：在开启 memheap 之后原来的 heap 功能将被关闭，两者只可以通过打开或关闭 RT_USING_MEMHEAP_AS_HEAP 来选择其一

memheap 工作机制如下图所示，首先将多块内存加入 memheap_item 链表进行粘合。当分配内存块时，会先从默认内存堆去分配内存，**当分配不到时会查找 memheap_item 链表，尝试从其他的内存堆上分配内存块**。

应用程序不用关心当前分配的内存块位于哪个内存堆上，就像是在操作一个内存堆。

![memheap 处理多内存堆](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/memory/figures/08memheap.png)

### 内存堆配置和初始化

在使用内存堆时，必须要在系统初始化的时候进行堆的初始化，可以通过下面的函数接口完成：

```c
void rt_system_heap_init(void* begin_addr, void* end_addr);
```

这个函数会把参数 begin_addr，end_addr 区域的内存空间作为内存堆来使用。

下表描述了该函数的输入参数：

rt_system_heap_init() 的输入参数

| **参数**   | **描述**           |
| ---------- | ------------------ |
| begin_addr | 堆内存区域起始地址 |
| end_addr   | 堆内存区域结束地址 |

在使用 memheap 堆内存时，必须要在系统初始化的时候进行堆内存的初始化，可以通过下面的函数接口完成：

```c
rt_err_t rt_memheap_init(struct rt_memheap  *memheap,
                        const char  *name,
                        void        *start_addr,
                        rt_uint32_t size)
```

如果有多个不连续的 memheap 可以多次调用该函数将其初始化并加入 memheap_item 链表。

下表描述了rt_memheap_init() 的输入参数与返回值

| **参数**   | **描述**           |
| ---------- | ------------------ |
| memheap    | memheap 控制块     |
| name       | 内存堆的名称       |
| start_addr | 堆内存区域起始地址 |
| size       | 堆内存大小         |
| **返回**   | ——                 |
| RT_EOK     | 成功               |

### 内存堆的管理方式

对内存堆的操作如下图所示，包含：**初始化**、**申请内存块**、**释放内存**，所有使用完成后的动态内存都应该被释放，以供其他程序的申请使用。

![内存堆的操作](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/memory/figures/08heap_ops.png)

#### 分配和释放内存块

从内存堆上分配用户指定大小的内存块，函数接口如下：

```c
void *rt_malloc(rt_size_t nbytes);
```

rt_malloc 函数会从系统堆空间中找到合适大小的内存块，然后把内存块可用地址返回给用户。

下表描述了rt_malloc() 的输入参数和返回值：

| **参数**         | **描述**                                          |
| ---------------- | ------------------------------------------------- |
| nbytes           | 需要分配的内存块的大小，单位为字节 （1B = 8 bit） |
| **返回**         | ——                                                |
| 分配的内存块地址 | 成功                                              |
| RT_NULL          | 失败                                              |

对 rt_malloc 的返回值进行判空是非常有必要的。应用程序使用完从内存分配器中申请的内存后，必须**及时释放**，否则会造成内存泄漏。

##### 使用动态分配内存示例

```c
int *pi;
pi = rt_malloc(100);
if(pi == NULL) 
{
    rt_kprintf("malloc failed\r\n");
}
```



释放内存块的函数接口如下：

```c
void rt_free (void *ptr);
```

rt_free 函数会把待释放的内存还回给堆管理器中。在调用这个函数时用户需传递待释放的内存块指针，如果是空指针直接返回。

下表rt_free() 的输入参数

| **参数** | **描述**           |
| -------- | ------------------ |
| ptr      | 待释放的内存块指针 |

```c
#include "rtthread.h"
/*
使用实例:

rt_free 的参数必须是以下其一：

- 是 NULL 。
- 是一个先前从 rt_malloc、 rt_calloc 或 rt_realloc 返回的值。

*/

int main()
{

    /*
    进行对ptr赋值,以null值为例子
    */
    int *ptr = RT_NULL;
    /* 释放内存块指针 */
    rt_free(ptr);
}
```



#### 重分配内存块

在已分配内存块的基础上重新分配内存块的大小（增加或缩小），可以通过下面的函数接口完成：

```c
void *rt_realloc(void *rmem, rt_size_t newsize);
```

在进行重新分配内存块时，原来的内存块数据保持不变（缩小的情况下，后面的数据被自动截断）。

下表描述了rt_realloc() 的输入参数和返回值：

| **参数**             | **描述**           |
| -------------------- | ------------------ |
| rmem                 | 指向已分配的内存块 |
| newsize              | 重新分配的内存大小 |
| **返回**             | ——                 |
| 重新分配的内存块地址 | 成功               |



- rt_realloc 函数用于**修改**一个原先已经分配的内存块的大小。

  使用这个函数，你可以使一块内存扩大或者缩小。如果它用于扩大一个内存块，那么这块内存原先的内容依然保留，新增加的内存**添加**到**原先**内存块的**后面**，新内存并未以任何方式进行初始化。如果它用于缩小一个内存块，该内存尾部的部分内存便被拿掉，剩余部分内存的原先内容依然保留。

- 如果原先的内存块无法改变大小，rt_realloc 将分配另一块正确大小的内存，并把原先那块内存的内容**复制到新的块上**。因此，在使用 rt_realloc 之后，你就不能再使用指向旧内存块的指针，而是应该改用 rt_realloc 所返回的新指针。

- 如果 rt_realloc 的**第一个参数为 NULL, 那么它的行为就和 rt_malloc 一样**。



##### 使用重新分配内存的示例

```c
#include <rtthread.h>

#define THREAD_PRIORITY      25
#define THREAD_STACK_SIZE    512
#define THREAD_TIMESLICE     5

int exsample_realloc(void)
{
    /* 初始分配的元素数量和每个元素的大小 */
    rt_size_t initial_count = 5;
    rt_size_t new_count = 10;
    rt_size_t size = sizeof(int);

    /* 分配初始内存 */
    void *ptr = rt_malloc(initial_count * size);
    if (ptr == RT_NULL) {
        rt_kprintf("Initial memory allocation failed.\n");
        return -1;
    }

    /* 将指针强制转换为 int 类型指针 */
    int *intArray = (int *)ptr;

    /* 使用初始分配的内存 */
    for (int i = 0; i < initial_count; i++) {
        intArray[i] = i * 10;
    }

    /* 打印初始分配的内存内容 */
    rt_kprintf("Initial allocated memory contents:\n");
    for (int i = 0; i < initial_count; i++) {
        rt_kprintf("%d ", intArray[i]);
    }
    rt_kprintf("\n");

    /* 重新分配内存 */
    ptr = rt_realloc(ptr, new_count * size);
    if (ptr == RT_NULL) {
        rt_kprintf("Re-allocation failed.\n");
        rt_free(intArray);
        return -1;
    }

    /* 更新指针 */
    intArray = (int *)ptr;

    /* 使用重新分配后的内存 */
    for (int i = initial_count; i < new_count; i++) {
        intArray[i] = i * 10;
    }

    /* 打印重新分配后的内存内容 */
    rt_kprintf("Re-allocated memory contents:\n");
    for (int i = 0; i < new_count; i++) {
        rt_kprintf("%d ", intArray[i]);
    }
    rt_kprintf("\n");

    /* 释放内存 */
    rt_free(ptr);

    return 0;
}


int rt_realloc_sample(void)
{
    rt_thread_t tid = RT_NULL;

    /* 创建线程 1 */
    tid = rt_thread_create("thread1",
                           exsample_realloc, RT_NULL,
                           THREAD_STACK_SIZE,
                           THREAD_PRIORITY,
                           THREAD_TIMESLICE);
    if (tid != RT_NULL)
        rt_thread_startup(tid);

    return 0;
}

/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(rt_realloc_sample, rt_readlloc_sample);
```

#### 分配多内存块

从内存堆中分配连续内存地址的多个内存块，可以通过下面的函数接口完成：

```c
  void *rt_calloc(rt_size_t count, rt_size_t size);
```

下表描述了rt_calloc() 的输入参数和返回值：

| **参数**                   | **描述**                                    |
| -------------------------- | ------------------------------------------- |
| count                      | 内存块数量                                  |
| size                       | 内存块容量                                  |
| **返回**                   | ——                                          |
| 指向第一个内存块地址的指针 | 成功 ，并且所有分配的内存块都被初始化成零。 |
| RT_NULL                    | 分配失败                                    |



##### 使用分配多内存块示例

```c
#include <rtthread.h>

#define THREAD_PRIORITY      25
#define THREAD_STACK_SIZE    512
#define THREAD_TIMESLICE     5

int exsample_calloc(void)
{
    /* 定义要分配的元素数量和每个元素的大小 */
    rt_size_t count = 5;
    rt_size_t size = sizeof(int);

    /* 分配内存 */
    void *ptr = rt_calloc(count, size);
    if (ptr == RT_NULL) {
        rt_kprintf("Memory allocation failed.\n");
        return -1;
    }

    /* 将指针强制转换为 int 类型指针 */
    int *intArray = (int *)ptr;

    /* 使用分配的内存 */
    for (int i = 0; i < count; i++) {
        intArray[i] = i * 10;
    }

    /* 打印分配的内存内容 */
    rt_kprintf("Allocated memory contents:\n");
    for (int i = 0; i < count; i++) {
        rt_kprintf("%d ", intArray[i]);
    }
    rt_kprintf("\n");

    /* 释放内存 */
    rt_free(ptr);

    return 0;
}

int rt_calloc_sample(void)
{
    rt_thread_t tid = RT_NULL;

    /* 创建线程 1 */
    tid = rt_thread_create("thread1",
                            exsample_calloc, RT_NULL,
                           THREAD_STACK_SIZE,
                           THREAD_PRIORITY,
                           THREAD_TIMESLICE);
    if (tid != RT_NULL)
        rt_thread_startup(tid);

    return 0;
}
/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(rt_calloc_sample, rt_calloc sample);
```

#### 设置内存钩子函数

在分配内存块过程中，用户可设置一个钩子函数，调用的函数接口如下：

```c
void rt_malloc_sethook(void (*hook)(void *ptr, rt_size_t size));
```

设置的钩子函数会在内存**分配完成后**进行回调。回调时，会把分配到的内存块地址和大小做为入口参数传递进去。

下表描述了rt_malloc_sethook() 的输入参数：

| **参数** | **描述**     |
| -------- | ------------ |
| hook     | 钩子函数指针 |

其中 hook 函数接口如下：

```c
void hook(void *ptr, rt_size_t size)；
```

下表描述了 hook 函数的输入参数：

分配钩子 hook 函数接口参数

| **参数** | **描述**             |
| -------- | -------------------- |
| ptr      | 分配到的内存块指针   |
| size     | 分配到的内存块的大小 |

在释放内存时，用户可设置一个钩子函数，调用的函数接口如下：

```c
void rt_free_sethook(void (*hook)(void *ptr));
```

设置的钩子函数会在调用内存释放完成前进行回调。

回调时，释放的内存块地址会做为入口参数传递进去（此时内存块并**没有被释放**）。

下表描述了rt_free_sethook() 的输入参数：

| **参数** | **描述**     |
| -------- | ------------ |
| hook     | 钩子函数指针 |

其中 hook 函数接口如下：

```c
void hook(void *ptr);
```

下表描述了 hook 函数的输入参数：

钩子函数 hook 的输入参数

| **参数** | **描述**           |
| -------- | ------------------ |
| ptr      | 待释放的内存块指针 |

##### 内存钩子函数示例

```c
#include <rtthread.h>

#define THREAD_PRIORITY      25
#define THREAD_STACK_SIZE    512
#define THREAD_TIMESLICE     5

/* 定义内存钩子函数 */
void malloc_hook(void *ptr, rt_size_t size)
{
    rt_kprintf("malloc: %p, size: %ld\n", ptr, size);
}

void free_hook(void *ptr)
{
    rt_kprintf("free: %p\n", ptr);
}

/* 注册内存钩子函数 */
void register_memory_hooks()
{
    rt_malloc_sethook(malloc_hook);
    rt_free_sethook(free_hook);
}

/* 初始分配的元素数量和每个元素的大小 */
rt_size_t initial_count = 5;
rt_size_t new_count = 10;
rt_size_t size = sizeof(int);

/* 内存操作函数 */
int exsample_realloc(void)
{
    /* 分配初始内存 */
    void *ptr = rt_malloc(initial_count * size);
    if (ptr == RT_NULL) {
        rt_kprintf("Initial memory allocation failed.\n");
        return -1;
    }

    /* 将指针强制转换为 int 类型指针 */
    int *intArray = (int *)ptr;

    /* 使用初始分配的内存 */
    for (int i = 0; i < initial_count; i++) {
        intArray[i] = i * 10;
    }

    /* 打印初始分配的内存内容 */
    rt_kprintf("Initial allocated memory contents:\n");
    for (int i = 0; i < initial_count; i++) {
        rt_kprintf("%d ", intArray[i]);
    }
    rt_kprintf("\n");

    /* 重新分配内存 */
    ptr = rt_realloc(ptr, new_count * size);
    if (ptr == RT_NULL) {
        rt_kprintf("Re-allocation failed.\n");
        rt_free(intArray); 
        return -1;
    }

    /* 更新指针 */
    intArray = (int *)ptr;

    /* 使用重新分配后的内存 */
    for (int i = initial_count; i < new_count; i++) {
        /* 给新扩展的元素赋值 */
        intArray[i] = i * 10; 
    }

    /* 打印重新分配后的内存内容 */
    rt_kprintf("Re-allocated memory contents:\n");
    for (int i = 0; i < new_count; i++) {
        rt_kprintf("%d ", intArray[i]);
    }
    rt_kprintf("\n");

    /* 释放内存 */
    rt_free(ptr);

    return 0;
}

int rt_memory_hook_sample(void)
{
    rt_thread_t tid = RT_NULL;

    /* 注册内存钩子函数 */
    register_memory_hooks();

    /* 创建线程 */
    tid = rt_thread_create("thread1",
                           exsample_realloc, RT_NULL,
                           THREAD_STACK_SIZE,
                           THREAD_PRIORITY,
                           THREAD_TIMESLICE);
    if (tid != RT_NULL)
        rt_thread_startup(tid);

    return 0;
}

/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(rt_memory_hook_sample, rt_memory_hook_sample);
```

### 总结

动态内存使用总结：

- 检查从 rt_malloc 函数返回的指针是否为 **NULL**
- 不要访问动态分配内存**之外**的内存
- 不要向 rt_free 传递一个**并非**由 rt_malloc 函数返回的指针
- 在释放动态内存之后**不要**再访问它
- 使用 **sizeof** 计算数据类型的长度，提高程序的可移植性

常见的动态内存错误：

- 对 NULL 指针进行解引用
- 对分配的内存进行操作时**越过边界**
- 释放**并非动态分配**的内存
- 释放一块动态分配的内存的**一部分** (rt_free(ptr + 4))
- 动态内存**被释放后继续使用**

内存碎片：频繁的调用内存分配和释放接口会导致**内存碎片**，一个避免内存碎片的策略是使用 `内存池 + 内存堆` **混用**的方法。

### 内存堆管理应用示例

这是一个内存堆的应用示例，这个程序会创建一个动态的线程，这个线程会动态申请内存并释放，每次申请更大的内存，当申请不到的时候就结束，如下代码所示：

内存堆管理

```c
#include <rtthread.h>

#define THREAD_PRIORITY      25
#define THREAD_STACK_SIZE    512
#define THREAD_TIMESLICE     5

/* 线程入口 */
void thread1_entry(void *parameter)
{
    int i;
    /* 内存块的指针 */
    char *ptr = RT_NULL; 
    /* 内存块的指针 */

    for (i = 0; ; i++)
    {
        /* 每次分配 (1 << i) 大小字节数的内存空间 */
        ptr = rt_malloc(1 << i);

        /* 如果分配成功 */
        if (ptr != RT_NULL)
        {
            /*
            *此处是以1往左移位，因此，内存块大小为2的指数次方
            *2进制       10进制
            *0001        1
            *0010        2
            *0100        4
            *1000        8
            *.....
            *1 << n      2^n
            */
            rt_kprintf("get memory :%d byte\n", (1 << i));

            /* 释放内存块，及时释放内存块，避免导致内存泄漏 */
            rt_free(ptr);
            rt_kprintf("free memory :%d byte\n", (1 << i));

            /* 将指针置空 */
            ptr = RT_NULL;
        }
        else
        {
            /* 若申请内存失败则进行退出，并告知目前无法申请的内存大小 */
            rt_kprintf("try to get %d byte memory failed!\n", (1 << i));
            return;
        }
    }
}

int dynmem_sample(void)
{
    rt_thread_t tid = RT_NULL;

    /* 创建线程 1 */
    tid = rt_thread_create("thread1",
                           thread1_entry, RT_NULL,
                           THREAD_STACK_SIZE,
                           THREAD_PRIORITY,
                           THREAD_TIMESLICE);
    if (tid != RT_NULL)
        rt_thread_startup(tid);

    return 0;
}
/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(dynmem_sample, dynmem sample);
```



仿真运行结果如下：

```
\ | /
- RT -     Thread Operating System
 / | \     3.1.0 build Aug 24 2018
 2006 - 2018 Copyright by rt-thread team
msh >dynmem_sample
msh >get memory :1 byte
free memory :1 byte
get memory :2 byte
free memory :2 byte
…
get memory :16384 byte
free memory :16384 byte
get memory :32768 byte
free memory :32768 byte
try to get 65536 byte memory failed!
```

例程中分配内存成功并打印信息；

当试图申请 65536 byte **即 64KB 内存**时，由于 RAM 总大小**只有 64K**，而**可用 RAM 小于 64K**，所以分配**失败**。



# 中断管理

什么是中断？简单的解释就是系统正在处理某一个正常事件，忽然被另一个需要马上处理的紧急事件打断，系统转而处理这个紧急事件，待处理完毕，再恢复运行刚才被打断的事件。

生活中，我们经常会遇到这样的场景：

当你正在专心看书的时候，忽然来了一个电话，于是记下书的页码，去接电话，接完电话后接着刚才的页码继续看书，这是一个典型的中断的过程。

电话是老师打过来的，让你赶快交作业，你判断交作业的优先级比看书高，于是电话挂断后先做作业，等交完作业后再接着刚才的页码继续看书，这是一个典型的**在中断中进行任务调度**的过程。

这些场景在嵌入式系统中也很常见，当 CPU 正在处理内部数据时，外界发生了紧急情况，要求 CPU 暂停当前的工作转去处理这个 **异步事件**。

处理完毕后，再回到原来被中断的地址，继续原来的工作，这样的过程称为**中断**。实现这一功能的系统称为 **中断系统**，申请 CPU 中断的请求源称为 **中断源**。

中断是一种异常，异常是导致处理器脱离正常运行转向执行特殊代码的任何事件，如果不及时进行处理，轻则系统出错，重则会导致系统毁灭性地瘫痪。所以正确地处理异常，避免错误的发生是提高软件鲁棒性（稳定性）非常重要的一环。

如下图是一个简单的中断示意图。

![中断示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/interrupt/figures/09interrupt_work.png)

中断处理与 CPU 架构密切相关，所以，本章会先介绍 ARM Cortex-M 的 CPU 架构，然后结合 Cortex-M CPU 架构来介绍 RT-Thread 的中断管理机制，读完本章，大家将深入了解 RT-Thread 的中断处理过程，如何添加中断服务程序（ISR）以及相关的注意事项。

## Cortex-M CPU 架构基础

不同于老的经典 ARM 处理器（例如：ARM7, ARM9），ARM Cortex-M 处理器有一个非常不同的架构，Cortex-M 是一个家族系列，其中包括 Cortex M0/M3/M4/M7 多个不同型号，每个型号之间会有些区别，例如 Cortex-M4 比 Cortex-M3 多了浮点计算功能等，但它们的编程模型基本是一致的，因此本介绍中断管理和移植的部分都不会对 Cortex M0/M3/M4/M7 做太精细的区分。

本节主要介绍和 RT-Thread 中断管理相关的架构部分。

### 寄存器简介 

Cortex-M 系列 CPU 的寄存器组里有 R0~R15 共 16 个通用寄存器组和若干特殊功能寄存器，如下图所示。

通用寄存器组里的 **R13 作为堆栈指针寄存器** (Stack Pointer，SP)；

**R14 作为连接寄存器** (Link Register，LR)，用于在调用子程序时，**存储返回地址**；

**R15 作为程序计数器** (Program Counter，PC)，其中堆栈指针寄存器可以是**主堆**栈指针（MSP），也可以是**进程堆**栈指针（PSP）。

![Cortex-M 寄存器示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/interrupt/figures/09interrupt_table.png)

特殊功能寄存器包括**程序状态字寄存器组**（PSRs）、**中断屏蔽寄存器组**（PRIMASK, FAULTMASK, BASEPRI）、**控制寄存器**（CONTROL），可以通过 **MSR/MRS 指令**来访问特殊功能寄存器，例如：

```
MRS R0, CONTROL ; 读取 CONTROL 到 R0 中
MSR CONTROL, R0 ; 写入 R0 到 CONTROL 寄存器中
```

程序状态字寄存器里**保存算术与逻辑标志，例如负数标志，零结果标志，溢出标志等等**。

中断屏蔽寄存器组**控制 Cortex-M 的中断使能**。

控制寄存器用来**定义特权级别**和**当前使用哪个堆栈指针**。

如果是具有浮点单元的 Cortex-M4 或者 Cortex-M7，控制寄存器**也用来指示浮点单元当前是否在使用**，浮点单元包含了 **32 个浮点通用寄存器 S0~S31** 和**特殊 FPSCR 寄存器**（Floating point status and control register）。

### 操作模式和特权级别

Cortex-M 引入了操作模式和特权级别的概念，分别为线程模式和处理模式，如果**进入异常或中断处理则进入处理模式**，其他情况则为线程模式。

![Cortex-M 工作模式状态图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/interrupt/figures/09interrupt_work_sta.png)

Cortex-M 有两个运行级别，分别为**特权级**和**用户级**，线程模式可以工作在特权级或者用户级，而处理模式总工作在特权级，可通过 CONTROL 特殊寄存器控制。工作模式状态切换情况如上图所示。

Cortex-M 的堆栈寄存器 SP 对应两个物理寄存器 MSP 和 PSP，MSP 为主堆栈，PSP 为进程堆栈，处理模式总是使用 MSP 作为堆栈，线程模式可以选择使用 MSP 或 PSP 作为堆栈，同样通过 CONTROL 特殊寄存器控制。复位后，Cortex-M 默认进入线程模式、特权级、使用 MSP 堆栈。

### 嵌套向量中断控制器

Cortex-M 中断控制器名为 NVIC（**嵌套向量中断控制器**），支持**中断嵌套**功能。当一个中断触发并且系统进行响应时，处理器硬件会将当前运行位置的上下文寄存器自动压入中断栈中，这部分的寄存器包括 PSR、PC、LR、R12、R3-R0 寄存器。

![Cortex-M 内核和 NVIC 关系示意图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/interrupt/figures/09relation.png)

当系统正在服务一个中断时，如果有一个更高优先级的中断触发，那么处理器同样会打断当前运行的中断服务程序，然后把这个中断服务程序上下文的 PSR、PC、LR、R12、R3-R0 寄存器自动保存到中断栈中。



### PendSV 系统调用

PendSV 也称为可悬起的系统调用，它是一种异常，可以像普通的中断一样被挂起，它是专门用来辅助操作系统进行上下文切换的。PendSV 异常会被初始化为最低优先级的异常。每次需要进行上下文切换的时候，会手动触发 PendSV 异常，在 PendSV 异常处理函数中进行上下文切换。

在下一章《内核移植》中会详细介绍利用 PendSV 机制进行操作系统上下文切换的详细流程。





## RT-Thread 中断工作机制

### 中断向量表

**中断向量表**是所有中断处理程序的入口，如下图所示是 Cortex-M 系列的中断处理过程：把一个函数（用户中断服务程序）同一个虚拟中断向量表中的中断向量联系在一起。当中断向量对应中断发生的时候，被挂接的用户中断服务程序就会被调用执行。

![中断处理过程](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/interrupt/figures/09interrupt_handle.png)

在 Cortex-M 内核上，所有中断都采用中断向量表的方式进行处理，即当一个中断触发时，处理器将直接判定是哪个中断源，然后直接跳转到相应的固定位置进行处理，每个中断服务程序必须排列在一起放在统一的地址上（这个地址必须要设置到 NVIC 的中断向量偏移寄存器中）。中断向量表一般由**一个数组定义或在起始代码中给出**，默认采用起始代码给出：

```c
  __Vectors     DCD     __initial_sp             ; Top of Stack
                DCD     Reset_Handler            ; Reset 处理函数
                DCD     NMI_Handler              ; NMI 处理函数
                DCD     HardFault_Handler        ; Hard Fault 处理函数
                DCD     MemManage_Handler        ; MPU Fault 处理函数
                DCD     BusFault_Handler         ; Bus Fault 处理函数
                DCD     UsageFault_Handler       ; Usage Fault 处理函数
                DCD     0                        ; 保留
                DCD     0                        ; 保留
                DCD     0                        ; 保留
                DCD     0                        ; 保留
                DCD     SVC_Handler              ; SVCall 处理函数
                DCD     DebugMon_Handler         ; Debug Monitor 处理函数
                DCD     0                        ; 保留
                DCD     PendSV_Handler           ; PendSV 处理函数
                DCD     SysTick_Handler          ; SysTick 处理函数

… …

NMI_Handler             PROC
                EXPORT NMI_Handler              [WEAK]
                B       .
                ENDP
HardFault_Handler       PROC
                EXPORT HardFault_Handler        [WEAK]
                B       .
                ENDP
… …
```

请注意代码后面的 [WEAK] 标识，它是**符号弱化标识**，在 [WEAK] 前面的符号(如 NMI_Handler、HardFault_Handler）将被执行**弱化处理**，如果整个代码在链接时遇到了名称相同的符号（例如与 NMI_Handler 相同名称的函数），那么代码将使用未被弱化定义的符号（与 NMI_Handler 相同名称的函数），而**与弱化符号相关的代码将被自动丢弃**。

以 **SysTick 中断**为例，在系统启动代码中，需要填上 SysTick_Handler 中断入口函数，然后实现该函数即可对 SysTick 中断进行响应，中断处理函数示例程序如下所示：

```c
void SysTick_Handler(void)
{
    /* enter interrupt */
    rt_interrupt_enter();

    rt_tick_increase();

    /* leave interrupt */
    rt_interrupt_leave();
}
```

### 中断处理过程

RT-Thread 中断管理中，将中断处理程序分为**中断前导程序**、**用户终端服务程序**、**中断后续程序**三部分，如下图：

![中断处理程序的 3 部分](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/interrupt/figures/09interrupt_work_process.png)

#### 中断前导程序

中断前导程序主要工作如下：

1）**保存 CPU 中断现场**，这部分跟 CPU 架构相关，不同 CPU 架构的实现方式有差异。

对于 Cortex-M 来说，该工作由硬件自动完成。

当一个中断触发并且系统进行响应时，处理器硬件会将当前运行部分的上下文寄存器自动压入中断栈中，这部分的寄存器包括 PSR、PC、LR、R12、R3-R0 寄存器。

2）**通知内核进入中断状态**，调用 rt_interrupt_enter() 函数，作用是把全局变量 rt_interrupt_nest 加 1，用它来**记录中断嵌套的层数**，代码如下所示。

```c
void rt_interrupt_enter(void)
{
    rt_base_t level;

    level = rt_hw_interrupt_disable();
    rt_interrupt_nest ++;
    rt_hw_interrupt_enable(level);
}
```

#### 用户中断服务程序

在用户中断服务程序（ISR）中，分为两种情况，第一种情况是不进行线程切换，这种情况下用户中断服务程序和中断后续程序运行完毕后退出中断模式，返回被中断的线程。

另一种情况是，在中断处理过程中需要进行线程切换，这种情况会调用 **rt_hw_context_switch_interrupt() 函数**进行上下文切换，该函数跟 CPU 架构相关，不同 CPU 架构的实现方式有差异。

在 Cortex-M 架构中，rt_hw_context_switch_interrupt() 的函数实现流程如下图所示，它将设置需要切换的线程 rt_interrupt_to_thread 变量，然后触发 PendSV 异常（PendSV 异常是专门用来辅助上下文切换的，且被初始化为最低优先级的异常）。PendSV 异常被触发后，不会立即进行 PendSV 异常中断处理程序，因为此时还在中断处理中，只有当中断后续程序运行完毕，真正退出中断处理后，才进入 PendSV 异常中断处理程序。

![rt_hw_context_switch_interrupt() 函数实现流程](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/interrupt/figures/09fun1.png)

#### 中断后续程序

中断后续程序主要完成的工作是:

1 **通知内核离开中断状态**，通过调用 rt_interrupt_leave() 函数，将全局变量 rt_interrupt_nest 减 1，代码如下所示。

```c
void rt_interrupt_leave(void)
{
    rt_base_t level;

    level = rt_hw_interrupt_disable();
    rt_interrupt_nest --;
    rt_hw_interrupt_enable(level);
}
```

2 **恢复中断前的 CPU 上下文**，如果在中断处理过程中未进行线程切换，那么恢复 from 线程的 CPU 上下文，如果在中断中进行了线程切换，那么恢复 to 线程的 CPU 上下文。这部分实现跟 CPU 架构相关，不同 CPU 架构的实现方式有差异，在 Cortex-M 架构中实现流程如下图所示。

![rt_hw_context_switch_interrupt() 函数实现流程](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/interrupt/figures/09fun2.png)



### 中断嵌套

在允许中断嵌套的情况下，在执行中断服务程序的过程中，如果出现高优先级的中断，当前中断服务程序的执行将被打断，以执行高优先级中断的中断服务程序，当高优先级中断的处理完成后，被打断的中断服务程序才又得到继续执行，如果需要进行线程调度，线程的上下文切换将在所有中断处理程序都运行结束时才发生，如下图所示。

![中断中的线程切换](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/interrupt/figures/09ths_switch.png)



### 中断栈

在中断处理过程中，在系统响应中断前，软件代码（或处理器）需要把当前线程的上下文保存下来（通常保存在当前线程的线程栈中），再调用中断服务程序进行中断响应、处理。

在进行中断处理时（实质是调用用户的中断服务程序函数），中断处理函数中很可能会有自己的局部变量，这些都需要相应的栈空间来保存，所以中断响应依然需要一个栈空间来做为上下文，运行中断处理函数。



中断栈可以保存在打断线程的栈中，当从中断中退出时，返回相应的线程继续执行。

中断栈也可以与线程栈完全分离开来，即每次进入中断时，在保存完打断线程上下文后，切换到新的中断栈中独立运行。在中断退出时，再做相应的上下文恢复。



使用独立中断栈相对来说更容易实现，并且对于线程栈使用情况也比较容易了解和掌握（否则必须要为中断栈预留空间，如果系统支持中断嵌套，还需要考虑应该为嵌套中断预留多大的空间）。



RT-Thread 采用的方式是**提供独立的中断栈**，即中断发生时，中断的前期处理程序会将用户的栈指针**更换到系统事先留出的中断栈空间中**，等中断退出时**再恢复**用户的栈指针。这样中断就不会占用线程的栈空间，从而提高了内存空间的利用率，且随着线程的增加，这种减少内存占用的效果也越明显。

在 Cortex-M 处理器内核里有两个堆栈指针，一个是**主堆栈指针（MSP），是默认的堆栈指针**，在运行第一个线程之前和在中断和异常服务程序里使用；另一个是**线程堆栈指针（PSP），在线程里使用**。在中断和异常服务程序退出时，**修改 LR 寄存器的第 2 位的值为 1**，线程的 SP 就由 MSP 切换到 PSP。



### 中断的底半处理

RT-Thread 不对中断服务程序所需要的处理时间做任何假设、限制，但如同其他实时操作系统或非实时操作系统一样，用户需要保证所有的中断服务程序在尽可能短的时间内完成（中断服务程序在系统中相当于拥有最高的优先级，会抢占所有线程优先执行）。这样在发生中断嵌套，或屏蔽了相应中断源的过程中，不会耽误嵌套的其他中断处理过程，或自身中断源的下一次中断信号。

当一个中断发生时，中断服务程序需要取得相应的硬件状态或者数据。如果中断服务程序接下来要对状态或者数据进行简单处理，比如 CPU 时钟中断，中断服务程序只需对一个系统时钟变量进行加一操作，然后就结束中断服务程序。这类中断需要的运行时间往往都比较短。但对于另外一些中断，中断服务程序在取得硬件状态或数据以后，还需要进行一系列更耗时的处理过程，通常需要将该中断分割为两部分，即**上半部分**（Top Half）和**底半部分**（Bottom Half）。在上半部分中，取得硬件状态和数据后，打开被屏蔽的中断，给相关线程发送一条通知（可以是 RT-Thread 所提供的信号量、事件、邮箱或消息队列等方式），然后结束中断服务程序；而接下来，相关的线程在接收到通知后，接着对状态或数据进行进一步的处理，这一过程称之为**底半处理**。

为了详细描述底半处理在 RT-Thread 中的实现，我们以一个虚拟的网络设备接收网络数据包作为范例，如下代码，并假设接收到数据报文后，系统对报文的分析、处理是一个相对耗时的，比外部中断源信号重要性小许多的，而且在不屏蔽中断源信号情况下也能处理的过程。

这个例子的程序创建了一个 nwt 线程，这个线程在启动运行后，将阻塞在 nw_bh_sem 信号上，一旦这个信号量被释放，将执行接下来的 nw_packet_parser 过程，开始 Bottom Half 的事件处理。

```c
/*
 * 程序清单：中断底半处理例子
 */

/* 用于唤醒线程的信号量 */
rt_sem_t nw_bh_sem;

/* 数据读取、分析的线程 */
void demo_nw_thread(void *param)
{
    /* 首先对设备进行必要的初始化工作 */
    device_init_setting();

    /*.. 其他的一些操作..*/

    /* 创建一个 semaphore 来响应 Bottom Half 的事件 */
    nw_bh_sem = rt_sem_create("bh_sem", 0, RT_IPC_FLAG_PRIO);

    while(1)
    {
        /* 最后，让 demo_nw_thread 等待在 nw_bh_sem 上 */
        rt_sem_take(nw_bh_sem, RT_WAITING_FOREVER);

        /* 接收到 semaphore 信号后，开始真正的 Bottom Half 处理过程 */
        nw_packet_parser (packet_buffer);
        nw_packet_process(packet_buffer);
    }
}

int main(void)
{
    rt_thread_t thread;

    /* 创建处理线程 */
    thread = rt_thread_create("nwt",demo_nw_thread, RT_NULL, 1024, 20, 5);

    if (thread != RT_NULL)
        rt_thread_startup(thread);
}
```

接下来让我们来看一下 demo_nw_isr 中是如何处理 Top Half，并开启 Bottom Half 的，如下例。

```c
void demo_nw_isr(int vector, void *param)
{
    /* 当 network 设备接收到数据后，陷入中断异常，开始执行此 ISR */
    /* 开始 Top Half 部分的处理，如读取硬件设备的状态以判断发生了何种中断 */
    nw_device_status_read();

    /*.. 其他一些数据操作等..*/

    /* 释放 nw_bh_sem，发送信号给 demo_nw_thread，准备开始 Bottom Half */
    rt_sem_release(nw_bh_sem);

    /* 然后退出中断的 Top Half 部分，结束 device 的 ISR */
}
```

从上面例子的两个代码片段可以看出，中断服务程序通过对一个信号量对象的等待和释放，来完成中断 Bottom Half 的起始和终结。由于将中断处理划分为 Top 和 Bottom 两个部分后，使得中断处理过程变为异步过程。这部分系统开销需要用户在使用 RT-Thread 时，必须认真考虑中断服务的处理时间是否大于给 Bottom Half 发送通知并处理的时间。



## RT-Thread 中断管理接口

为了把操作系统和系统底层的异常、中断硬件隔离开来，RT-Thread 把中断和异常封装为一组抽象接口，如下图所示：

![image-20241217153946884](C:\Users\FFZNB\AppData\Roaming\Typora\typora-user-images\image-20241217153946884.png)



### 中断服务程序挂接

系统把用户的**中断服务程序** (handler) 和**指定的中断号**关联起来，可调用如下的接口挂载一个新的中断服务程序：

```c
rt_isr_handler_t rt_hw_interrupt_install(int vector,
                                        rt_isr_handler_t  handler,
                                        void *param,
                                        char *name);
```

调用 rt_hw_interrupt_install() 后，当这个中断源产生中断时，系统将自动调用装载的中断服务程序。

下表描述了rt_hw_interrupt_install() 的输入参数和返回值：

| **参数** | **描述**                                         |
| -------- | ------------------------------------------------ |
| vector   | vector 是挂载的中断号                            |
| handler  | 新挂载的中断服务程序                             |
| param    | param 会作为参数传递给中断服务程序               |
| name     | 中断的名称                                       |
| **返回** | ——                                               |
| return   | 挂载这个中断服务程序之前挂载的中断服务程序的句柄 |



注：这个 API 并不会出现在每一个移植分支中，例如通常 Cortex-M0/M3/M4 的移植分支中就没有这个 API。

中断服务程序是一种需要特别注意的**运行环境**，它运行在非线程的执行环境下（一般为芯片的一种特殊运行模式（特权模式）），在这个运行环境中不能使用挂起当前线程的操作，因为当前线程并不存在，执行相关的操作会有类似打印提示信息，“Function [abc_func] shall not used in ISR”，含义是不应该在**中断服务程序**中调用的函数）。



### 中断源管理

通常在 ISR 准备处理某个中断信号之前，我们需要先屏蔽该中断源，在 ISR 处理完状态或数据以后，及时的打开之前被屏蔽的中断源。

屏蔽中断源可以保证在接下来的处理过程中**硬件状态或者数据**不会受到干扰，可调用下面这个函数接口：

```c
void rt_hw_interrupt_mask(int vector);
```

调用 rt_hw_interrupt_mask 函数接口后，相应的中断将会被屏蔽（通常当这个中断触发时，中断状态寄存器会有相应的变化，但并不送达到处理器进行处理）。

下表描述了rt_hw_interrupt_mask() 的输入参数：

| **参数** | **描述**       |
| -------- | -------------- |
| vector   | 要屏蔽的中断号 |

注：这个 API 并不会出现在每一个移植分支中，例如通常 Cortex-M0/M3/M4 的移植分支中就没有这个 API。



为了尽可能的不丢失硬件中断信号，可调用下面的函数接口**打开**被屏蔽的中断源：

```c
void rt_hw_interrupt_umask(int vector);
```

调用 rt_hw_interrupt_umask 函数接口后，如果中断（及对应外设）被配置正确时，中断触发后，将送到到处理器进行处理。

下表描述了rt_hw_interrupt_umask() 的输入参数

| **参数** | **描述**           |
| -------- | ------------------ |
| vector   | 要打开屏蔽的中断号 |

注：这个 API 并不会出现在每一个移植分支中，例如通常 Cortex-M0/M3/M4 的移植分支中就没有这个 API。



### 全局中断开关

**全局中断开关**也称为**中断锁**，是禁止多线程访问临界区最简单的一种方式，即通过关闭中断的方式，来保证当前线程不会被其他事件打断（因为整个系统已经不再响应那些可以触发线程重新调度的外部事件），也就是当前线程不会被抢占，除非这个线程主动放弃了处理器控制权。

当需要**关闭整个系统的中断**时，可调用下面的函数接口：

```c
rt_base_t rt_hw_interrupt_disable(void);
```

下表描述了rt_hw_interrupt_disable() 的返回值

| **返回** | **描述**                                     |
| -------- | -------------------------------------------- |
| 中断状态 | rt_hw_interrupt_disable 函数运行前的中断状态 |



**恢复中断**也称开中断。rt_hw_interrupt_enable()这个函数用于 **“使能” 中断**，它恢复了调用 rt_hw_interrupt_disable()函数前的中断状态。如果调用 rt_hw_interrupt_disable()函数前是关中断状态，那么调用此函数后依然是关中断状态。恢复中断往往是和关闭中断成对使用的，调用的函数接口如下：

```c
void rt_hw_interrupt_enable(rt_base_t level);
```

下表描述了rt_hw_interrupt_enable() 的输入参数：

| **参数** | **描述**                                      |
| -------- | --------------------------------------------- |
| level    | 前一次 rt_hw_interrupt_disable 返回的中断状态 |

1）使用中断锁来操作临界区的方法可以应用于任何场合，且其他几类同步方式都是依赖于中断锁而实现的，可以说中断锁是最强大的和最高效的同步方法。

只是使用中断锁最主要的问题在于，在中断关闭期间系统将**不再响应任何中断**，也就**不能响应外部的事件**。

所以中断锁对系统的实时性影响非常巨大，当使用不当的时候会导致系统完全无实时性可言（可能导致系统完全偏离要求的时间需求）；而使用得当，则会变成一种快速、高效的同步方式。



例如，为了保证一行代码（例如赋值）的互斥运行，最快速的方法是使用中断锁而不是信号量或互斥量：

```c
    /* 关闭中断 */
    level = rt_hw_interrupt_disable();
    /* 执行互斥操作 */
    a = a + value;
    /* 恢复中断 */
    rt_hw_interrupt_enable(level);
```

在使用中断锁时，需要确保关闭中断的时间非常短，例如上面代码中的 a = a + value; 

意味着**在同一时间只能有一个任务执行这个操作，以防止多个任务同时修改`a`的值，导致数据不一致**。

也可换成另外一种方式，例如使用信号量：

```c
    /* 获得信号量锁 */
    rt_sem_take(sem_lock, RT_WAITING_FOREVER);
    /* 执行互斥操作 */
    a = a + value;
    /* 释放信号量锁 */
    rt_sem_release(sem_lock);
```

这段代码在 rt_sem_take 、rt_sem_release 的实现中，已经存在使用中断锁保护信号量内部变量的行为，所以对于简单如 a = a + value; 的操作，使用中断锁将更为简洁快速。

2）函数 rt_base_t rt_hw_interrupt_disable(void) 和函数 void rt_hw_interrupt_enable(rt_base_t level) 一般需要配对使用，从而保证正确的中断状态。



**操作区别：**

1. **使用中断禁用（中断锁）**：
   - 这种方法通过快速禁用和恢复中断来实现临界区的互斥访问。它的优点是实现简单，执行速度快，因为它直接操作硬件中断，而不需要进行操作系统层面的上下文切换。
   - 但是，这种方法要求临界区代码必须非常短，以减少对系统响应性的影响。如果临界区代码过长，可能会导致系统无法及时响应外部事件或中断，从而影响系统性能。
2. **使用信号量**：
   - 信号量是一种更高级的同步机制，它允许任务或线程在等待资源可用时进入睡眠状态，而不是忙等待。这可以减少CPU的浪费，并允许操作系统调度其他任务运行。
   - 在RTOS中，信号量的获取和释放操作通常也是通过中断禁用来保护的，以确保操作的原子性。这意味着即使使用信号量，操作系统内部也会使用类似中断禁用的技术来保护信号量的内部状态。
   - 信号量提供了更灵活的同步控制，例如可以设置超时时间，允许任务在一定时间内未能获得信号量时释放CPU并尝试其他操作。



在 RT-Thread 中，开关全局中断的 API 支持多级嵌套使用，简单嵌套中断的代码如下代码所示：

简单嵌套中断使用

```c
#include <rthw.h>

void global_interrupt_demo(void)
{
    rt_base_t level0;
    rt_base_t level1;

    /* 第一次关闭全局中断，关闭之前的全局中断状态可能是打开的，也可能是关闭的 */
    level0 = rt_hw_interrupt_disable();
    /* 第二次关闭全局中断，关闭之前的全局中断是关闭的，关闭之后全局中断还是关闭的 */
    level1 = rt_hw_interrupt_disable();

    do_something();

    /* 恢复全局中断到第二次关闭之前的状态，所以本次 enable 之后全局中断还是关闭的 */
    rt_hw_interrupt_enable(level1);
    /* 恢复全局中断到第一次关闭之前的状态，这时候的全局中断状态可能是打开的，也可能是关闭的 */
    rt_hw_interrupt_enable(level0);
}
```

这个特性可以给代码的开发带来很大的便利。例如在某个函数里关闭了中断，然后调用某些子函数，再打开中断。这些子函数里面也可能存在开关中断的代码。由于**全局中断的 API 支持嵌套使用**，用户无需为这些代码做特殊处理。



### 中断通知

当整个系统被中断打断，进入中断处理函数时，需要**通知内核当前已经进入到中断状态**。

针对这种情况，可通过以下接口：

```c
void rt_interrupt_enter(void);
void rt_interrupt_leave(void);
```

这两个接口分别用在中断前导程序和中断后续程序中，均会对 **rt_interrupt_nest（中断嵌套深度）**的值进行修改：

每当进入中断时，可以调用 rt_interrupt_enter() 函数，用于通知内核，当前已经进入了中断状态，并增加中断嵌套深度（执行 rt_interrupt_nest++）；

每当退出中断时，可以调用 rt_interrupt_leave() 函数，用于通知内核，当前已经离开了中断状态，并减少中断嵌套深度（执行 rt_interrupt_nest --）。注意**不要在应用程序中调用这两个接口函数**。

使用 rt_interrupt_enter/leave() 的作用是，在中断服务程序中，如果调用了内核相关的函数（如释放信号量等操作），则可以通过判断当前中断状态，让内核及时调整相应的行为。

例如：在中断中释放了一个信号量，唤醒了某线程，但通过判断发现当前系统处于中断上下文环境中，那么在进行线程切换时应该采取中断中线程切换的策略，而不是立即进行切换。

但如果中断服务程序**不会调用内核相关的函数（释放信号量等操作）**，这个时候，**也可以不调用 rt_interrupt_enter/leave() 函数**。

在上层应用中，在内核需要知道当前已经进入到中断状态或当前嵌套的中断深度时，可调用 **rt_interrupt_get_nest() 接口**，它会**返回 rt_interrupt_nest**。

如下：

```c
rt_uint8_t rt_interrupt_get_nest(void);
```

下表描述了 rt_interrupt_get_nest() 的返回值

| **返回** | **描述**                       |
| -------- | ------------------------------ |
| 0        | 当前系统不处于中断上下文环境中 |
| 1        | 当前系统处于中断上下文环境中   |
| 大于 1   | **当前中断嵌套层次**           |



## 中断与轮询

当驱动外设工作时，其编程模式到底采用**中断模式触发**还是**轮询模式触发**往往是驱动开发人员首先要考虑的问题，并且这个问题在实时操作系统与分时操作系统中差异还非常大。

因为轮询模式本身采用顺序执行的方式：查询到相应的事件然后进行对应的处理。所以轮询模式从实现上来说，相对简单清晰。

往串口中写入数据，仅当串口控制器写完一个数据时，程序代码才写入下一个数据（否则这个数据丢弃掉）。

相应的代码可以是这样的：

```c
/* 轮询模式向串口写入数据 */
    while (size)
    {
        /* 判断 UART 外设中数据是否发送完毕 */
        while (!(uart->uart_device->SR & USART_FLAG_TXE));
        /* 当所有数据发送完毕后，才发送下一个数据 */
        uart->uart_device->DR = (*ptr & 0x1FF);

        ++ptr; --size;
    }
```

在实时系统中轮询模式可能会出现非常大问题，因为在实时操作系统中，**当一个程序持续地执行时（轮询时），它所在的线程会一直运行，比它优先级低的线程都不会得到运行**。

而分时系统中，这点恰恰相反，几乎没有优先级之分，可以在一个时间片运行这个程序，然后在另外一段时间片上运行另外一段程序。

所以通常情况下，实时系统中更多采用的是**中断模式**来驱动外设。

当数据达到时，由中断唤醒相关的处理线程，再继续进行后续的动作。

例如一些携带 FIFO（包含一定数据量的先进先出队列）的串口外设，其写入过程可以是这样的，如下图所示：

![中断模式驱动外设](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/interrupt/figures/09interrupt_reque.png)

线程先向串口的 FIFO 中写入数据，当 FIFO 满时，线程主动挂起。串口控制器持续地从 FIFO 中取出数据并以配置的波特率（例如 115200bps）发送出去。

当 FIFO 中所有数据都发送完成时，将向处理器触发一个中断；当中断服务程序得到执行时，可以唤醒这个线程。

这里举例的是 FIFO 类型的设备，在现实中也有 DMA 类型的设备，原理类似。



对于**低速设备**来说，运用这种模式非常好，因为在串口外设把 FIFO 中的数据发送出去前，处理器可以运行其他的线程，这样就提高了系统的整体运行效率（甚至对于分时系统来说，这样的设计也是非常必要）。

但是对于一些高速设备，例如传输速度达到 10Mbps 的时候，假设一次发送的数据量是 32 字节，我们可以计算出发送这样一段数据量需要的时间是：(32 X 8) X 1/10Mbps = 25us。

当数据需要持续传输时，系统将在 25us 后触发一个中断以唤醒上层线程继续下次传递。假设系统的线程切换时间是 8us（通常实时操作系统的线程上下文切换时间只有几个 us），那么当整个系统运行时，对于数据带宽利用率将只有 25/(25+8) =75.8%。但是采用轮询模式，数据带宽的利用率则可能达到 100%。这个也是大家普遍认为实时系统中数据吞吐量不足的缘故，系统开销消耗在了线程切换上（有些实时系统甚至会如本章前面说的，采用**底半处理，分级的**中断处理方式，相当于又**拉长**中断到发送线程的时间开销，效率会更进一步下降）。



通过上述的计算过程，我们可以看出其中的一些关键因素：

发送数据量越小，发送速度越快，对于数据吞吐量的影响也将越大。

归根结底，取决于系统中产生中断的频度如何。当一个实时系统想要提升数据吞吐量时，可以考虑的几种方式：

1）增加每次数据量发送的长度，每次尽量让外设尽量多地发送数据；

2）必要情况下更改中断模式为轮询模式。同时为了解决轮询方式一直抢占处理机，其他低优先级线程得不到运行的情况，可以把**轮询线程的优先级适当降低**。



## 全局中断开关使用示例

这是一个中断的应用例程：在多线程访问同一个变量时，使用开关全局中断对该变量进行保护，如下代码所示：

使用开关中断进行全局变量的访问

```c
#include <rthw.h>
#include <rtthread.h>

#define THREAD_PRIORITY      20
#define THREAD_STACK_SIZE    512
#define THREAD_TIMESLICE     5

/* 同时访问的全局变量 */
static rt_uint32_t cnt;
void thread_entry(void *parameter)
{
    rt_uint32_t no;
    rt_uint32_t level;

    no = (rt_uint32_t) parameter;
    while (1)
    {
        /* 关闭全局中断 */
        level = rt_hw_interrupt_disable();
        /* 执行互斥操作 */
        cnt += no;
        /* 恢复全局中断 */
        rt_hw_interrupt_enable(level);

        rt_kprintf("protect thread[%d]'s counter is %d\n", no, cnt);
        rt_thread_mdelay(no * 10);
    }
}

/* 用户应用程序入口 */
int interrupt_sample(void)
{
    rt_thread_t thread;

    /* 创建 t1 线程 */
    thread = rt_thread_create("thread1", thread_entry, (void *)10,
                              THREAD_STACK_SIZE,
                              THREAD_PRIORITY, THREAD_TIMESLICE);
    if (thread != RT_NULL)
        rt_thread_startup(thread);


    /* 创建 t2 线程 */
    thread = rt_thread_create("thread2", thread_entry, (void *)20,
                              THREAD_STACK_SIZE,
                              THREAD_PRIORITY, THREAD_TIMESLICE);
    if (thread != RT_NULL)
        rt_thread_startup(thread);

    return 0;
}

/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(interrupt_sample, interrupt sample);
```

仿真运行结果如下：

```shell
 \ | /
- RT -     Thread Operating System
 / | \     3.1.0 build Aug 27 2018
 2006 - 2018 Copyright by rt-thread team
msh >interrupt_sample
msh >protect thread[10]'s counter is 10
protect thread[20]'s counter is 30
protect thread[10]'s counter is 40
protect thread[20]'s counter is 60
protect thread[10]'s counter is 70
protect thread[10]'s counter is 80
protect thread[20]'s counter is 100
protect thread[10]'s counter is 110
protect thread[10]'s counter is 120
protect thread[20]'s counter is 140
…
```

注：由于关闭全局中断会导致整个系统不能响应中断，所以在使用关闭全局中断做为互斥访问临界区的手段时，**必须需要保证关闭全局中断的时间非常短**，例如运行数条机器指令的时间。



# 内核移植

经过前面内核章节的学习，大家对 RT-Thread 也有了不少的了解，但是如何将 RT-Thread 内核移植到不同的硬件平台上，很多人还不一定熟悉。

内核移植就是指将 RT-Thread 内核在不同的芯片架构、不同的板卡上运行起来，能够具备线程管理和调度，内存管理，线程间同步和通信、定时器管理等功能。

移植可分为 CPU 架构移植和 BSP（Board support package，板级支持包）移植两部分。

本章将展开介绍 CPU 架构移植和 BSP 移植，CPU 架构移植部分会结合 Cortex-M CPU 架构进行介绍，因此有必要回顾下上一章的中断管理介绍的 “Cortex-M CPU 架构基础” 的内容，

本章最后以实际移植到一个开发板的示例展示 RT-Thread 内核移植的完整过程，读完本章，我们将了解如何完成 RT-Thread 的内核移植。

## CPU 架构移植

在嵌入式领域有多种不同 CPU 架构，例如 Cortex-M、ARM920T、MIPS32、RISC-V 等等。为了使 RT-Thread 能够在不同 CPU 架构的芯片上运行，RT-Thread 提供了一个 **libcpu 抽象层**来**适配**不同的 CPU 架构。l

ibcpu 层向上对内核提供统一的接口，包括全局中断的开关，线程栈的初始化，上下文切换等。

RT-Thread 的 libcpu 抽象层向下提供了一套统一的 CPU 架构移植接口，这部分接口包含了全局中断开关函数、线程上下文切换函数、时钟节拍的配置和中断函数、Cache 等等内容。

下表是 CPU 架构移植需要实现的接口和变量。

libcpu 移植相关 API：

| **函数和变量**                                               | **描述**                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| rt_base_t rt_hw_interrupt_disable(void);                     | 关闭全局中断                                                 |
| void rt_hw_interrupt_enable(rt_base_t level);                | 打开全局中断                                                 |
| rt_uint8_t *rt_hw_stack_init(void *tentry, void *parameter, rt_uint8_t *stack_addr, void *texit); | 线程栈的初始化，内核在线程创建和线程初始化里面会调用这个函数 |
| void rt_hw_context_switch_to(rt_uint32_t to);                | 没有来源线程的上下文切换，在调度器启动第一个线程的时候调用，以及在 signal 里面会调用 |
| void rt_hw_context_switch(rt_uint32_t from, rt_uint32_t to); | 从 from 线程切换到 to 线程，用于线程和线程之间的切换         |
| void rt_hw_context_switch_interrupt(rt_uint32_t from, rt_uint32_t to); | 从 from 线程切换到 to 线程，用于中断里面进行切换的时候使用   |
| rt_uint32_t rt_thread_switch_interrupt_flag;                 | 表示需要在中断里进行切换的标志                               |
| rt_uint32_t rt_interrupt_from_thread, rt_interrupt_to_thread; | 在线程进行上下文切换时候，用来保存 from 和 to 线程           |



### 实现全局中断开关

无论内核代码还是用户的代码，都可能存在一些变量，需要在多个线程或者中断里面使用，如果没有相应的保护机制，那就可能导致临界区问题。

RT-Thread 里为了解决这个问题，提供了一系列的线程间同步和通信机制来解决。但是这些机制都需要用到 libcpu 里提供的全局中断开关函数。

它们分别是：

```c
/* 关闭全局中断 */
rt_base_t rt_hw_interrupt_disable(void);

/* 打开全局中断 */
void rt_hw_interrupt_enable(rt_base_t level);
```

下面介绍在 Cortex-M 架构上如何实现这两个函数，前文中曾提到过，Cortex-M 为了快速开关中断，实现了 CPS 指令，可以用在此处。

```c
CPSID I ;PRIMASK=1， ; 关中断
CPSIE I ;PRIMASK=0， ; 开中断  
```

#### 关闭全局中断

在 rt_hw_interrupt_disable() 函数里面需要依序完成的功能是：

1）. **保存当前的全局中断状态，并把状态作为函数的返回值**。

2）. **关闭全局中断**。

基于 MDK，在 Cortex-M 内核上实现关闭全局中断，如下代码所示：

关闭全局中断

```c
;/*
; * rt_base_t rt_hw_interrupt_disable(void);
; */
rt_hw_interrupt_disable    PROC      ;PROC 伪指令定义函数
    EXPORT  rt_hw_interrupt_disable  ;EXPORT 输出定义的函数，类似于 C 语言 extern
    MRS     r0, PRIMASK              ; 读取 PRIMASK 寄存器的值到 r0 寄存器
    CPSID   I                        ; 关闭全局中断
    BX      LR                       ; 函数返回
    ENDP                             ;ENDP 函数结束
```

上面的代码首先是使用 MRS 指令将 PRIMASK 寄存器的值保存到 r0 寄存器里，然后使用 “CPSID I” 指令关闭全局中断，最后使用 BX 指令返回。

r0 存储的数据就是函数的返回值。中断可以发生在 “MRS r0, PRIMASK” 指令和 “CPSID I” 之间，这并不会导致全局中断状态的错乱。



关于寄存器在函数调用的时候和在中断处理程序里是如何管理的，不同的 CPU 架构有不同的约定。在 ARM 官方手册《Procedure Call Standard for the ARM ® Architecture》里可以找到关于 Cortex-M 的更详细的介绍寄存器使用的约定。



#### 打开全局中断

在 rt_hw_interrupt_enable(rt_base_t level) 里，将变量 level 作为需要恢复的状态，覆盖芯片的全局中断状态。

基于 MDK，在 Cortex-M 内核上的实现打开全局中断，如下代码所示：

打开全局中断

```c
;/*
; * void rt_hw_interrupt_enable(rt_base_t level);
; */
rt_hw_interrupt_enable    PROC      ; PROC 伪指令定义函数
    EXPORT  rt_hw_interrupt_enable  ; EXPORT 输出定义的函数，类似于 C 语言 extern
    MSR     PRIMASK, r0             ; 将 r0 寄存器的值写入到 PRIMASK 寄存器
    BX      LR                      ; 函数返回
    ENDP                            ; ENDP 函数结束
```

上面的代码首先是使用 MSR 指令将 r0 的值寄存器写入到 PRIMASK 寄存器，从而恢复之前的中断状态。

### 实现线程栈初始化

在**动态创建线程**和**初始化线程**的时候，会使用到内部的线程初始化函数_rt_thread_init()，_rt_thread_init() 函数会调用栈初始化函数 rt_hw_stack_init()，在栈初始化函数里会手动构造一个上下文内容，这个上下文内容将被作为每个线程第一次执行的初始值。

上下文在栈里的排布如下图所示：

![栈里的上下文信息](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/porting/figures/10stack.png)

以下代码是栈初始化的代码：

**在栈里构建上下文**

```c
rt_uint8_t *rt_hw_stack_init(void       *tentry,
                             void       *parameter,
                             rt_uint8_t *stack_addr,
                             void       *texit)
{
    struct stack_frame *stack_frame;
    rt_uint8_t         *stk;
    unsigned long       i;

    /* 对传入的栈指针做对齐处理 */
    stk  = stack_addr + sizeof(rt_uint32_t);
    stk  = (rt_uint8_t *)RT_ALIGN_DOWN((rt_uint32_t)stk, 8);
    stk -= sizeof(struct stack_frame);

    /* 得到上下文的栈帧的指针 */
    stack_frame = (struct stack_frame *)stk;

    /* 把所有寄存器的默认值设置为 0xdeadbeef */
    for (i = 0; i < sizeof(struct stack_frame) / sizeof(rt_uint32_t); i ++)
    {
        ((rt_uint32_t *)stack_frame)[i] = 0xdeadbeef;
    }

    /* 根据 ARM  APCS 调用标准，将第一个参数保存在 r0 寄存器 */
    stack_frame->exception_stack_frame.r0  = (unsigned long)parameter;
    /* 将剩下的参数寄存器都设置为 0 */
    stack_frame->exception_stack_frame.r1  = 0;                 /* r1 寄存器 */
    stack_frame->exception_stack_frame.r2  = 0;                 /* r2 寄存器 */
    stack_frame->exception_stack_frame.r3  = 0;                 /* r3 寄存器 */
    /* 将 IP(Intra-Procedure-call scratch register.) 设置为 0 */
    stack_frame->exception_stack_frame.r12 = 0;                 /* r12 寄存器 */
    /* 将线程退出函数的地址保存在 lr 寄存器 */
    stack_frame->exception_stack_frame.lr  = (unsigned long)texit;
    /* 将线程入口函数的地址保存在 pc 寄存器 */
    stack_frame->exception_stack_frame.pc  = (unsigned long)tentry;
    /* 设置 psr 的值为 0x01000000L，表示默认切换过去是 Thumb 模式 */
    stack_frame->exception_stack_frame.psr = 0x01000000L;

    /* 返回当前线程的栈地址       */
    return stk;
}
```

### [实现上下文切换](https://www.rt-thread.org/document/site/#/rt-thread-version/rt-thread-standard/programming-manual/porting/porting?id=实现上下文切换)

在不同的 CPU 架构里，线程之间的上下文切换和中断到线程的上下文切换，上下文的寄存器部分可能是有差异的，也可能是一样的。在 Cortex-M 里面上下文切换都是统一使用 PendSV 异常来完成，切换部分并没有差异。但是为了能适应不同的 CPU 架构，RT-Thread 的 libcpu 抽象层还是需要实现三个线程切换相关的函数：

1） rt_hw_context_switch_to()：没有来源线程，切换到目标线程，在调度器启动第一个线程的时候被调用。

2） rt_hw_context_switch()：在线程环境下，从当前线程切换到目标线程。

3） rt_hw_context_switch_interrupt ()：在中断环境下，从当前线程切换到目标线程。

在线程环境下进行切换和在中断环境进行切换是存在差异的。线程环境下，如果调用 rt_hw_context_switch() 函数，那么可以马上进行上下文切换；而在中断环境下，需要等待中断处理函数完成之后才能进行切换。

由于这种差异，在 ARM9 等平台，rt_hw_context_switch() 和 rt_hw_context_switch_interrupt() 的实现并不一样。在中断处理程序里如果触发了线程的调度，调度函数里会调用 rt_hw_context_switch_interrupt() 触发上下文切换。中断处理程序里处理完中断事务之后，中断退出之前，检查 rt_thread_switch_interrupt_flag 变量，如果该变量的值为 1，就根据 rt_interrupt_from_thread 变量和 rt_interrupt_to_thread 变量，完成线程的上下文切换。

在 Cortex-M 处理器架构里，基于自动部分压栈和 PendSV 的特性，上下文切换可以实现地更加简洁。

线程之间的上下文切换，如下图表示：

![线程之间的上下文切换](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/porting/figures/10ths_env1.png)

硬件在进入 PendSV 中断之前自动保存了 from 线程的 PSR、PC、LR、R12、R3-R0 寄存器，然后 PendSV 里保存 from 线程的 R11~R4 寄存器，以及恢复 to 线程的 R4~R11 寄存器，最后硬件在退出 PendSV 中断之后，自动恢复 to 线程的 R0~R3、R12、LR、PC、PSR 寄存器。

中断到线程的上下文切换可以用下图表示：

![中断到线程的切换](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/porting/figures/10ths_env2.png)

硬件在进入中断之前自动保存了 from 线程的 PSR、PC、LR、R12、R3-R0 寄存器，然后触发了 PendSV 异常。在 PendSV 异常处理函数里保存 from 线程的 R11~R4 寄存器，以及恢复 to 线程的 R4~R11 寄存器，最后硬件在退出 PendSV 中断之后，自动恢复 to 线程的 R0~R3、R12、PSR、PC、LR 寄存器。

显然，在 Cortex-M 内核里 rt_hw_context_switch() 和 rt_hw_context_switch_interrupt() 功能一致，都是在 PendSV 里完成剩余上下文的保存和回复。所以我们仅仅需要实现一份代码，简化移植的工作。

#### [实现 rt_hw_context_switch_to()](https://www.rt-thread.org/document/site/#/rt-thread-version/rt-thread-standard/programming-manual/porting/porting?id=实现-rt_hw_context_switch_to)

rt_hw_context_switch_to() 只有目标线程，没有来源线程。这个函数里实现切换到指定线程的功能，下图是流程图：

![rt_hw_context_switch_to() 流程图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/porting/figures/10switch.png)

在 Cortex-M3 内核上的 rt_hw_context_switch_to() 实现（基于 MDK），如下代码所示：

MDK 版 rt_hw_context_switch_to() 实现

```c
;/*
; * void rt_hw_context_switch_to(rt_uint32_t to);
; * r0 --> to
; * this fucntion is used to perform the first thread switch
; */
rt_hw_context_switch_to    PROC
    EXPORT rt_hw_context_switch_to
    ; r0 的值是一个指针，该指针指向 to 线程的线程控制块的 SP 成员
    ; 将 r0 寄存器的值保存到 rt_interrupt_to_thread 变量里
    LDR     r1, =rt_interrupt_to_thread
    STR     r0, [r1]

    ; 设置 from 线程为空，表示不需要保存 from 的上下文
    LDR     r1, =rt_interrupt_from_thread
    MOV     r0, #0x0
    STR     r0, [r1]

    ; 设置标志为 1，表示需要切换，这个变量将在 PendSV 异常处理函数里切换时被清零
    LDR     r1, =rt_thread_switch_interrupt_flag
    MOV     r0, #1
    STR     r0, [r1]

    ; 设置 PendSV 异常优先级为最低优先级
    LDR     r0, =NVIC_SYSPRI2
    LDR     r1, =NVIC_PENDSV_PRI
    LDR.W   r2, [r0,#0x00]       ; read
    ORR     r1,r1,r2             ; modify
    STR     r1, [r0]             ; write-back

    ; 触发 PendSV 异常 (将执行 PendSV 异常处理程序)
    LDR     r0, =NVIC_INT_CTRL
    LDR     r1, =NVIC_PENDSVSET
    STR     r1, [r0]

    ; 放弃芯片启动到第一次上下文切换之前的栈内容，将 MSP 设置启动时的值
    LDR     r0, =SCB_VTOR
    LDR     r0, [r0]
    LDR     r0, [r0]
    MSR     msp, r0

    ; 使能全局中断和全局异常，使能之后将进入 PendSV 异常处理函数
    CPSIE   F
    CPSIE   I

    ; 不会执行到这里
    ENDP复制错误复制成功
```

#### [实现 rt_hw_context_switch()/ rt_hw_context_switch_interrupt()](https://www.rt-thread.org/document/site/#/rt-thread-version/rt-thread-standard/programming-manual/porting/porting?id=实现-rt_hw_context_switch-rt_hw_context_switch_interrupt)

函数 rt_hw_context_switch() 和函数 rt_hw_context_switch_interrupt() 都有两个参数，分别是 from 线程和 to 线程。它们实现从 from 线程切换到 to 线程的功能。下图是具体的流程图：

![rt_hw_context_switch()/ rt_hw_context_switch_interrupt() 流程图](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/porting/figures/10switch2.png)

在 Cortex-M3 内核上的 rt_hw_context_switch() 和 rt_hw_context_switch_interrupt() 实现（基于 MDK），如下代码所示：

rt_hw_context_switch()/rt_hw_context_switch_interrupt() 实现

```c
;/*
; * void rt_hw_context_switch(rt_uint32_t from, rt_uint32_t to);
; * r0 --> from
; * r1 --> to
; */
rt_hw_context_switch_interrupt
    EXPORT rt_hw_context_switch_interrupt
rt_hw_context_switch    PROC
    EXPORT rt_hw_context_switch

    ; 检查 rt_thread_switch_interrupt_flag 变量是否为 1
    ; 如果变量为 1 就跳过更新 from 线程的内容
    LDR     r2, =rt_thread_switch_interrupt_flag
    LDR     r3, [r2]
    CMP     r3, #1
    BEQ     _reswitch
    ; 设置 rt_thread_switch_interrupt_flag 变量为 1
    MOV     r3, #1
    STR     r3, [r2]

    ; 从参数 r0 里更新 rt_interrupt_from_thread 变量
    LDR     r2, =rt_interrupt_from_thread
    STR     r0, [r2]

_reswitch
    ; 从参数 r1 里更新 rt_interrupt_to_thread 变量
    LDR     r2, =rt_interrupt_to_thread
    STR     r1, [r2]

    ; 触发 PendSV 异常，将进入 PendSV 异常处理函数来完成上下文切换
    LDR     r0, =NVIC_INT_CTRL
    LDR     r1, =NVIC_PENDSVSET
    STR     r1, [r0]
    BX      LR复制错误复制成功
```

#### [实现 PendSV 中断](https://www.rt-thread.org/document/site/#/rt-thread-version/rt-thread-standard/programming-manual/porting/porting?id=实现-pendsv-中断)

在 Cortex-M3 里，PendSV 中断处理函数是 PendSV_Handler()。在 PendSV_Handler() 里完成线程切换的实际工作，下图是具体的流程图：

![PendSV 中断处理](https://www.rt-thread.org/document/site/rt-thread-version/rt-thread-standard/programming-manual/porting/figures/10pendsv.jpg)

如下代码是 PendSV_Handler 实现：

```c
; r0 --> switch from thread stack
; r1 --> switch to thread stack
; psr, pc, lr, r12, r3, r2, r1, r0 are pushed into [from] stack
PendSV_Handler   PROC
    EXPORT PendSV_Handler

    ; 关闭全局中断
    MRS     r2, PRIMASK
    CPSID   I

    ; 检查 rt_thread_switch_interrupt_flag 变量是否为 0
    ; 如果为零就跳转到 pendsv_exit
    LDR     r0, =rt_thread_switch_interrupt_flag
    LDR     r1, [r0]
    CBZ     r1, pendsv_exit         ; pendsv already handled

    ; 清零 rt_thread_switch_interrupt_flag 变量
    MOV     r1, #0x00
    STR     r1, [r0]

    ; 检查 rt_interrupt_from_thread 变量是否为 0
    ; 如果为 0，就不进行 from 线程的上下文保存
    LDR     r0, =rt_interrupt_from_thread
    LDR     r1, [r0]
    CBZ     r1, switch_to_thread

    ; 保存 from 线程的上下文
    MRS     r1, psp                 ; 获取 from 线程的栈指针
    STMFD   r1!, {r4 - r11}       ; 将 r4~r11 保存到线程的栈里
    LDR     r0, [r0]
    STR     r1, [r0]                ; 更新线程的控制块的 SP 指针

switch_to_thread
    LDR     r1, =rt_interrupt_to_thread
    LDR     r1, [r1]
    LDR     r1, [r1]                ; 获取 to 线程的栈指针

    LDMFD   r1!, {r4 - r11}       ; 从 to 线程的栈里恢复 to 线程的寄存器值
    MSR     psp, r1                 ; 更新 r1 的值到 psp

pendsv_exit
    ; 恢复全局中断状态
    MSR     PRIMASK, r2

    ; 修改 lr 寄存器的 bit2，确保进程使用 PSP 堆栈指针
    ORR     lr, lr, #0x04
    ; 退出中断函数
    BX      lr
    ENDP复制错误复制成功
```

### [实现时钟节拍](https://www.rt-thread.org/document/site/#/rt-thread-version/rt-thread-standard/programming-manual/porting/porting?id=实现时钟节拍)

有了开关全局中断和上下文切换功能的基础，RTOS 就可以进行线程的创建、运行、调度等功能了。有了时钟节拍支持，RT-Thread 可以实现对相同优先级的线程采用时间片轮转的方式来调度，实现定时器功能，实现 rt_thread_delay() 延时函数等等。

libcpu 的移植需要完成的工作，就是确保 rt_tick_increase() 函数会在时钟节拍的中断里被周期性的调用，调用周期取决于 rtconfig.h 的宏 RT_TICK_PER_SECOND 的值。

在 Cortex M 中，实现 SysTick 的中断处理函数即可实现时钟节拍功能。

```c
void SysTick_Handler(void)
{
    /* enter interrupt */
    rt_interrupt_enter();

    rt_tick_increase();

    /* leave interrupt */
    rt_interrupt_leave();
}复制错误复制成功
```

## [BSP 移植](https://www.rt-thread.org/document/site/#/rt-thread-version/rt-thread-standard/programming-manual/porting/porting?id=bsp-移植)

相同的 CPU 架构在实际项目中，不同的板卡上可能使用相同的 CPU 架构，搭载不同的外设资源，完成不同的产品，所以我们也需要针对板卡做适配工作。RT-Thread 提供了 BSP 抽象层来适配常见的板卡。如果希望在一个板卡上使用 RT-Thread 内核，除了需要有相应的芯片架构的移植，还需要有针对板卡的移植，也就是实现一个基本的 BSP。主要任务是建立让操作系统运行的基本环境，需要完成的主要工作是：

1）初始化 CPU 内部寄存器，设定 RAM 工作时序。

2）实现时钟驱动及中断控制器驱动，完善中断管理。

3）实现串口和 GPIO 驱动。

4）初始化动态内存堆，实现动态堆内存管理。           





