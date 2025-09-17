## ThreadPool Learning Day04

基于前三天的深入分析，第四天的目标是：自己复现整个线程池项目。

首先，我们在Ubuntu虚拟机中创建一个MyThreadPool文件夹：

![image-20250917090401046](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250917090401046.png)

输入code .

就可以使用vscode打开这个文件夹了

包含了必要的头文件后，我们先声明ThreadPool类

线程池就如同一个流水线工厂，工厂里要有执行任务的工人（线程vector）、任务传送带（任务队列）、提交任务的送货员（enqueue函数）、上下班信号（stop）、中途休息信号（条件变量）。分析完后，我们明确了ThreadPool类的内容。



接着是构造函数与析构函数。在创建工厂后，我们需要立刻招聘工人，所以我们在构造函数中创建好线程，具体多少个由用户自己定义，所以要传一个参数，用来表示创建的线程数量。析构函数发送下班信号，通知左右线程，并等待所有线程做完手头的工作，因此不需要任何参数。

接下来到enqueue函数。用户通过它提交任务（函数），用户具体提交什么函数、它的参数是什么有多少，这个需要等用户具体提交任务的时候才知道。因此，我们不能把这个enqueue函数的参数写死了，使用模板即可解决这个问题。

enqueue函数返回什么？在enqueue中我们提交了任务，提醒任意一个工人来执行任务，然后enqueue函数就结束了。这个时候工人可能还没有执行完任务，所以，这个任务的生命周期必须要超过enqueue函数的生命周期。因此，我们在堆上开辟空间给这个任务，然后用指针管理这片内存。具体是什么指针，后续再说。为了统一，任务队列中的元素都是返回值为void()的无参可调用对象。如何拿到任务执行完的结果？所以，我们希望使用std::future，因此我们可以配合std::packaged_task使用，因为它提供了get_future()接口。这样一来，我们将任务打包，在堆上开辟内存给这个std::packaged_task对象，用指针ptr去管理它，后续这个任务执行完毕后，我们可以通过ptr->get_future()获取和这个std::packaged有共享资源的std::future，这样就可以获取到结果了。

因此，enqueue函数应该返回一个std::future，这个std::future是什么类型的？这个类型就是用户提交的函数返回的类型，具体是什么需要等到用户提交了才知道，但没关系，我们使用了模板。用户提交的函数返回的类型就是std::invoke_result_t<F,Args...>，不需要加typename，因为这本身就是一个类型。为了方便，我们给它起一个别名return_type。所以enqueue函数返回的就是std::future<return_type>对象。但由于这个return_type是编译期才确定的，所以enqueue函数的返回值，我们用auto处理即可。

:question:追问：为什么函数f也要完美转发？

答：函数也有左右值之分。

```c++
// 1. 函数指针（左值）
void func() {}
enqueue(func);            // 左值

// 2. Lambda表达式（左值或右值）
auto lambda = []{};
enqueue(lambda);          // 左值
enqueue([]{ return 42; }); // 右值

// 3. 函数对象（左值或右值）
struct Task {
    int operator()() const { return 42; }
};
Task task;
enqueue(task);            // 左值
enqueue(Task{});          // 右值

// 4. std::function（左值或右值）
std::function<int()> func_obj = []{ return 42; };
enqueue(func_obj);        // 左值
enqueue(std::move(func_obj)); // 右值
```

注：对于普通的函数指针，左右值区别微乎其微，因为指针的拷贝很廉价。但为了一致性与通用性，依然使用完美转发



接下来实现构造函数

一上来先让stop=false，保证工厂一造完就开工。接着是创建工人团队，也就是说创建线程。我们知道，创建线程时要告诉这个线程它要做的工作，也就是说我们要传入一个可调用对象。那在这里每个线程做的工作是什么？

线程工作手册（可调用对象）：每个线程都要保持在岗状态（死循环）。为了减轻负担，如果没有收到下班通知且任务队列为空，那么线程可以阻塞，进入休息状态，等到有任务了（被唤醒）再处于就绪状态。被唤醒后看看是不是收到下班信号且没有任务了，如果是，就下班（return）。如果不是，那么就从任务队列中取任务，执行。

为了方便，我们使用emplace_back原地构造这个线程工作手册对象。

这里发现了一个问题，如果构造函数不加inline关键字，会报错：

Function 'ThreadPool' defined in a header file;function definitions in header files can lead to ODR violations

这就又回到了Day02关于inline的分析了。ThreadPool类是头文件only的，如果这个头文件被多个编译单元（.cpp文件）包含，就会违反ODR规则，导致链接错误。

ODR：One Definition Rule

至于为什么会违反，请参见Day02的笔记。

***不过这里我要纠正一下Day02笔记中的一个说法。此处构造函数前加inline不是可有可无的，是必须的！***

:x: **Day02笔记中我提到，C++17之后类内函数隐式内联，所以这个线程池项目的作者可能是为了兼容旧版本。我搞错了，这个构造函数并不是类内定义的函数！它只是在类内声明，定义是在类外实现的！**

所以解决办法有三个：

1.使用inline

2.直接在类定义内实现，隐式内联

3.声明在.h中，定义在.cpp中

现代C++中，对于模板类和小函数，一般都在头文件中定义，小函数在头文件内联，模板类是强制要在头文件中定义，因为编译器在看到模板使用的同一个地方看到模板定义。大函数在.h声明，在.cpp定义

模板函数在头文件定义时不需要显式声明inline，因为每个实例化都会产生一个独立的函数实例，不违反ODR，编译器会自动处理链接问题



接着实现enqueue函数

根据前面的分析，我们知道enqueue函数的功能是用一个return_type接收人物的类型然后擦除（std::bind），将其打包为一个std::packaged_task对象，获取future，用指针管理这个对象，然后将其放入任务队列中。

```c++
 auto task = std::make_shared< std::packaged_task<return_type()> >(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...)
    );

```

:question: 追问：这里std::packaged_task是如何包装std::bind的？

答：

```c++
auto task=std::make_shared<T>(args...);
//等价于：
auto task=new T(args...);
```

**make_shared的参数并不是指定对象内容，而是传递给其类型的构造函数的**



再讲一下昨天没讲清楚的一个点：为什么emplace不需要显式指定类型？

昨天我们说过，emplace是模板函数：

```c++
template<typename... Args>
void emplace(Args&&... args) {  // 转发引用
    // 在容器内存中直接构造
    new (memory_location) T(std::forward<Args>(args)...);
}
```

它接受任意类型和数量的参数，然后在容器内直接构造对象。

**在enqueue函数中，emplace看到参数是lambda，于是使用lambda构造std::function<void()>（因为队列存储的元素是std::function<void()>类型）**



还有一点，如何理解emplace参数中lambda表达式中的按值捕获task？

就是复制了一个shared_ptr，增加了引用计数，这样原始task销毁后，工作线程还能够拿到副本执行。

如果使用普通的按值捕获，可能会意外捕获不需要的变量，而且代码意图不清晰。

**按值捕获shared_ptr，确保了线程安全的引用计数管理，对象在需要时一定存货，不再使用时自动清理。**



最后一点，为什么for(std::thread &worker:workers)中要加引用符号？

首先，std::thread是不可拷贝构造的。

其次，就算可以拷贝构造，这里在遍历到每一个线程时，如果不加引用，都会创建一个副本。那么后续的.join()也只是副本调用.join()，原线程根本都没调用.join()

最终成品：

```c++
#include <iostream>
#include <thread>
#include <vector>
#include <queue>
#include <functional>
#include <condition_variable>
#include <future>
#include <stdexcept>
#include <mutex>

class ThreadPool {
public:
    ThreadPool(size_t);
    template<class F, class... Args>
    auto enqueue(F&& f, Args&&... args);
    ~ThreadPool();
private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;

    std::condition_variable condition;
    std::mutex queue_mutex;
    bool stop;
};

inline ThreadPool::ThreadPool(size_t threads) : stop(false) {
    for(size_t i = 0; i < threads; ++i) {
        workers.emplace_back(
            [this]() {
                for(;;) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(this->queue_mutex);
                        this->condition.wait(lock,
                            [this]() { return this->stop || !this->tasks.empty(); });
                        if (this->stop && this->tasks.empty()) {
                            return;
                        }
                        task = std::move(this->tasks.front());
                        this->tasks.pop();
                    }
                    task();
                }
            }
        );
    }
}

template<class F, class... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args) {
    using return_type = std::invoke_result_t<F, Args...>;
    
    auto task = std::make_shared<std::packaged_task<return_type()>>(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...)
    );
    
    std::future<return_type> res = task->get_future();
    {
        std::unique_lock<std::mutex> lock(queue_mutex);
        if(stop) {
            throw std::runtime_error("enqueue on stopped ThreadPool");
        }
        tasks.emplace([task]() { (*task)(); });
    }
    condition.notify_one();
    return res;
}

inline ThreadPool::~ThreadPool() {
    {
        std::unique_lock<std::mutex> lock(queue_mutex);
        stop = true;
    }
    condition.notify_all();
    for(std::thread &worker : workers) {
        worker.join();
    }
}
```













