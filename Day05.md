## ThreadPool Learning Day05

涉及：如何编译并执行c++程序、如何进行性能测试、使用objdump反汇编、volatile关键字。

第五天：

1.编译并运行昨天复现的线程池代码

2.进行性能测试：对比线程池与直接创建线程的性能差异

写两个测试的函数：

main.cpp:

```c++
#include "ThreadPool.h"
#include <iostream>
#include <chrono>

void simple_task(int id) {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    std::cout << "Task " << id << " completed by thread "
              << std::this_thread::get_id() << std::endl;
}

int compute_task(int x) {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    return x * x;
}

int main() {
    std::cout << "--- ThreadPool Pool Test ---" << std::endl;
    ThreadPool pool(4); // Create a thread pool with 4 threads
    std::vector<std::future<void>> futures;
    
    for(int i = 0; i < 10; ++i) {
        futures.emplace_back(
            pool.enqueue(simple_task, i)
        );
    }
    
    std::cout << "-- Compute Task Test --" << std::endl;
    std::vector<std::future<int>> results;
    for(int i = 0; i < 5; ++i) {
        results.emplace_back(
            pool.enqueue(compute_task, i)
        );
    }
    
    for(size_t i = 0; i < results.size(); ++i) {
        std::cout << "Result of compute_task( " << i << " ) = " << results[i].get() << std::endl;
    }
    
    std::cout << "All tasks completed." << std::endl;
    return 0;
}
```

:one: get_id()

其功能是获取当前线程的ID，每个线程都有自己的独一无二的ID

补充说一点：thread构造函数的参数，如果使用引用传递，那么需要加std::ref

```c++
#include <functional> // 需要包含这个头文件

void modify_data(int& value, std::string& text) {
    value *= 2;
    text += " modified";
}

int main() {
    int x = 100;
    std::string msg = "test";
    
    // 使用 std::ref 传递引用
    std::thread t(modify_data, std::ref(x), std::ref(msg));
    t.join();
    
    std::cout << "x: " << x << ", msg: " << msg << std::endl;
    // 输出: x: 200, msg: test modified
}
```

接着，编译：

```bash
g++ -std=c++20 -pthread main.cpp -o thread_pool_test
```

:one:g++

g++是GNU C++编译器，它是GCC（GNU Compiler Collection）的一部分

:two:-std=c++20

告诉编译器使用C++20标准来编译代码

:three:-pthread

表示链接POSIX线程库，用来链接多线程相关的库文件以及定义必要的预处理宏

在Linux系统上编译多线程程序时必须加上这个选项

:four:-o

main.cpp 源代码文件

-o thread_pool_test 指定输出文件名

-o直接生成最终的可执行文件



最终我们可以看到目录下生成了thread_pool_test.exe可执行文件

运行：
```bash
./thread_pool_test
```

结果：

![image-20250918134049630](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250918134049630.png)

注意到Task2的完成语句还没有输出完，Task0的完成语句已经抢先输出了

这是正常现象，因为输出完成语句不是原子操作，不同线程可能在不同步骤之间交错进行，导致输出混乱

另外，id为124881284875840的线程处理了多个任务，以及不同的线程ID在工作，这些都证明了线程池在正常工作

解决输出竞争，我们可以加锁

```c++
#include<mutex>

std::mutex cout_mutex;

void simple_task(int id) {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    {
        std::lock_guard<std::mutex>lock(cout_mutex);
   		std::cout << "Task " << id << " completed by thread "
              << std::this_thread::get_id() << std::endl;
    }//自动解锁
}
```

重新编译运行，结果如下：

![image-20250918140503201](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250918140503201.png)

:question:追问：为什么compute_task不会出现输出竞争？

答：因为compute_task的任务是计算，不是输出。输出是在主线程中顺序执行的。simple_task的任务是输出。



接下来进行性能测试：放1000个任务，每个任务是计算10000个表达式

```c++
#include "ThreadPool.h"
#include <chrono>
#include <iostream>
#include <vector>
#include <thread>
#include <math.h>

// 性能测试任务
void performance_task(int iterations) {
    double result = 0.0;
    for (int i = 0; i < iterations; ++i) {
        result += std::sin(i) * std::cos(i);
    }
}

// 线程池性能测试
void test_thread_pool(int task_count, int iterations_per_task) {
    ThreadPool pool(std::thread::hardware_concurrency());
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<std::future<void>> futures;
    for (int i = 0; i < task_count; ++i) {
        futures.emplace_back(pool.enqueue(performance_task, iterations_per_task));
    }
    
    for (auto& future : futures) {
        future.wait();
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "ThreadPool time: " << duration.count() << " ms" << std::endl;
}

// 直接创建线程性能测试
void test_direct_threads(int task_count, int iterations_per_task) {
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<std::thread> threads;
    for (int i = 0; i < task_count; ++i) {
        threads.emplace_back(performance_task, iterations_per_task);
    }
    
    for (auto& thread : threads) {
        thread.join();
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "Direct threads time: " << duration.count() << " ms" << std::endl;
}

int main() {
    const int TASK_COUNT = 1000;
    const int ITERATIONS = 10000;
    
    std::cout << "=== Performance Comparison ===" << std::endl;
    std::cout << "Tasks: " << TASK_COUNT << ", Iterations per task: " << ITERATIONS << std::endl;
    
    // 测试线程池
    test_thread_pool(TASK_COUNT, ITERATIONS);
    
    // 测试直接创建线程
    test_direct_threads(TASK_COUNT, ITERATIONS);
    
    return 0;
}
```

:one:std::thread::hardware_concurrency()

它返回当前系统支持的真正并发线程数量。不把线程数量写死。因为如果线程太少，无法充分利用多核CPU。如果线程太多，线程切换开销太大，性能反而下降

:two:std::chrono::high_resolution_clock

高精度时钟，提供当前系统可用的最高精度时间测量，具体精度取决于硬件和操作系统。主要用于性能测试、基准测试、超短时间间隔测量

:three:future::wait()

用于等待和该future绑定的任务完成。一个任务对应一个future。如果任务没有完成，会阻塞。

如果不调用wait()，那么在把所有任务放到队列后就开始计算end的值了，有很大概率任务都还没完成，因为执行任务的线程在另一个栈中，会造成极大误差。

:four:std::chrono::duration_cast

进行时间单位精度的转换。

```c++
// 高精度 → 低精度（截断）
auto t = std::chrono::nanoseconds(1234567);  // 1,234,567 纳秒
auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t);
// ms = 1 毫秒（1234567 / 1000000 = 1.234567 → 截断为1）

// 低精度 → 高精度（扩展）  
auto s = std::chrono::seconds(2);           // 2 秒
auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(s);
// ms = 2000 毫秒（2 * 1000 = 2000）
```

直接输出end-start，编译器不知道时间单位，会报错。需要明确转换到具体的单位

记录开始时间、结束时间、做差、比较，即可。

编译、运行，最终结果：

![image-20250918225416496](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250918225416496.png)

可以看到，线程池比直接创建线程快了28%左右。要知道，std::sin和std::cos是非常昂贵的浮点三角函数计算，任务执行开销是巨大的，线程管理开销就相对微小了。这就好比法拉利和和自行车在拥挤的街道中行驶，法拉利虽然启动快、极速高，但在漫长的行驶过程中，它的优势被拥挤的人群稀释了。

为了凸显线程池的优势，我们换一个轻量级的任务，让任务数量再庞大一点，让TASK_COUNT=10000

```c++
void lightweight_task(int iterations) {
    volatile int x=0;
    x+=1;
}
```

注意到编译器在volatile位置发出了警告，因为x+=1不是原子操作

volatile告诉编译器：这个变量可能被外部修改（以编译器无法预知的方式），不要优化

但是volatile不保证原子性（一个线程写到一半时，另一个线程可能读到中间状态）、内存可见性（不能保证一个线程的写入被另一个线程立即看到，具体可见Day02的笔记，一个操作至少要分LOAD、EXECUTE、STORE三个步骤）、线程安全性，多个线程同时修改volatile变量仍然会产生数据竞争

什么叫编译器优化？

编译器会帮我们“改写”代码，让它跑得更快，但保持相同的行为

```c++
// 我们写的代码
int x = 5;
int y = x * 2;
std::cout << y;

// 编译器优化后可能变成
std::cout << 10;  // 直接计算结果，省去中间变量
```

优化级别：

```bash
g++ -O0  # 不优化，调试用，保持代码原样
g++ -O1  # 基本优化
g++ -O2  # 推荐优化级别
g++ -O3  # 激进优化，可能改变行为
```

这个可以在写编译语句的时候，在g++后面加上

```bash
# 测试不同优化级别的性能差异
g++ -O0 -std=c++20 -pthread benchmark.cpp -o bench_o0
g++ -O1 -std=c++20 -pthread benchmark.cpp -o bench_o1  
g++ -O2 -std=c++20 -pthread benchmark.cpp -o bench_o2
g++ -O3 -std=c++20 -pthread benchmark.cpp -o bench_o3

# 分别运行测试
time ./bench_o0
time ./bench_o1
time ./bench_o2
time ./bench_o3
```

-O0优化是默认选项，编译器会尝试最快的编译速度，生成最直接的机器代码

行为：变量通常直接存储在内存中，而不是寄存器中。代码的执行流程和你在源代码中写的完全一样，每一步调试都能看到预期的变量值

-O1,-O2,-O3是递增的优化级别，数字越大优化越激进，编译时间越长，但生成的代码性能通常越好

优化手段：内联函数、删除未使用的代码和变量、指令重排、使用寄存器存储变量等

g++ -O0可以加上-g用来生成调试信息

```bash
g++ -O0 -g -std=c++20 -pthread main.cpp -o program_debug
```

-g 告诉编译器在生成的可执行文件中嵌入调试信息。



另外，我们可以使用objdump反汇编（功能类似于翻译），查看汇编指令

在另一个文件夹新建一个main.cpp

```c++
// main.cpp
#include <iostream>

int add(int a, int b) {
    int sum = a + b; // 第4行：设置一个断点在这里
    return sum;
}

int main() {
    int x = 5;
    int y = 10;
    int result = add(x, y); 
    std::cout << "The result is: " << result << std::endl;
    return 0;
}
```

编译：
```c++
g++ -O0 -g main.cpp -o debug_program
```



objdump几乎总是默认已经安装在Ubuntu系统上的，查看其版本：

```bash
objdump --version
```

如果没安装：

先确保获取最新的软件信息

```bash
sudo apt update
```

再安装binutils

```bash
sudo apt install binutils
```

最后再用--version验证一下即可



### objdump基本语法：

```bash
objdump <options> <your_executable_file>
```

1.反汇编并显示源代码（需要-g编译）

```bash
objdump -S my_program
```

2.只反汇编特定函数

例如只想看main函数

```bash
objdump -S my_program | grep -A 20 'main.*:'
```

3.查看可执行文件的段头信息

```bash
objdump -h my_program
```

4.查看符号表

```bash
objdump -t my_program
```

解释一下：

:one:|

|这个竖线符号叫做管道。其作用是将前一个命令的输出，作为后一个命令的输入。就好像水管，把从左边水龙头出来的水引到右边的过滤器（下一个命令）中。

objdump -S my_program这个命令会输出非常多内容（整个程序的汇编和源代码，如下图）

![image-20250918235752764](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250918235752764.png)

我们不想手动在这么多内容里找main函数，所以用管道把它输出的所有文本直接传给grep命令去处理

如果没有管道，我们就得先把objdump的输出保存到一个文件里，然后再用grep去搜索这个文件，需要两步操作。使用管道一步搞定

:two:grep

全称：**g**lobal search a **r**egular **e**xpression and **p**rint（全局搜索正则表达式并打印）

grep是一个文本搜索工具。在上面的例子中，它从管道那里收到了objdump产生的所有文本，它的任务是在里面寻找包含字符串的行

所以我们搜索这个模式就能找到main函数的起始位置

例如：

```bash
# 在 file.txt 中搜索所有包含 "hello" 的行
grep "hello" file.txt
```

:three:main.*

在objdump -S的输出中，一个函数（比如main函数）的开始是这样标记的：<main>:

为什么我们搜索的字符串为什么不能是"<main>:"或者<main>:？

< 和 >在Shell和grep中有特殊含义。在Linux shell中它们两个是重定向操作符

< file 表示从file中读取输入，另一个是将输出写入到file（覆盖原文件）

当我们把搜索模式用双引号或单引号括起来时，相当于告诉Shell里面的内容是一个整体，不要把它们当作特殊符号处理

所以，如果我们输入：

```bash
grep -A 20 <main>: my_program
```

Shell会将<main>解释为一个文件名，因为它觉得<是输入重定向

为什么加了引号可能还是不成功？

原因一：函数名被改编

C++编译器为了支持函数重载，会对函数名进行改编。一个简单的main函数可能不会被改编，但其它函数会

源代码中的函数：

```c++
void foo(int)
```

objdump输出的名字：
```bash
<_Z3fooi>
```

几种搜索方式：

方法一：使用转义字符

最规范地写法是使用单引号，它能防止Shell解释所有特殊字符

```bash
objdump -S my_program | grep -A 20 '<main>:'
```

方法二：使用更宽松的模式，只搜索包含main和冒号的行

```bash
# 搜索包含 "main" 和 ":" 的行
objdump -S my_program | grep -A 20 'main.*:'
```

.*是正则表达式，表示”任意数量的任意字符“

'main.*:'的意思是：匹配所有包含main，后面跟着任意数量的任意字符直到一个冒号的行

这种方式可以解决函数名被改编的问题，缺点是可能找到多个结果

方法三：先找到地址，再反汇编特定部分

1.先找到main函数的地址

```bash
objdump -S my_program | grep 'main.*:' 
```

假设输出是：0000000000001153 <main>：那么1153就是main函数的起始地址（十六进制）

2.反汇编这个函数

```bash
objdump -S my_program --start-address=0x1153 | head -n 30
```

--start-address=0x1153告诉objdump从内存地址0x1153开始反汇编

| head -n 30 表示用管道将输出传给head命令，只显示前30行。因为一个函数结束时会有ret命令，我们不需要看整个程序



这里我们使用了正则表达式应对函数名被改编的情况，但是缺点就是可能找到多个结果。好消息是，我们可以知道函数改编后的名字：

```bash
nm my_program | grep myFunction
```

假设输出是：0000000000001169 T _Z10myFunctioni

我们就得到了精确的改编后的名字_Z10myFunctioni

得到了改编后的名字，我们就可以自由使用上述三种方法了！



:four:-A 20

A是After的缩写

-A 20表示：找到匹配'main.*:'的那一行后，把这一行以及它后面的20行都打印出来

为什么是20？因为一个函数通常不会只有一两行汇编指令，我们想看到这个函数比较完整的汇编代码。20是一个估计值，可以改成30或50等数字。

类似地：-B N：打印匹配行之前的N行（Before）

-C N：打印匹配行及前后各N行（Context）



**总结一下这条命令的完整执行流程：**

**1.执行objdump -S my_program**

**objdump开始反汇编my_program程序，将交错显示源代码和汇编代码的大量文本输出到屏幕上**

**2.管道传输：|**

**这些文本并没有真正显示在屏幕上，而是被管道截获，发送给了右边的grep命令**

**3.执行右边：grep -A 20 'main.*:'**

**grep逐行扫描，寻找包含main且后面跟着任意数量的任意字符，直到一个冒号的行**

**一旦找到，它就把这一行以及紧接着的后面20行抓取出来**

**4.最终输出：**

**grep将它抓取到的1+20行文本打印到终端屏幕上**

objdump的基本语法就先到这里，接着继续



编译过后，使用objdump

```bash
objdump -S my_program_release > disassembly_no_source.txt
```

可以在vscode中看到生成了一个新的文件，点进去就可以看到disassembly_no_source.txt文件了

当然也可以使用nano或者vim查看

```bash
nano disassembly_no_source.txt
```

```bash
vim disassembly_no_source.txt
```

下图是使用nano查看的结果

![image-20250918235604474](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250918235604474.png)

按Ctrl+X即可退出



反汇编的内容补充到这里，回归到性能测试

volatile告诉编译器：这个变量可能会被意外改变（硬件、信号等）

而atomic则是告诉编译器：这个变量可能会被其它线程改变

```c++
void lightweight_task(int iterations) {
    std::atomic<int>x=0;
    for(int i=0;i<iterations;++i) {
        x.fetch_add(1,std::memory_order_relaxed);
    }
}
```

**现在x是一个原子变量，fetch_add的作用是：以原子操作的方式，给原子变量x的值加上1，并返回它做加法运算之前的值。注意，是做加法运算之前，不是之后**

举个例子：

假设原子变量的当前值是5

1.fetch_add的行为：

```c++
int old_value = x.fetch_add(3); // 原子操作：加上3
// 操作完成后：
//   x 的新值变成了 8 (5 + 3)
//   old_value 的值是 5 (加法之前的值)
```

2.如果想要返回新值：

```c++
int new_value = (x += 3); // 这是一个复合操作，但对于原子变量也是原子的
// 操作完成后：
//   x 的新值变成了 8
//   new_value 的值是 8 (加法之后的值)
```

注意，对于原子变量，x+=i也是一个原子操作。但对于非原子变量就不是了，参见Day02的笔记

如果我们自己实现一个”读取并增加“的操作，需要分两步：

1.int temp=x  读取当前值（旧值）

2.x=temp+1 写入新值

在多线程环境下，这两步之间可能会被其它的线程打断。fetch_add的功效在于将这两步变成了一步。fetch_add的重点是获取。就好比你从货架上拿走了商品，然后那个货架上又补了新的（+=n），你拿到的是你最终取走的那个商品。

std::memory_order_relaxed 只保证操作本身的原子性和修改的顺序一致性，放弃了对其它内存操作的任何保证。可以理解成只管自己不顾全局的独行侠。它保证自己的任务能原子地完成，但不在乎这个任务何时被其它线程看到，也不在乎这个任务和自己队伍里其它任务（同一线程的其它内存操作）的先后顺序

这个内容放到明天讲，不然似乎又要跑题了



改了任务后的性能对比：

![image-20250919013709955](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250919013709955.png)

快了将近50%，毕竟一个任务循环10000次，也还是比较重的。

改成循环10次：

![image-20250919014720716](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250919014720716.png)

快了接近10倍





