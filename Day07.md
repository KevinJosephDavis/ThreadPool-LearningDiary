## ThreadPool Learning Day07

1.用std::memory_order_relaxed制造一个问题，使用gdb进行调试，接着用release-acquire修复问题

2.了解std::mutex底层实现：回答三个问题：

（i）std::mutex底层是怎么实现的？它用了什么内存序？

（ii）除了用锁，还能怎么实现线程同步？

（iii）什么时候该用relaxed？什么时候必须用seq_cst？



建议跳过“使用relaxed制造一个问题”这一章节



### 使用relaxed制造一个问题

```c++
#include <iostream>
#include <atomic>
#include <thread>

std::atomic<int>data(0);
std::atomic<bool>ready(false);

void producer() {
    data.store(42,std::memory_order_relaxed);
    ready.store(true,std::memory_order_relaxed); // 问题所在
}

void consumer() {
    while(!ready.load(std::memory_order_relaxed)){
        // 忙等待
    }
    std::cout<<"data is:"<<data.load(std::memory_order_relaxed)<<std::endl;
}

int main() {
    for(size_t i=0;i<1000;++i) {
        data=0;
        ready=false;
        std::thread t1(producer);
        std::thread t2(consumer);
        t1.join();
        t2.join();
    }
    return 0;
}
```

编译，先不使用优化：

```bash
g++ -g -O0 -std=c++20 main.cpp -o bug -pthread
```

```bash
# 在bash中，运行10000次或直到出错
count=0; while ./spsc_bug; do ((count++)); echo "Run $count: OK"; done; echo "Failed after $count runs!"
```

这是一条组合命令，用分号连接了多个部分。为了更清晰，我们把它写成多行

```bash
count=0
while ./bug
do
    ((count++))
    echo "Run $count: OK"
done
echo "Failed after $count runs!"
```

:one:count=0

作用：初始化一个名为count的变量，并将其值设置为0

这个变量用来记录我们的程序成功运行了多少次

:two:while循环的条件是执行./bug这个命令，也就是运行我们编译好的程序

一个命令执行成功与否决定了它的退出状态。如果成功执行，那么退出状态为0，反之为非0.

因此，这行的意思是，只要./bug这个命令成功退出（返回0），就继续执行循环体内的操作

:three:do ... done

这是while循环的循环体，只要while循环的条件为真，就会执行do和done之间的命令

:four:((count++))

让count自增1，((...))是bash中进行算术运算的方法

:five:echo "Run $count: OK"

在终端打印信息

echo：打印命令

"Run $count: OK"：要打印的字符串，其中$count会被替换为变量count当前的值。类似C语言中的%d

这条语句打印当前是第几次成功运行

:six:echo "Failed after $count runs !"

当while循环结束后，执行这条命令

什么时候结束？当./bug运行失败（即main函数返回非0值，或者程序崩溃、断言失败等），while的条件就不满足了，循环结束

循环接收后，打印这条消息，告诉我们程序在运行了count次后，终于失败了

所以，上面那段代码会一直运行下去，因为不管Data=0还是42，它都会return 0，while循环还会继续

因此我们需要修改一下，当data不等于42时，exit(1)

```c++
#include <iostream>
#include <atomic>
#include <thread>

std::atomic<int>data(0);
std::atomic<bool>ready(false);

void producer() {
    data.store(42,std::memory_order_relaxed);
    ready.store(true,std::memory_order_relaxed); //release
}

void consumer() {
    while(!ready.load(std::memory_order_relaxed)){
        // busy wait
    }
    int value=data.load(std::memory_order_relaxed);
    if(value!=42) {
        std::cerr<<"Error:value="<<value<<"but expected 42!"<<std::endl;
        std::exit(1);
    } else {
        std::cout<<"value is correct:"<<value<<std::endl;
    }
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();
    return 0; // 正确时才返回0
}
```

使用O3编译

```bash
g++ -g -O3 -std=c++20 main.cpp -o bug_o3 -pthread
```

再次运行

```bash
count=0; while ./bug_o3; do ((count++)); echo "Run $count: OK"; done; echo "Failed after $count runs!"
```

可惜运行了二十多万次都没有出现失败的情况，我们使用ThreadSanitizer（TSan）

编译，运行。出现了错误

![image-20250920175156757](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250920175156757.png)

有可能是GLIBC（GNU的C库）的版本和TSan运行时库可能存在兼容性问题。考虑使用LLVM的链接器ld.lld

在安装LLVM的工具链之前，先sudo apt update，发现进度一直为0

尝试将阿里云镜像替换为清华镜像

```bash
sudo sed -i 's/mirrors.aliyun.com/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list
```

但是进度依然为0

先看看能否访问外部IP

```bash
ping -c 4 114.114.114.114
```

![image-20250920175910032](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250920175910032.png)

这表明虚拟机完全没有外网连接

先关闭虚拟机

```bash
sudo poweroff
```

点击“编辑虚拟机设置”，找到“网络适配器”，选择NAT模式，并确保选择了“启动时连接”

它的作用是让虚拟机共享主机的IP地址上网，相当于主机是一台路由器，虚拟机是这个路由器下的一台设备

我这里一直是选上了

重新启动虚拟机



ps：虚拟机一直卡在关机界面或者开机界面的处理方法：

:one:通过任务管理器强制结束线程

在Windows主机上键入Ctrl+Shift+Esc打开任务管理器

在详细信息标签页中找到vmware-vmx.exe，这是主要的虚拟机进程。右键点击“结束任务”。如果vmware主界面也卡死了，那就找到vmware.exe，右键点击“结束任务”

:two:使用命令提示符关闭

进入cmd，查找vmware进程

```cmd
tasklist | findstr "vmware"
```

使用taskkill强制结束

```cmd
taskkill /f /im vmware-vmx.exe
taskkill /f /im vmware.exe
```



重启后，ping一下谷歌的IP 8.8.8.8

```bash
ping -c 4 8.8.8.8
```

发现网络连通，继续安装lld链接器



首先更新软件源列表

```bash
sudo apt update
```

安装lld-14链接器

```bash
sudo apt install lld-14
```

发现有报错：403 Forbidden

这意味着清华镜像源的服务器拒绝了下载这个特定软件包的请求

我们换回Ubuntu官方源来安装这个包

```bash
# 临时注释掉所有清华源，换回官方源
sudo sed -i 's|http://mirrors.tuna.tsinghua.edu.cn/ubuntu|http://archive.ubuntu.com/ubuntu|g' /etc/apt/sources.list

# 更新
sudo apt update

# 安装 lld-14
sudo apt install lld-14

# 安装成功后，恢复清华源
sudo sed -i 's|http://archive.ubuntu.com/ubuntu|http://mirrors.tuna.tsinghua.edu.cn/ubuntu|g' /etc/apt/sources.list
sudo apt update
```

检测安装是否成功：

```bash
lld-14 --version
```

![image-20250920210123286](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250920210123286.png)

上图表示已经安装成功，如果想看版本：

```bash
ld.lld-14 --version
```

在此之前，我们安装clangd，方法和安装lld-14一样

但是依然报错FATAL，使用了静态链接也依然不行，使用gdb调试竟然没有出现崩溃，而是正常结束了！可能是gdb的调试环境以某种方式影响了内存布局、线程调度或初始化顺序，恰好避免了触发TSan内部的那个致命错误

这个问题就到这里结束吧，没必要浪费时间在这里了。我们已经很清楚relaxed会造成什么问题、为什么会造成问题以及如何解决这个问题了



### std::mutex的底层实现

| 层级           | 组件              | 说明                                                         |
| -------------- | ----------------- | ------------------------------------------------------------ |
| 7.C++程序      | .cpp              | `#include<thread>`，`include<mutex>`                         |
| 6.C++标准库    | libstdc++.so(GCC) | 提供std::mutex，std::thread等类，它们是对下层Pthreads的C++封装 |
| 5.glibc        | libc.so.6         | 这是一个库，提供两套功能：<br>1.C标准库函数：如printf，malloc<br>2.POSIX系统接口：如pthread_mutex_lock.Pthreads是glibc的一部分 |
| 4.系统调用封装 | syscall()         | glibc调用诸如pthread_mutex_lock这样的函数，转换为对Linux系统调用的调用 |
| 3.Linux内核    | futex()           | 关键的系统调用，高效实现用户态同步                           |
| 2.硬件架构     | x86/ARM指令       | 内核使用cmpxchg等CPU原子指令实现futex                        |
| 1.物理硬件     | CPU、内存         | 最终的执行层                                                 |

所以，std::mutex的调用流程是：
1.我们调用std::mutex::lock()

2.调用C++标准库（libstdc++）实现的std::mutex::lock()，其内部直接调用pthread_mutex_lock()

3.调用glibc中的POSIX线程库提供的pthread_mutex_lock()函数，它的实现思路：

在用户态尝试使用原子指令快速获取锁。如果失败，则调用futex系统调用，请求内核将自己挂起

4.Linux内核处理futex系统调用，管理线程的挂起和环形队列



比喻理解：

Linux内核：市政府发电局

glibc：家里的总电闸和墙上的所有插座，它提供了：

1.标准的两孔/三孔插座（C标准库，如printf）

2.一个特殊的工业插座（POSIX接口，如pthread_mutex_lock）

C++标准库：一个高级的智能充电站，它必须插在墙上的工业插座上才能工作，它为C++程序提供了更漂亮更方便的USB-C接口（std::mutex）

程序：一个手机，插在智能充电站上充电



### std::mutex使用的内存序

std::mutex使用的内存序是std::memory_order_seq_cst

当线程调用lock()时，相当于acquire操作，保证了临界区内所有内存操作不会在它之前（昨天说的，在它后面的必然在它后面）。此时临界区开始

线程调用unlock()时，相当于release操作，保证了临界区内所有内存操作不会在它之后（在它前面的必然在它前面）。此时临界区结束

这样一来，当有线程解锁时，别的线程就知道这个线程完成了某个任务。具体是什么任务，别的线程不清楚、也不需要清楚。它只需要知道这个线程完成了这个任务，并且自己可以基于这个任务的结果做自己的事情

就好比开发项目，项目经理和甲方明确了需求，就和程序员进行交接。程序员并不需要知道项目经理和甲方的谈话内容，只需要基于项目经理给的需求开始干活。如果没有内存序，那么可能项目经理和程序员交接后，又和甲方谈话并且改动了需求，那么程序员就会根据错误的需求敲代码了，浪费了时间与人力。

:question:追问：调用lock相当于获取操作，获取的是什么？

答：获取的是单独使用某片内存的权限，或者说，是“我准备好接收 之前由任何线程 在此锁的释放操作之前 所做出的所有状态变更 的权限”

还是用一开始理解互斥锁的例子：进出房间。线程A拿到锁（拿到单独使用某片内存区域的权限），进入房间，将它的装修想法写在笔记本上（CPU缓存），对房间进行了装修（主存），解锁（释放单独使用某片内存区域的权限）。线程B调用lock()，进入房间，它必须要接受A装修的成果。

类似的，释放的是什么？释放的是修改成果。它的意思是：我在此临界区内所做的所有修改，现在已经全部完成并就绪了，现在我把这个“已就绪”的消息（通过释放锁）广播出去，这样别的线程进来装修，就一定能看到我的装修成果

类似地，看一下relaxed的问题。它的问题在于，它只保证“写下装修想法”（操作本身）是原子的，但不能保证这个想法被及时地付诸实践（写入主存）

例如：

线程A：

1.在笔记本上写：我要把墙刷成蓝色 -> `data.store(42,std::memory_order_relaxed)`

2.在门口挂牌：我完工了 -> `flag.store(true,std::memory_order_relaxed)`

线程B：

1.看到门口挂着“完工”的牌子 -> `flag.load(std::memory_order_relaxed) == true`

2.推门进去一看，墙还是白色的 -> `data.load(std::memory_order_relaxed)  == 0`

问题根源：因为用的是relaxed，CPU或编译器为了优化，可能会让线程A先挂牌，再去刷墙。线程B看到牌子就冲进去了，自然看不到最终的效果

std::mutex通过使用release-acquire，保证工人在挂上“完工”的牌子之前，所有装修的内容必须从笔记本同步到房间中（缓存刷入主存）。保证工人在进入房间时，能看到最新的装修成果（使缓存失效，从主存读取）

### 

用面试的方式回答：

std::mutex 的内部实现使用了最强的 std::memory_order_seq_cst 内存序。

**这是因为互斥锁的核心目标有两个：一是提供互斥性，保证一次只有一个线程执行临界区代码；二是提供内存同步，保证一个线程在临界区内所做的所有内存修改，对下一个获得锁的线程是立即可见的。**

**从内存序的角度看：**

**lock() 操作相当于一个 acquire操作**。它会建立一个**同步点**，确保当前线程在临界区内的**所有后续读写操作**都不会被重排到 lock() 之前。

**unlock() 操作相当于一个 release 操作**。它也会建立一个**同步点**，确保当前线程在临界区内的**所有先前读写操作**都不会被重排到 unlock()之后。

**当线程A调用 unlock()释放锁，线程B随后调用 lock() 获得锁时，这两个同步点之间会成功配对，建立起一个同步关系。这保证了线程A在临界区内的所有修改，对线程B在临界区内的所有操作都是可见的。**



事实上，我们前面所说的“从缓存中写入主存”的说法，在硬件层面上并非如此。写入主存太慢了

现代CPU很少会直接为了一个变量的可见性就把整个缓存行写回主内存。它有一套更高效的机制：

1. 缓存一致性协议（Cache Coherence Protocol）

   CPU核心们通过这个协议来维护所有缓存数据的一致性。

   当一个核心（线程A）要修改一个缓存的数据时，它会先通过协议让其他核心（线程B）中对应的缓存行失效（I状态）。

   然后线程A在自己的缓存中进行修改（变为M状态）。

   当线程B试图读取这个数据时，会发现自己的缓存是无效的，于是它会从线程A的缓存中直接拉取最新数据，而不是从慢得多的主内存中读取。

2. 内存屏障的作用：

   编译器或CPU的乱序执行可能会破坏上述流程的顺序。

   seq_cst 等内存序生成的内存屏障指令，其作用就是防止这种重排，并强制CPU完成当前的缓存一致性操作，然后再执行屏障后的指令。

所以，更准确的说法是：
unlock() 中的释放语义不仅确保修改完成，还确保所有相关的“缓存失效”或“数据广播”的指令都已完成。随后，lock() 中的获取语义确保之后的操作能看到这些同步后的结果。



### 其它实现同步的方法

| 机制                    | 原理                                             | 适用场景                                    |
| :---------------------- | :----------------------------------------------- | :------------------------------------------ |
| **原子操作**            | 利用CPU指令实现无锁的原子读、写、RMW操作         | 计数器、标志位、简单的数据结构              |
| **互斥锁**              | 通过让等待线程睡眠来避免忙等待                   | **通用**，保护复杂的临界区或数据结构        |
| **自旋锁（Spin Lock）** | 线程在等待时循环检查（忙等待），不进入睡眠       | 临界区非常短，且不想承受线程睡眠/唤醒的开销 |
| **信号量**              | 一个计数器，用于控制访问共享资源的线程数量       | 限制并发数、生产者-消费者模型               |
| **条件变量**            | 允许线程在某个条件不满足时主动阻塞并等待通知     | 复杂的等待/通知场景，如任务队列、等待资源   |
| **屏障**                | 让一组线程彼此等待，直到所有线程都到达某个执行点 | 并行计算，需要同步多个线程的阶段            |

这里提到了std::mutex的另一个优势，补充一下

std::mutex 的两大核心是：

1. **提供内存同步**（上文提及的，通过 release/acquire 语义）
2. **提供调度同步**（表格中提及的，让等待线程睡眠，避免忙等待）

这两者结合在一起，才构成了一个完整、高效的互斥锁



当一个线程尝试获取一个已经被锁定的 std::mutex 时，它会经历以下过程：

1. 快速路径：线程首先会尝试一次原子操作（比如一个 compare-and-swap），希望能快速获取锁。如果成功，它就进入临界区。这发生在用户态，非常快

2. 慢速路径：如果快速路径失败（锁已被占用），它就不会傻傻地循环（忙等待），而是会：

   通过一个系统调用（如 futex）陷入内核态

   内核将这个线程标记为睡眠状态，并将其从调度器的运行队列中移出

   内核记录下这个线程正在等待这个特定的锁

3. 解锁与唤醒：

   当持有锁的线程调用 unlock() 时，它同样会先走快速路径

   如果发现有人在等待这个锁，它就会通过系统调用通知内核

   内核会选择一个正在睡眠的等待线程，将其状态改为可运行，并放入调度队列

4. 被唤醒的线程：一旦被调度器选中，该线程会再次尝试获取锁。这次大概率会成功，然后它就可以继续执行了

优点：

避免忙等待：如果让线程循环尝试，它会白白浪费整个CPU时间片，占着CPU核心却不做任何有用功，导致CPU利用率100%，其他任务也无法运行

节省资源：让等待线程睡眠，CPU核心就可以去执行其他真正就绪的线程，极大地提高了整个系统的效率和资源利用率



用前面的比喻：

一个房间（临界区）门口有一个智能锁（std::mutex）

1. 线程A来了，刷卡（原子操作）成功，进入房间装修
2. 线程B也来了，刷卡失败。它不会傻傻地站在门口不停地刷卡（忙等待）。而是在旁边的休息室（内核的等待队列）里找个沙发坐下睡觉（睡眠）
3. 线程A干完活出来时，不仅真的把房间装修好了（内存同步），还会按一下休息室的门铃（系统调用）
4. 门铃吵醒了线程B（唤醒），它醒来后再次去刷卡，这次成功进入

所以，一个完整的 std::mutex既保证了房间状态的正确交接（内存序），也保证了等待者的高效调度（睡眠/唤醒）



### 什么时候用relaxed？什么时候用seq_cst？

使用std::memory_order_relaxed的场景：

操作是独立的：例如一个全局的计数器，多个线程同时递增它。我们只需要关注最终结果，并不在乎哪个线程在哪一时刻加了1

```c++
std::atomic<int>cnt(0);
cnt.fetch_add(1,std::memory_order_relaxed);
```

使用std::memory_order_seq_cst的场景：

1.不知道该用什么的时候：正确性高于性能

2.需要多个原子变量之间保持一致的顺序（保证每个线程看到的顺序是一样的）

3.需要与代码中其它的seq_cst操作交互

使用std::memory_order_release和std::memory_order_acquire的场景：

保护一段临界区：用release store，用acquire load

```c++
// 线程 1
data = 42; // 要保护的数据
flag.store(true, std::memory_order_release); // 发布：保证data=42不会重排到此之后

// 线程 2
if (flag.load(std::memory_order_acquire)) { // 获取：保证之后的读操作不会重排到此之前
    assert(data == 42); 
}
```





