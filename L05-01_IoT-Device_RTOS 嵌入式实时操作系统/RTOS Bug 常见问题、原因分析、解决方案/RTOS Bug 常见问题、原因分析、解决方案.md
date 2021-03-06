# RTOS Bug 常见问题、原因分析、解决方案

使用 RTOS 会使调试复杂化。

RTOS 可能会引入诸如优先级反转、死锁和任务抖动等问题。



# ■■■■■■■■■■■■■■■■■■■■■■

# 互斥量 相关问题

## 互斥量 vs 二值信号量 

- 拥有互斥量的线程拥有互斥量的所有权，互斥量支持递归访问且能防止线程**优先级翻转**；  
- 持有该互斥量的线程也能够再次获得这个锁而不被挂起，这就是**递归访问**。
- 并且互斥量只能由持有线程释放，而信号量则可以由任何线程释放。  
-  在信号量中， 由于已经不存在可用的信号量，线程递归获取信号量时会发生主动挂起（最终形成**死锁**） 。  
- 如果想要用于实现同步（线程之间或者线程与中断之间）， 二值信号量或许是更好的选择， 虽然互斥量也可以用于线程与线程、 线程与中断的同步，但是互斥量更多的是用于保护资源的互锁。用于互锁的互斥量可以充当保护资源的令牌。  
- 互斥量的加锁和解锁必须由同一线程分别对应使用，信号量可以由一个线程释放，另一个线程得到。
- 用于临界资源的保护一般建议使用互斥量。  
- **注意：**的是互斥量不能在中断服务函数中使用。 
  - 

## 互斥量的注意事项、问题

### 注意01：互斥量不能在中断服务函数中使用。 

- 其他能挂起的操作也都不能用在中断中。 
- 因为互斥锁上的锁定操作可以睡眠，并且在 ISR 中睡眠是非法的。导致中断无法退出，系统无法正常调试。如果您在中断时进行任何阻塞调用，则调度程序将永远不会运行。
- 系统中，有一个线程上下文和一个中断上下文。中断由硬件调用，而线程由RTOS调度程序调度。现在当中断发生时，任何线程都将立即被抢占;中断必须运行完成，只能被优先级较高的中断（支持嵌套中断）抢占。所有挂起的中断将在调度程序运行之前运行完毕。
- 请改用自旋锁。在中断处理程序中可以使用自旋锁”意思是使用自旋锁即使它一时获取不到需要的资源，也会在那里自旋，不会让出处理器，而导致睡眠。



### 问题01：多个线程获取互斥锁，多次优先级反转，而优先级的继承只能一次

- 互斥量的设置
  - 互斥量 flag 使用 FIFO 模式。
  - A（优先级20）已经获得互斥锁
  - 此时 B（优先级 10） 尝试获取互斥锁
  - 然后 C (优先级 1) 尝试获取互斥锁

- 运行过程：
  - 当此时 A 的优先级被暂时提高到 1，运行 A。
  - 等运行完毕后。由于采用 FIFO模式， A 线程会把锁交给线程 B，使得线程 B 开始运行。
  - 但是此时再也不会将 B 的优先级提升到 C 的优先级了，此时 C 就必须等到 B 完全运行退出，相当于再次出现了优先级反转问题。

- 这里问题的关键是，提升优先级的动作只会发生一次。

### 问题02：低优先级的线程得到意外运行

- A（优先级20）已经获得互斥锁
- 此时 B（优先级 10） 尝试获取互斥锁
- 如果 B 因为超时退出，而 A 的优先级仍然被短暂提高。
- 造成 A 线程意外得到了运行。

### 解决方案：应当处理优先级提权的场景

- 任务尝试获取互斥锁时，应当查看当前获取该锁的任务的优先级是否比自己低

- 任务获取到互斥锁时，应当查看阻塞在当前互斥量列表中的所有任务，看他们的优先级是否比自己高。

- 如果有任务比自己的优先级高，则将自己的优先级提高到最高任务的优先级（在 Flag 为 prio 模式下是不存在这个问题的，因为获取互斥锁的任务一定是当前列表中最高优先级的任务）

- 高优先级的阻塞时间不是 forever 时，应当在退出等待队列时，降低先前被提权的任务的优先级。

  

# ■■■■■■■■■■■■■■■■■■■■■■

# 优先级反转

## 现象：

- 优先级反转是实时系统中的一个问题，当使用基于优先级的抢占式内核时会发生。
- 使用信号量会导致的另一个潜在问题是线程优先级翻转问题。  
  - 所谓优先级翻转，即当一个高优先级线程试图通过信号量机制访问共享资源时，如果该信号量已被一低优先级线程持有，而这个低优先级线程在运行过程中可能又被其它一些中等优先级的线程抢占，因此造成高优先级线程被许多具有较低优先级的线程阻塞，实时性难以得到保证。  
  - 有优先级为 A、 B 和 C 的三个线程，优先级 A> B > C。
  - 线程 A， B 处于挂起状态，等待某一事件触发，线程 C 正在运行，此时线程 C 开始使用某一共享资源 M。
  - 在使用过程中，线程 A 等待的事件到来，线程 A 转为就绪态，因为它比线程 C 优先级高，所以立即执行。
  - 但是当线程 A 要使用共享资源 M 时，由于其正在被线程 C 使用，因此线程 A 被挂起切换到线程 C 运行。
  - 如果此时线程 B 等待的事件到来，则线程 B 转为就绪态。
  - 由于线程 B 的优先级比线程 C 高，因此线程 B 开始运行，直到其运行完毕，线程 C 才开始运行。
  - 只有当线程 C 释放共享资源 M 后，线程 A 才得以执行。
  - 在这种情况下，优先级发生了翻转：线程 B 先于线程 A 运行。
  - 这样便不能保证高优先级线程的响应时间。  
  - 在更坏的情况下，如 Task A和TaskC 之间有多个这样的 “Task B” 存在，这样的优先级反转问题可能会导致整个系统的崩溃。
- 优先级翻转的危害很大
  - 发生优先级翻转，对我们操作系统是致命的危害，会导致系统的高优先级线程阻塞时间过长。  
  - 假如很多个这样子的（中等优先级）线程打断最低优先级的线程，那这个系统最高优先级线程岂不是崩溃了。特别是对时序要求比较严格的系统。

## 解决方案：避免优先级反转

### 方案一：优先级继承：互斥信号量中使用了优先级继承

- 可以使用 RTOS 的互斥量机制来解决上面描述的优先级反转问题。互斥量可以解决优先级翻转问题，实现的是**优先级继承算法**。
  - 优先级继承是通过在线程 A 尝试获取共享资源而被挂起的期间内，将线程 C 的优先级提升到线程 A 的优先级别，从而解决优先级翻转引起的问题。
  - 这样能够防止 C（间接地防止 A）被 B 抢占。
  - 优先级继承是指，提高某个占有某种资源的低优先级线程的优先级，使之与所有等待该资源的线程中优先级最高的那个线程的优先级相等，然后执行，而当这个低优先级线程释放该资源时，优先级重新回到初始设定。
  - 因此，继承优先级的线程避免了系统资源被任何中间优先级的线程抢占。  
  - 这个优先级继承机制确保高优先级线程进入阻塞状态的时间尽可能短，以及将已经出现的 “优先级翻转” 危害降低到最小。但要完全消除需要在设计时避免。
  
- **注意 1：**在获得互斥量后，请尽快释放互斥量，并且在持有互斥量的过程中，不得再调用 rt_thread_control()等函数接口更改持有互斥量线程的优先级。   

- **注意 2：**不要出现多个线程获取互斥锁，多次优先级反转。因为目前，优先级的继承只能执行一次。

- **注意 3：**优先级继承 ，尽可能地降低高优先级任务处于阻塞态的时间，并且将已经出现的“优先级翻转”的影响降到最低，但是只能减少影响，不能完全避免。

- **注意 4：**使用 Tracealyzer 的执行实例视图，开发者立刻就能发现问题。

  ![优先级翻转, 优先级继承](figures/优先级翻转, 优先级继承.png)

### 方案二：设置优先级上限，给临界区一个高优先级

- 设置优先级上限，给临界区一个高优先级，进入临界区的进程都将获得这个高优先级
- 如果其他试图进入临界区的进程的优先级都低于这个高优先级，那么优先级反转就不会发生。

### 方案三：使用中断禁止，通过禁止中断来保护临界区

- 通过禁止中断来保护临界区，采用此种策略的系统只有两种优先级：可抢占优先级和中断禁止优先级。
- 第三种方法就是使用中断禁止，前者为一般进程运行时的优先级，后者为运行于临界区的优先级。

# ■■■■■■■■■■■■■■■■■■■■■■

# 死锁

## 现象：

- 死锁是至少两个任务相互等待另一个任务拥有的资源，导致任务都无法继续进行，就会发生死锁。
- 死锁可能不会立即发生，它很大程度上取决于两个任务何时需要彼此的资源。
- 例如， 在信号量中， 由于已经不存在可用的信号量，线程递归获取信号量时会发生主动挂起（最终形成**死锁**） 。  

## 检测方法：

- 通过监视/显示每个任务的执行频率（RTOS 切换任务的频率）, 来检测是否有死锁。
- 任务中加放一些计数器，如果至少两个任务的计数出现停止，则可能存在死锁。

## 解决方案：避免死锁

- 在设计时避免死锁的发生，如所有的任务都按照固定的顺序使用共享资源，或者同一时间任务不持有多个共享资源。
- 任务先获取所有必需的资源，以相同的顺序获取它们，以相反的顺序释放它们。
- 在 RTOS API 调用中使用超时机制，以避免永远等待资源可用。
- 检查 RTOS API 返回的错误代码，以确保对所需资源的请求成功。

# ■■■■■■■■■■■■■■■■■■■■■■

# 线程饥饿

## 现象：

- 在嵌入式多任务系统中，一些任务可能会执行缓慢，或者甚至得不到执行，常见的原因是由于优先级顺序设置不正确导致。
- 高优先级的任务使用了太多的 CPU 时间，低优先级的任务可能就没有足够的时间执行，这就是所谓的线程饥饿。
- 饥饿的影响是响应性和产品特性的下降，例如嵌入式目标的显示更新缓慢、通信堆栈中的数据包丢失、操作界面响应迟缓等。

## 解决方案：解决饥饿问题

- 优化消耗大多数CPU 带宽的代码。例如，减少该任务/线程的时间片
- 高优先级应该保留给可预测、循环执行、且循环周期较短的任务。
- 对于高优先级、CPU占用时间长的任务应该拆分为多个任务，缩小时间关键代码，通过任务同步机制，将占用CPU时间多的工作交给中或低优先级任务处理。
- 将受影响的任务的优先级提高，确实可以改善问题，但违背了使用优先级的意义。

# ■■■■■■■■■■■■■■■■■■■■■■

# 线程抖动

## 现象：

- 周期性执行的任务，随机发生的延迟时间叫做**抖动**。

- 虽然轻微的抖动很难避免，但是抖动太严重就会导致性能变差，间歇性的数据丢失。

- 例如，每隔 5 ms 调整电机的控制参数，如果控制任务的抖动过大就会使得控制性能就变差。

- 除了线程饥饿会导致抖动之外，RTOS 系统配置也会有影响，例如系统节拍定时器节拍频率。

  

## 解决方案：

- 理想情况下，两个节拍之间的时间应该比系统中最频繁任务的周期时间短得多。
- 使用 Tracealyzer 通过记录任务执行的时间，在以时间为坐标的图上很快可以发现抖动很大的执行实例，假如这是对抖动敏感的任务，这可能就是导致系统问题的原因了。

# ■■■■■■■■■■■■■■■■■■■■■■

# 临界区相关的问题

## 什么是共享资源

先解释共享资源，可以是一段程序、一个功能、一个动作、一段指令或者传输几个字节，也可以是不能同步运行的不相关的多段程序，不同的程序被封装成一个“外壳”，被认为是同一种共享资源。

## 什么是临界区 Critical Section

- 临界区指的是一个访问共用资源（例如：共用设备或是共用存储器）的程序片段，而这些共用资源又无法同时被多个线程访问的特性。
- 每次只准许一个进程进入临界区，进入后不允许其他进程进入。
- 不论是硬件临界资源，还是软件临界资源，多个进程必须互斥地对它进行访问。
- 有多个线程试图同时访问临界区，那么在有一个线程进入后其他所有试图访问此临界区的线程将被挂起，并一直持续到进入临界区的线程离开。临界区在被释放后，其他线程可以继续抢占，并以此达到用原子方式操作共享资源的目的。
- 在使用临界区时，一般不允许其运行时间过长，只要进入临界区的线程还没有离开，其他所有试图进入此临界区的线程都会被挂起而进入到等待状态，并会在一定程度上影响程序的运行性能。尤其需要注意的是不要将等待用户输入或是其他一些外界干预的操作包含到临界区。
- 虽然临界区同步速度很快，但却只能用来同步本进程内的线程，而不可用来同步多个进程中的线程。

## 临界区的保护（实现互斥）

- 互斥锁方式保护方法
  - 采用互斥锁需要更多资源，但是可提高的系统的实时性。
- 中断屏蔽方法
  - 因为 CPU 只在发生中断时引起线程或进程的切换，这样屏蔽 “总中断” 就能保证当前运行的进程/线程将临界区代码顺利执行完，从而保证互斥。
  - 屏蔽中断需要的资源更少，但是会影响系统的实时性。
  - 对于内核来说，屏蔽中断方式是很方便，但是将关中断的权力交给用户，则很不明智。（例如，若一个进程关中断后，不再开中断，则系统可能会因此终止。）
- 软件实现方法（设置标志位）
  - 在进入区设置和检查一些标志来标明是否有进程在临界区中。
  - 如果已有进程在临界区中，则在进程进入区通过循环检查进行等待，进程离开临界区后则在退出区修改标志。

## 如何减少临界资源保护的情形出现

- 将公共变量尽可能定义为 static
- 仅将必要的对外接口定义为 extern，其他接口都定义为 static
- 确认 extern 接口是否需要进行临界区保护并依据需要执行

## 如何检查项目中是否存在未进行保护的临界区

- 找出临界资源
- 查找所有引用该资源的代码
- 检查每个代码片段是否进行了必要的保护

## 进临界区(关全局中断)是否会影响数据的接收

- 嵌入式实时操作系统中，进入临界区的具体操作往往就是关掉系统的所有可以关闭的中断。
- 如果有一个外设刚刚要产生一个中断请求时，这时候恰好进入了临界区，disable所有中断
  - 那么这个外设的中断会不会被丢弃，是不是会有数据丢失了呢？
  - 比如串口的FIFO中断，我们设置成RXFIFO收到5个字时产生接收中断，那么上述情况发生时是不是这5个字就丢掉呢？
- 数据不会丢失。
  - 因为中断产后往往需要被清除，如果不清除中断产生标志位的话，系统会一直有这个中断到来。
  - 当上述RXFIFO中断将要产生时，系统刚刚关了全局中断，那好这个串口中断没有产生请求，但是也没被清除中断标志位；
  - 于是，等临界区退出后，它会继续产生这个中断请求，之后进入相应中断处理函数接收FIFO中的数据，并清除中断，这样一来数据就成功的被接收到了；
- 两个注意点：
  - 进临界区的时候要尽量短，否则系统可能会漏掉新来的数据。
  - 这个FIFO设置的不能太满，好让系统在退出临界区之前还可以接收一定数量的外设进来的数据。

# ■■■■■■■■■■■■■■■■■■■■■■

# 中断相关的问题

## 中断嵌套：不同 CPU 中断状态下切换到线程上下文说明

### 问题说明

关于中断嵌套情况下的，中断返回问题的疑问，情况如下：

- 首先发生较低优先级的中断
- 该低优先级中断被高优先级中断打断
- 在高优先级中断里释放信号量唤醒线程
- 在高优先级中断退出时，是否会出现直接切换到线程，而回到到低优先级中断的情况

### 问题解释

上述问题发生在允许中断嵌套的情况下，针对这个问题，需要讨论两类不同的 CPU 中断设计：

- 第一类 CPU：进入中断的同时会屏蔽同类型的中断，如 IRQ 或 FIQ，例如 cortex-A 系列
  - 对于第一类 CPU，这种 CPU 比较常见，例如 cortex-A 系列等。
  - 在 RT-Thread 操作系统中，进入中断后，**需要先将 `rt_interrupt_nest` 加一后再打开中断**，这就能确保即使该中断被更高优先级的中断打断返回，`rt_interrupt_nest` 的值仍然至少为 1，这就使得此时不会通过调度函数切换到线程上下文，而是必须等到最低一级中断完全处理完后，进行中断退出时，才会切换回线程状态。
  - 通过这种方式可以保证，如果中断没有完全退出，是不可能越过低级别的中断处理，直接切换到线程状态的。

- 第二类 CPU：进入中断时不会屏蔽中断，这种 CPU 会在进入中断时会自动保存一部分程序现场，例如 cortex-M系列
  - 对于第二类 CPU（例如 cortex-M），CPU 必须提供自动保存一部分上下文的功能，以避免中断被打断，造成现场破坏的问题。
  - 以 cortex-M 系列 CPU 为例，其线程切换总是在 `pendSV` 中进行，而 `pendSV` 是一种异常级别最低的异常。
  - 因此，从硬件机制上保证了线程的切换操作只能在所有的中断都退出的情况下进行。也就不可能发生中断没有完全退出而直接切换到线程的情形。

# 定时、延时相关问题

## 定时器中断累计误差问题

### 现象： 

- 以一个自动重装载的定时周期为 1ms 的定时器为例，如果在其发生中断的时候，中断正好被关闭了，那么定时器中断会在下次系统中断被打开之后发生。那么这种情况是否会造成累计误差呢？也就是说会不会造成以后每次的中断都向后偏移呢？
- 由于定时器是自动重装载的，当其中断被触发后，将会在中断控制器的相应寄存器位置位。
- 此时如果系统中断没有被打开，那么此中断就被 pending 直到中断重新被打开。
- 同时，定时器已经开始了第二次的计数，也就是说，无论中断处理函数是否被执行，定时器都会重新开始计时，开始在指定时间触发下次中断。

### 结论

- 自动重装载定时器的中断处理函数被短暂延误，并不会出现累计误差。
- 但是要注意，关闭中断时间太长，可能会导致丢中断的情况。
- **注意：**如果中断关闭的时间超过了定时周期 1ms，那么就会出现丢中断的情况，即连续尝试触发两次中断，但是不会叠加在中断控制器的使能位上，而是只会在系统中断开启后执行一次中断处理函数，这种情况会导致系统工作异常。

# ■■■■■■■■■■■■■■■■■■■■■■

# 内存 相关问题

## 内存非对齐访问问题 1：CPU 和 编译命令的配置

- CPU是否开启非对齐访问检查

- 编译命令禁用/使能非对齐内存访问选项

### 现象： 

- 系统中的结构体数据，如果添加了 `__packed` 属性，则会以紧凑的方式进行内存排布，此时其中的一些数据在内存中的排布就是非对齐的。

- 在程序运行时，如果系统不允许非对齐访问，此时对该结构体中的非对齐数据进行访问，则会出现 **data abort** 的错误。

- 如果在编译和链接时添加 `-mno-unaligned-access` 不支持非对齐内存访问选项，将会告诉编译器，生成操作这些非对齐数据指令，需要一个字节一个字节地读取，然后将结果拼凑成最终的数据。用这种方式操作数据降低了数据的访问效率，但是可以避免出现非对齐访问错误。

  - 如果关闭了非对齐访问检查，此时 CPU 访问非对齐数据将不会报错，在底层硬件实现时，可能会将一次访问拆成多次对齐访问来实现，但是在软件层是不感知的。尽管如此，还是降低了数据的访问效率。

  - 在 armv7 中可以开启或者关闭非对齐访问检查，例如使用如下指令关闭非对齐访问检查：

    ```assembly
        /* disable the data alignment check */
        mrc p15, 0, r1, c1, c0, 0
        bic r1, #(1<<1)
        mcr p15, 0, r1, c1, c0, 0
    ```

- 另外，一些**强序内存**（例如设备内存）是不支持非对齐访问的。

### 非对齐访问参数测试

- 编译如下源码：

```c
struct st_a {
    char a;
    int  b;
} __attribute__((packed));

int get_b(struct st_a *p)
{
    return p->b;
}
```

- 第一种情况：不支持（禁用）非对齐访问

  - 编译命令：

    ```
    arm-none-eabi-gcc.exe -S arm.c -o arm_no_unaligned_access.s -O2 -mno-unaligned-access -mcpu=cortex-a7
    ```

  - 实验结果如下：可以清楚地在汇编在代码中看到，如果开启了禁止非对齐访问，在操作非对齐地址的数据时，读取了多次，每次只读取一个字节。

    ![内存对齐问题__编译采用_不支持（禁用）非对齐访问](figures/内存对齐问题__编译采用_不支持（禁用）非对齐访问.png)



- 支持非对齐访问

  - 编译命令：

    ```
    arm-none-eabi-gcc.exe -S arm.c -o arm_no_unaligned_access.s -O2 -mno-unaligned-access -mcpu=cortex-a7
    ```

  - 实验结果如下：

    ![内存对齐问题__编译采用_支持非对齐访问](figures/内存对齐问题__编译采用_支持非对齐访问.png)

### 解决方案： 

如果系统中出现了非对齐访问错误，则要从两方面入手检查：

- cpu 是否开启了非对齐访问检查，如果开启了该检查，那么出现非对齐访问就会出现 data abort。
- 编译时，是否告知编译器帮忙处理非对齐访问的问题，如果告诉编译器，目标机器不允许非对齐访问，那么编译器发现将要访问非对齐的地址时，会执行单字节的访问指令，进而避免非对齐访问错误。
- 对于 AARCH64 架构。 
  - 编译器的配置项为 `-mstrict-align`，该选项会使得编译器将非对齐的多字节访问指令拆分为多个单字节访问指令，以避免出现非对齐访问异常。
  - 不加该选项生成的汇编代码，可以看出半字访问被修改成了单字节访问，从而避免了非对齐异常。



## 内存非对齐访问问题 2：sdio 存储器的文件系统的读写过程

### 现象： 

- 问题出现在基于 `sdio` 的存储器的文件系统的读写过程中。
- C lib 库中的 `memcpy` 函数的行为与 rt-thread 中 `rt_memcpy` 不一致，可能导致的非对齐访问错误
  - 如果使用了 C lib 库中的 `memcpy` 函数导致的硬件错误

### 解决方案： 

- 使用 RT-Thread 提供的  `rt_memcpy` 来代替 C 库中的  `memcpy` 函数
- 使用 RT-Thread 提供的 `rt_memcpy`，该函数会判断源地址和目标地址是否为四字节对齐。如果不是四字节对齐，那么将会尝试使用单字节的方式进行数据拷贝，这样做避免出现非对齐访问错误。但是使用单字节拷贝的情况会降低系统的效率，需要重新考虑和完善。
- 对于 SDIO 驱动而言，其传输所使用的内存地址是否支持非对齐，是要仔细考虑的地方。





# ■■■■■■■■■■■■■■■■■■■■■■

# 堆栈溢出 

## 现象： 

- 在基于实时内核的应用中，每个任务都需要自己的堆栈。任务所需堆栈的大小取决于应用程序。如果堆栈大于任务要求，则会浪费内存。如果堆栈太小，堆栈可能溢出。 

- 一些CPU，比如基于 ARMv8M 架构的 CPU，内置了堆栈溢出检测机制。然而，该特性并不能帮助确定合适的堆栈大小，它只是防止堆栈溢出的负面后果。 

 

## 解决方案： 

- 堆栈分配时，首先为任务堆栈分配更多空间，然后在已知最坏情况下运行应用程序，监视实际堆栈使用情况。 
- 我们可以通过分配更多内存来减少堆栈溢出的机会，通常需要 **25-50%** 的额外堆栈空间。 
- 线程栈大小可以这样设定
  - 对于资源相对较大的 MCU，可以适当设计较大的线程栈。
  - 也可以在初始时设置较大的栈，例如指定大小为 1K 或 2K 字节，然后在 FinSH 中用 list_thread 命令查看线程运行的过程中线程所使用的栈的大小，通过此命令，能够看到从线程启动运行时，到当前时刻点，线程使用的最大栈深度，而后加上适当的余量形成最终的线程栈大小，最后对栈空间大小加以修改。

# ■■■■■■■■■■■■■■■■■■■■■■

# 代码优化相关的问题

## 代码二级优化问题

### 现象： 

- 发现代码二级优化后，线程初始化异常，程序无法正常运行下去。

### 原因分析： 

- 出现这种问题的原因是对汇编相关的编程规范不够熟悉，
- 代码不支持二级优化的问题是由于先前支持 VFP 时留下的隐患，当用汇编实现 `vfp` 的栈初始化函数时，不清楚要遵循 **C 语言调用汇编语言的编程规范**，使用了 R5 R6 寄存器但是却没有保存并恢复他们的状态，这就导致了调用该函数的上下文被破坏，导致一些赋值异常，进而导致线程初始化异常，线程无法正常运行。
- 为什么问题只在二级优化的时候出现呢？
  - 经过测试发现当使用 O0 优化时，编译器在翻译代码时只只用了 R0-R3 寄存器，没有使用到我实现函数中所用到的 R5 R6，因此这个问题被掩盖了。
  - 当编译器开启二级优化时，**优化后的汇编代码使用了 R4-R11 等这些寄存器**，由于我在函数中改掉了这些寄存器，因此造成了编译器输出的汇编代码运行异常。

### 结论： 

- 这说明，在不同的优化等级下，编译器给出的机器码可能会选择不同的寄存器来完成相同功能的操作。
- 自己编写的汇编代码对于编译器来说是无法感知的，因此可能会破坏编译器规划的程序流，进而导致程序运行失败。

