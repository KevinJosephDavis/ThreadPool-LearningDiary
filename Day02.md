## ThreadPool Learning Day02

:question:3个问题：

:one:构造函数做了什么？

:two:有哪些关键语法点？

:three:这么设计的优势？

构造函数

```c++
class ThreadPool {
public:
    ThreadPool(size_t);
    template<class F, class... Args>
    auto enqueue(F&& f, Args&&... args)
        -> std::future<typename std::result_of<F(Args...)>::type>;
    ~ThreadPool();
private:
    // need to keep track of threads so we can join them
    std::vector< std::thread > workers;
    // the task queue
    std::queue< std::function<void()> > tasks;

    // synchronization
    std::mutex queue_mutex;
    std::condition_variable condition;
    bool stop;
};
```

```c++
// the constructor just launches some amount of workers
inline ThreadPool::ThreadPool(size_t threads)
    : stop(false)
{
    for(size_t i = 0; i < threads; ++i)
        workers.emplace_back(
            [this]
            {
                for(;;)
                {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(this->queue_mutex);
                        this->condition.wait(lock,
                            [this]{ return this->stop || !this->tasks.empty(); });
                        if(this->stop && this->tasks.empty())
                            return;
                        task = std::move(this->tasks.front());
                        this->tasks.pop();
                    }
                    task();
                }
            });
}
```

宏观上不难理解

1.构造函数创建了threads个线程，把它们加入到workers中

```c++
:stop(false);//设置工厂不停止运行
```

2.给工人分配任务

```c++
for(size_t i=0;i<threads;++i)
    workers.emplace_back(/*每个工人的工作手册*/);
```

3.告诉工人开工和下班的信号

4.领取任务、执行任务

比喻理解：

构造函数创建并启动了一条工厂生产线

threads：雇佣的工人数量

workers：工人团队（线程集合）

tasks：待处理的任务队列

condition：开工/下班的信号，类似于工长的哨子（通知机制）

接着分析语法。

:one:inline

inline的功能之一，就是将整个函数体代码嵌入到调用点然后展开。

```c++
inline int add(int a,int b) {
    return a+b;
}
int main() {
    int x=5;
    int y=3;
    int ans=add(x,y);//调用点
    return 0;
}
```

编译器可能会将代码转换成这样：

```c++
inline int add(int a,int b) {
    return a+b;
}
int main() {
    int x=5;
    int y=3;
    int ans=x+y;//add的函数体被嵌入到这个位置（调用点）。直接替换为函数体代码。
    return 0;
}
```

不过，inline只是建议，编译器可以忽略，而且编译器未必真的做文本替换。

但是，线程池构造函数的实现里的inline不是为了代码展开优化。其目的是：防止多个cpp文件包含到同一头文件时产生多重定义错误（因为整个实现都在头文件中）。inline的作用就是告诉链接器：这个函数定义可以在多个编译单元中出现，链接器请处理好重复定义的问题。

什么叫编译单元？编译单元=一个.cpp文件+它包含的所有头文件。

更准确地说，一个源文件（.cpp文件）经过预处理（所有#include和宏）后得到的完整代码，就是编译单元。

假设一个目录下有三个文件：main.cpp\utils.cpp\ThreadPool.h.

编译单元1：main.cpp+它包含的所有头文件（展开iostream/ThreadPool.h...的内容)

编译单元2：utils.cpp+它包含的所有头文件（再次展开iostream/ThreadPool.h...的内容）

编译过程：1.每个.cpp文件单独编译成.o目标文件；2.所有.o文件合并成最终程序

链接器发现main.o和utils.o中都包含ThreadPool的定义（实现），报错。加了inline，链接器知道这是允许重复的定义，选择一个使用，不报错。

比喻理解：每个单元是一个独立的工厂车间。每个车间都需要完整的工作手册（头文件）。如果没有inline，每个车间都会因为自己拿到了完整的工作手册而声称自己是总指挥部。有了inline，大家知道这些只不过是相同的副本，可以协调工作。

这么一来，inline就可以支持头文件only了。

头文件only，即整个库的实现都在头文件中，没有对应的.cpp文件。

头文件only的优点：

1.使用简单：只需要#include"ThreadPool.h"，无需编译链接.cpp文件（对开发者来说工作变轻了，不用处理复杂的编译链接）

2.跨平台：避免二进制兼容问题

3.模板友好：**模板必须在头文件中实现**

缺点：

1.编译时间长：每次包含都会重新编译整个实现（对编译器来说工作变重了，每次都要编译相同代码）

2.代码暴露：实现细节完全暴露给用户

为什么说模板必须在头文件中实现？

错误示范：

```c++
// ThreadPool.h （声明）
template<typename T>
class ThreadPool {
public:
    void enqueue(T task);  // 模板声明
};

// ThreadPool.cpp （定义）
template<typename T>
void ThreadPool<T>::enqueue(T task) {  // 模板定义
    // 实现代码...
}
```

因为模板不是普通的代码。编译器看到模板时，并不立即生成代码。只有当使用具体类型（实例化）时，才生成实际代码。编译器需要在看到模板使用的同一地方看到模板定义。因此，线程池代码中的构造函数的整个实现必须在头文件中，这也是为什么这个项目是头文件only，以及为什么需要使用inline.

不过，在C++20中引入了模块，解决了这个问题：

```c++
// ThreadPool.cpp （使用模块）
export module ThreadPool;

export template<typename T>
class ThreadPool {
public:
    void enqueue(T task) {
        // 实现可以在.cpp中
    }
};
```

此外，在C++17之后，类内定义的成员函数隐式内联。这个线程池项目可能是为了兼容旧标准或显式表达设计意图。

头文件only和大量使用inline是紧密相关的。头文件中有大量的接口，会被多个.cpp文件include，那么就有多个编译单元包含相同定义，因此需要inline避免链接错误。这时inline就不是可选的优化提示了，而是头文件only设计的必要条件。

如果不是头文件only，是传统的在.h中声明，.cpp中定义的方式，为什么就不强制需要inline了呢？

看一下流程：

main.cpp -> #include "ThreadPool.h" -> 知道了”ThreadPool构造函数存在“

-> main.o ->只知道构造函数签名，不知道具体实现

utils.cpp同理：

ThreadPool.cpp ->  #include "ThreadPool.h" +实现代码

-> ThreadPool.o -> 包含构造函数的定义

链接器：把main.o utils.o和ThreadPool.o合并。

main.o和utils.o说：我们都需要ThreadPool构造函数的定义。

ThreadPool.o说：我这里有定义。

匹配，不报错。

反之：main.o说：我有定义。

utils.o说：我也有定义。

用哪个？报错。

**总结：为什么线程池适合头文件only？**

**1.模板化接口：enqueue方法是模板，必须在头文件中实现**

**2.轻量级：代码量不大，编译时间影响小**

**3.易用：用户只需包含一个文件，无需处理编译链接**

**也因为头文件only，所以需要inline**



:two:lambda表达式中的[this]

```c++
// the constructor just launches some amount of workers
inline ThreadPool::ThreadPool(size_t threads)
    : stop(false)
{
    for(size_t i = 0; i < threads; ++i)
        workers.emplace_back(
            [this]
            {
                for(;;)
                {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(this->queue_mutex);
                        this->condition.wait(lock,
                            [this]{ return this->stop || !this->tasks.empty(); });
                        if(this->stop && this->tasks.empty())
                            return;
                        task = std::move(this->tasks.front());
                        this->tasks.pop();
                    }
                    task();
                }
            });
}
```

每一个类中都有一个this指针，通过this指针，我们可以访问到类的成员函数与变量。在lambda表达式中，[this]的作用就是：**让lambda表达式捕获当前类的this指针，从而可以访问类的所有成员函数与变量**。

追问1：这个构造函数的实现不是在ThreadPool作用域下吗？为什么不能直接访问ThreadPool类的成员函数与变量？

答：**虽然lambda表达式定义在ThreadPool作用域内，但它在另一个独立的线程中执行。**

```c++
ThreadPool::ThreadPool(size_t threads) {
    for(size_t i = 0; i < threads; ++i) {
        workers.emplace_back([this] {  
            // 在新线程中运行
        });
    }
    // 构造函数结束后，lambda可能还在其他线程运行
}
```

也就是说，因为lambda函数体的执行是发生在新线程中的，与执行构造函数的线程是并发的，所以我们要捕获this指针。

追问2：为什么不按值捕获？

答：按值捕获[=]，会尝试拷贝所有变量，但是mutex是不能拷贝的。

互斥锁通常对应操作系统的内核对象，本质是系统资源，不是数据。一个mutex应该只保护一个特定的资源。拷贝mutex会导致所有权混淆，哪个mutex保护哪个数据？

追问3：为什么不按引用捕获？

答：在多线程中，按引用捕获有极大风险，容易访问到已销毁的变量。

```c++
void dangerous_example() {
    std::vector<int> data = {1, 2, 3};
    std::thread t([&] {  // 按引用捕获
        std::cout << data[0];  // 危险，data可能已销毁
    });
    
    // 函数结束，data被销毁
    // 但线程可能还在运行，访问已销毁的data，导致未定义行为
    t.join();
}
```

捕获this指针，虽然也有生命周期要求，但是意图清晰。

另外，如果使用精确捕获特定成员（C++14以后），虽然安全，但捕获列表冗长。

```c++
// 需要访问多个成员时：
[&mutex = this->queue_mutex, 
 &cond = this->condition,
 &tasks = this->tasks,
 &stop = this->stop] {
    // 冗长的捕获列表...
    std::unique_lock lock(mutex);
    cond.wait(lock, [&] { return stop || !tasks.empty(); });
    // ...
}
```

[this]捕获的不是对象的拷贝，而是指针。这意味着没有拷贝开销（只是一个指针）、访问的是真实的成员变量、但也要确保对象生命周期（通过析构函数join线程，确保当this指针无效时，不会有线程试图访问它）。

**总结一下，为什么捕获this指针是最佳选择：**

**1.安全：明确表示lambda依赖整个对象的生命周期**

**2.简洁：不需要像精确捕获那样列出所有成员**

**3.性能：不需要像按值捕获那样拷贝变量**

**4.维护友好：新增成员不需要修改捕获列表**



:three:std::function<void()>

**std::function<void()>是一个通用的、多态的函数包装器。它可以存储、复制和调用任何可调用对象（如普通函数、lambda表达式、函数对象、其它函数的指针等）。它统一了接口，这很多态。**

<void()>是它的模板参数，void表示这个可调用对象没有返回值，()表示这个可调用对象不接受任何参数。

所以，std::function<void()> task; 这行代码的意思是：

**声明一个名为task的变量，它可以用来存储任何一个不接受参数且没有返回值的可调用对象。**

这么看来，这个线程池是不是功能太有限了？只能处理没有返回值且不接受任何参数的可调用对象？

并非如此。enqueue函数会将用户提交的任务包装成std::function<void()>类型并放入任务队列。工作线程从队列中取出的就是这个task对象，然后通过task()来执行它。



:four:为什么要有for(;;)死循环：

因为要保证工人不能在上班期间自己不干了。当下班信号没有发出时，工人要保持在岗。只要stop==false，线程就要处于准备从任务队列中取出任务开始工作的状态。



:five:unique_lock\mutex

```c++
for(;;)
{
    std::function<void()> task;
    {
        std::unique_lock<std::mutex> lock(this->queue_mutex);
        this->condition.wait(lock,
           [this]{ return this->stop || !this->tasks.empty(); });
        if(this->stop && this->tasks.empty())
            return;
        task = std::move(this->tasks.front());
        this->tasks.pop();
     }
     task();
}
```

std::mutex（互斥量）是最基本的同步原语，用于保护共享数据，防止多个线程同时访问（数据竞争）。可以将其视为房间的钥匙，一次只允许一个线程进入（获得锁）。这也是为什么叫做”互斥“，就好比概率论中的互斥事件，有你必然没有我，只要有我这个线程，就必然没有其它线程。常用接口：

lock()：上锁。如果互斥量已被其它线程锁住（有人进入了这个房间并锁了门），则调用线程会被阻塞，直到获得锁（其它人只能在门外等房内的人开锁出来）

unlock()：解锁。释放互斥量的所有权，让其它被阻塞的线程可以获取它（在房内办完事了，解开锁打开房门，让下一个人办事）

try_lock()：尝试上锁。非阻塞版本。如果互斥量当前未被锁，则锁住它并返回true；如果已被锁则立即返回false，线程不会阻塞（如果房内没人就进去并上锁，如果房内有人且上了锁就不等待）

一个危险的例子：

```c++
std::mutex mtx;
int shared_data = 0;

void unsafe_increment() {
    // ... 一些非临界区的操作 ...
    
    mtx.lock(); // 手动上锁
    shared_data++; // 临界区操作
    mtx.unlock(); // 手动解锁。如果临界区代码抛出异常，unlock()可能不会被调用，导致死锁
    
    // ... 其他非临界区的操作 ...
}
```

死锁：永久阻塞

再来看std::unique_lock

std::unique_lock是一个锁管理器，它遵循了RAII（资源获取即初始化）理念。它在构造时自动获取互斥量的锁，在析构时自动释放锁。这保证了即使发生异常，锁也能正确释放，从而避免了死锁。

其核心思想是：将锁的生命周期与一个局部变量（std::unique_lock对象）的生命周期绑定

常用接口：

构造函数unique_lock(mutex_type& m) ：构造时自动调用m.lock()

析构函数~unique_lock() ：析构时自动调用unlock()，保证异常安全

lock()：手动上锁：通常不需要，因为构造时已经做了

unlock()：手动解锁：允许你在锁的生命周期内暂时释放锁，让其它线程运行，然后再重新获取

try_lock()：类似mutex::try_lock()

owns_lock()：返回一个bool，表示当前是否拥有锁

安全的操作：

```c++
std::mutex mtx;
int shared_data = 0;

void safe_increment() {
    // ... 一些非临界区的操作 ...
    
    { // 引入一个新的作用域
        std::unique_lock<std::mutex> lock(mtx); // 构造时自动上锁
        shared_data++; // 临界区操作
    } // lock 对象在此处析构，自动调用 unlock()
    // 即使shared_data++抛出异常，栈展开也会保证lock被析构，锁被释放。
    
    // ... 其他非临界区的操作 ...
}
```

需要注意三点：

1.锁本身并不是保护某段数据，它只是保护了从上锁到解锁之间的代码段，并不会自动关联到某个变量。即：从mtx.lock()到mtx.unlock()之间的代码是受互斥保护的临界区，在这个区域内对共享资源的所有访问都是安全的。

2.一个变量是否为共享资源，这取决于这个变量本身的性质，而不是它是否在临界区内部。一个变量是否为共享资源，是由它的存储周期、作用域以及程序的设计决定的。比如全局变量、静态变量、被多个线程通过指针或引用访问的堆栈数据，它们生来就是共享的。又因为每个线程都有自己独立的栈空间，所以局部变量生来就是安全的。

3.由2，我们需要尽可能让临界区的范围小一点，只包含必须互斥的操作，把不涉及共享资源的操作移到锁外，以提高程序的并发性能。就好比，办事的人在那个上了锁的房间内办了一大堆本可以在家里完成的业务，用的时间自然就久了

举个例子：

:x: 低效做法：大量不涉及共享资源的操作塞入临界区：

```c++
std::mutex mtx;
std::queue<int> shared_queue;

void process_task() {
    std::unique_lock<std::mutex> lock(mtx); // 获取锁
    
    if (shared_queue.empty()) {
        lock.unlock();
        return;
    }
    
    int value = shared_queue.front();
    shared_queue.pop();
    
    // 开始一个非常耗时的计算，但这个计算并不需要锁保护
    // 然而锁还被这个线程持有，其他线程全被阻塞了
    double result = 0;
    for (int i = 0; i < 1000000; ++i) {
        result += std::sqrt(std::sin(value) + std::cos(i));
    }
    // 计算结束 
    
    std::cout << "Result: " << result << std::endl;
    
} // 锁在这里才被释放，太晚了
```

:white_check_mark:  高效做法：临界区只做必要的事：

```c++
std::mutex mtx;
std::queue<int> shared_queue;

void process_task() {
    int value_to_process;
    
    { // 创建一个最小作用域的临界区
        std::unique_lock<std::mutex> lock(mtx);
        
        if (shared_queue.empty()) {
            return; // 锁自动释放
        }
        
        value_to_process = shared_queue.front(); // 1. 从共享资源读取
        shared_queue.pop();                      // 2. 修改共享资源
    } // 临界区结束，锁在此处立即释放
    
    // 在锁之外执行耗时计算
    // 现在其他线程可以自由地访问队列了，不会在std::unique_lock<std::mutex> lock(mtx)阻塞
    double result = 0;
    for (int i = 0; i < 1000000; ++i) {
        result += std::sqrt(std::sin(value_to_process) + std::cos(i));
    }
    // 计算结束
    
    std::cout << "Result: " << result << std::endl;
}
```

追问：为什么在缩小临界区后会有性能提升？难道不是每个线程都从i=0开始算吗？

答：因为这个计算中含有value_to_process，每个线程都有自己的value_to_process.缩小临界区后，线程A解锁后线程B就可以去队列里拿数据，不必等待线程A计算完了再拿数据。但是如果将value_to_process改成i，那就完全没有性能提升了。这就好比100个人做试卷，每个人都是用2小时做完，1个人做试卷，也是用2小时做完，没有任何性能提升。

并发编程想要获得性能提升，要保证：

1.每个线程执行的任务是不同的

2.没有冗余计算

3.线程间协调的开销小于并行带来的收益

再来看std::unique_lock的几种构造模式：

1.默认模式：立即锁定

```c++
std::mutex mtx;
// 构造时立即调用 mtx.lock()
std::unique_lock<std::mutex> lock(mtx); 

// 现在临界区被锁定...
// ...操作共享资源

// 作用域结束，lock析构，自动解锁
```

2.延迟锁定模式：std::defer_lock

有时我们需要先创建锁对象但暂时不上锁，稍后再手动上锁

```c++
std::mutex mtx;
// 使用std::defer_lock标签，构造时不锁定mutex
std::unique_lock<std::mutex> lock(mtx, std::defer_lock); 

// ...这里可以执行一些不需要锁的操作...

lock.lock();   // 现在才手动上锁
// 临界区...
lock.unlock(); // 可以手动提前解锁

// ...又可以执行非临界区代码...

lock.lock();   // 再次上锁
// 另一个临界区...

// 作用域结束，如果锁还持有，会自动解锁
```

3.尝试锁定模式：std::try_to_lock

尝试获取锁，如果获取失败也不阻塞

```c++
std::mutex mtx;
// 尝试获取锁，不会阻塞
std::unique_lock<std::mutex> lock(mtx, std::try_to_lock); 

if (lock.owns_lock()) { // 检查是否成功获取了锁
    // 成功获取锁，可以安全操作共享资源
    std::cout << "成功获取锁" << std::endl;
} else {
    // 获取锁失败，执行其他不需要锁的操作
    std::cout << "获取锁失败，做其它工作" << std::endl;
}
// lock析构时，如果拥有锁会自动释放
```

4.接管已锁定的互斥量 std::adopt_lock

当你已经手动锁定了互斥量，但想要享受unique_lock的自动解锁好处时：

```c++
std::mutex mtx;
mtx.lock(); // 手动先锁定

// 现在用adopt_lock来接管这个已锁定的mutex
std::unique_lock<std::mutex> lock(mtx, std::adopt_lock); 

// 现在unique_lock管理这个已锁的mutex
// 临界区...

// 作用域结束会自动解锁
```

还有一个RAII锁std::lock_guard，它更轻量，更简单，但功能也更少。它只能在构造时上锁，析构时解锁，中间不能手动解锁，适用于简单的临界区。

std::unique_lock虽然更重，但功能也更强大。支持延迟锁定、手动解锁、条件变量等。

如果只是需要一个简单的、作用域内的锁：用std::lock_gurad

如果需要与条件变量配合，或需要灵活控制锁：用std::unique_lock

接下来我们看std::unique_lock是如何与条件变量配合的



:six:std::condition_variable condition

```c++
for(;;)
{
    std::function<void()> task;
    {
        std::unique_lock<std::mutex> lock(this->queue_mutex);
        this->condition.wait(lock,
           [this]{ return this->stop || !this->tasks.empty(); });
        if(this->stop && this->tasks.empty())
            return;
        task = std::move(this->tasks.front());
        this->tasks.pop();
     }
     task();
}
```

std::condition_variable引入了线程间的协作机制。std::mutex是让线程们”别打架，一个一个来“，std::condition_variable是让线程们”没事别瞎忙，等有活了再起来干”

在不知道condition_variable的条件下，该如何设计？

如果不知道有这个东西的存在，可以这么设计：定义一个int类型的全局变量flag，当任务队列为空时令flag=1，下班信号发出时令flag=2，有活干的时候令flag=0，然后在每一个线程中加一个while循环内嵌套if判断，判断flag的值。

但是这个方法缺陷很明显：

1.线程不停地循环、判断，消耗CPU。如果线程数量很多，会造成巨大的、完全不必要的CPU资源浪费和电力消耗

2.flag作为共享资源，是一个被多个线程读写的共享变量，需要加锁，代码变得复杂。而且，对flag的读写操作不是原子操作（比如，flag=1在汇编层面是好几条指令）。如果不加锁，需要面临数据竞争、缓存一致性问题

3.竞态条件。线程A检查flag，发现是1，队列空，于是它决定休息一下（执行yield或sleep，否则浪费CPU资源）。就在它决定休息但还未真正休息的瞬间，主线程提交了一个新的任务，将flag设置为0.线程A此时才真正休息，进入睡眠。造成的结果是：新任务来了，但工作线程在睡眠。直到下一次某个线程被调度醒来检查flag时才会发现它，响应延迟产生了。

响应延迟vsCPU资源消耗的两难困境：如果想减少CPU消耗，就必须让线程睡眠，但这就引出了一个新问题：睡眠多久？太短，CPU消耗依然很高；太长，响应延迟很高。无法同时优化CPU消耗和响应延迟。

std::condition_variable就是最佳解决方案。它实现了“零消耗等待”和“即时唤醒”的理想状态。

std::condition_variable通常用于生产者-消费者模型。生产者线程负责生产数据（任务），将其放入队列。消费者线程负责从队列中取出数据（任务）处理。队列是共享资源，需要用互斥锁保护。

消费者线程：

```c++
std::unique_lock<std::mutex> lock(mtx);
// 等待条件：如果条件不满足(队列为空且未停止)，就释放锁并阻塞在这里（休眠）
condition.wait(lock, [this] {
    return !tasks.empty() || stop; // 有任务或下班信号发出，就不再阻塞在这里
});

if (stop && tasks.empty()) return; // 如果是停止信号，就退出
// 条件满足（有任务），现在锁已经被重新获取，可以安全操作队列
auto task = std::move(tasks.front());
tasks.pop();
lock.unlock(); // 尽早释放锁

task(); // 在锁外执行任务
```

生产者线程：

```c++
{
    std::unique_lock<std::mutex> lock(mtx);
    tasks.push(std::move(new_task));
} // 锁在作用域结束时释放
condition.notify_one(); // 通知一个等待的线程：条件可能满足了，起来看一下
```

需要注意wait()的流程：

1.检查谓词：

如果返回true，不阻塞，继续执行

如果返回false，**原子地**解锁并阻塞在wait这个位置。解锁了，送货的（enqueue线程）才进得来。

2.当线程被notify_all()或notify_one()唤醒时，线程会重新获取锁

3.接着线程再次检查谓词条件，回到1

使用循环或谓词是为了防止虚假唤醒：线程可能在没有收到任何通知的情况下被操作系统唤醒。通过再次检查条件可以确保唤醒是真正有效的。

condition_variable的核心接口：

wait(lock,pred)：pred为false时**原子地**释放锁并阻塞，直到被唤醒且pred为真

notify_one()：唤醒任意一个正在等待此条件变量的线程。注意有两个前提条件：1.任意一个，这个选择是非确定的，由系统决定（当生产者只提交了一个任务时，就只需要一个工人，这种情况使用notify_one()） 2.正在等待此条件变量的线程（这个线程只能是阻塞的，不能是还没执行到wait()的，也不能是已经从wait()中返回的，即被唤醒并已经在干活的）

notify_all()：唤醒所有正在等待此条件变量的线程。同样，只对正在阻塞在这个特定条件变量上的线程有效。

前面提到了“原子地”，什么叫“原子地”？什么是“原子操作”？

**原子操作指的是一个或多个操作组成的序列，对于执行这些操作地线程以外地任何线程来说，这些操作要么完全执行完毕，要么完全没执行，不存在执行到一半的中间状态。在并发环境中，原子操作是不可分割的，执行过程中不会被线程调度机制打断。**

比喻理解：你从账户A向账户B转账100元，分两步：

1.从账户A减去100元

2.往账户B增加100元

非原子操作：这两个步骤不是原子性的，那么在你执行完第一步但还没执行到第二步的瞬间，系统查账会显式你的总金额少了100元。如果你委托了你的5个朋友对你的账户进行同样的操作，那么在这一瞬间，你的账户可能会少了600元。

原子操作：转账被设计成一个原子操作。那么对于你那五个朋友来说，你要么没有转帐，要么已经转账完成了，不存在A少了100元但B没加100元这种中间态。

其实就像数学里的离散函数，在某个点附近函数值发生突变，不存在中间态。

即使是一条简单的C++语句，在底层也可能对应多条CPU指令，以自增为例：

```c++
//假设cnt是一个全局int变量
cnt++;//这条语句不是原子性的
```

它可能被编译成这样的机器指令：

1.LOAD：将cnt的值从内存加载到CPU寄存器

2.ADD：在寄存器中将值加1

3.STORE：将寄存器中的新值存回cnt所在的内存

如果两个线程同时执行cnt++，可能会发生这样的交错执行：

| 时间线 | 线程A                | 线程B                |
| ------ | -------------------- | -------------------- |
| t1     | LOAD（读到0）        |                      |
| t2     | ADD（寄存器变成1）   |                      |
| t3     |                      | LOAD（也读到值0）    |
| t4     | STORE（内存中存为1） |                      |
| t5     |                      | ADD（寄存器变成1）   |
| t6     |                      | STORE（内存中存为1） |

最终cnt的值是1，而不是正确的2.这就是数据竞争，本质是缓存不一致。

实现原子性的两个方法：

1.硬件支持：现代CPU提供了原子指令。这些指令在硬件层面保证了对一个内存单元的读写操作在一个总线周期内完成。C++的std::atomic类型就是利用这些指令实现的

2.软件同步（互斥锁）：用锁将非原子操作序列（临界区）包裹起来，使得这一系列操作在效果上变得原子化

condition_variable的原子性：它将以下三个操作合并为一个不可分割的整体：

1.检查条件

2.释放锁

3.进入等待

如果没有这种原子性，就可能发生竞态条件：

1.线程A检查条件，为false，准备释放锁并阻塞

2.在线程A释放锁并阻塞之前，生产者线程提交了一个任务，调用notify_one()

3.但此时，线程A还没有真的释放锁并阻塞，即还没有调用wait()，因此notify_one()无效。上文说了，notify只对那些已经处于阻塞状态的线程有效。现在A是处于“即将阻塞”的状态

4.线程A释放锁，阻塞，任务无人处理

condition.wait(lock,pred)的原子性意味着：判断条件、解锁、阻塞这一系列动作被打包成了一个连续的、不可分割的原子操作。生产者的放任务+notify操作，要么发生在线程A判断条件之前（那么线程A会看到有活，直接干活），要么发生在线程A阻塞之后（那么notify生效，铃声吵醒工人A），绝不可能发生在中间。

**并发编程最麻烦的问题，本质上来说就是执行时序的不确定性，因为我们永远都不知道操作系统（CPU调度器）会在什么时候执行哪个线程。**

**总结一下condition_variable 的作用：让判断条件、解锁、阻塞这一动作原子化，避免竞态条件。**

比喻理解：1.检查与准备等待：工人（线程）拿到工厂钥匙（获取锁），检查任务清单（检查谓词），发现没任务且工厂没要求下班；2.安全地休息：工人不能锁门睡觉，否则送货的（enqueue线程）进不来。工人把钥匙交给门卫（原子地解锁），睡觉（阻塞）；3.送货员来了，提交货物，叫醒工人（notify_one()）。工人被叫醒，走到门卫处拿钥匙（原子地重新获取锁），再次检查任务清单，确认有货了，开始干活。

未来我们可以看一下如何使用std::atomic代替锁实现一些简单结构。

现在我们可以解释，为什么是std::unique_lock而不是std::lock_guard了。因为这个过程要求wait()的锁对象必须具备手动解锁和重新上锁的能力。std::lock_guard不提供任何lock()或unlock()公共接口。它在构造时上锁，析构时解锁，锁的生命周期完全由其对象的作用域决定，中间无法手动干预。





:seven:为什么是std::move

```c++
for(;;)
{
    std::function<void()> task;
    {
        std::unique_lock<std::mutex> lock(this->queue_mutex);
        this->condition.wait(lock,
           [this]{ return this->stop || !this->tasks.empty(); });
        if(this->stop && this->tasks.empty())
            return;
        task = std::move(this->tasks.front());
        this->tasks.pop();
     }
     task();
}
```

1.性能：避免昂贵的拷贝开销

std::function<void()>是一个封装了可调用对象地包装器，它可能持有被捕获的变量、函数对象等，其内部可能需要动态分配内存。如果没有std::move()，task=tasks.front()会调用std::function地拷贝构造函数或拷贝赋值运算符，这意味着可能会进行一次深拷贝，复制所有内部状态，以及可能会分配新的内存。这是一个潜在的性能开销较大的操作。

使用std::move会调用std::function的移动赋值运算符，这意味着task直接窃取了tasks.front()内部的资源（如指针、缓冲区），tasks.front()处于无效状态（可以安全析构）。这种操作开销极低，通常只是复制几个指针。

2.安全性：

对于某些只可移动不可拷贝的类型（如std::unique_ptr），task=tasks.front()无法编译

**总结std::move(tasks.front())的作用：**

**1.性能优化，避免了昂贵的拷贝操作，高效地转移其内部资源的所有权**

**2.所有权语义：明确表示从队列中取走任务，队列不再拥有这个任务，为接下来的pop()操作做好了准备**

**3.通用性：使得线程池能够处理那些只可移动不可拷贝的任务对象**



















