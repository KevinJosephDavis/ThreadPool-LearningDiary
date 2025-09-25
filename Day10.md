## ThreadPool Learning Day10/11/12

目标：完成性能测试



在平衡负载、非平衡负载、混合负载的条件下测量几个指标：

1.每个任务的平均用时

2.每秒处理的任务数量

3.第95%位置和第99%位置的延迟

写一个结构体`TestResult`，把上述的指标作为结构体的成员



测量这几个指标，需要什么？
1.要测平均用时，就要测总时长，再除以任务数量

2.要测每秒处理的任务数量，那就是任务数量/总时长（换成秒）

3.要测百分位数，就要测量每个任务执行的时间，再排序

所以要测量：1.总时长 2.每个任务的执行时间

至于任务数量，这个可以当作参数去传递

我们是要对比工作窃取优化后的线程池、标准线程池和直接创建线程的效率，所以具体是使用哪个池，需要等到调用的时候才知道，所以我们可以使用模板。这么一来，直接创建线程就需要单独写一个函数

不妨直接写一个类`PerformanceTest`，让`TestResult`成为其成员。把平衡负载、非平衡负载和混合负载都写成模板函数放进类里，再把直接创建线程（平衡负载情况）的测试写成函数放进类里。

这些函数都返回什么？返回测试结果`TestResult`，而由于每个函数都要计算QPS、平均用时、百分位数这些指标，所以我们不妨写一个处理数据的函数`calculate_result`，它接收什么参数？首先，它应该要知道自己处理的是普通线程池还是优化线程池的数据，所以我们需要传一个字符串类型的参数表示名字；其次，它要得到总时长、总任务数量、每个任务的执行时间，所以我们把这三个量作为参数传进去。其中，每个任务的执行时间应该用vector存储

```c++
   static TestResult calculate_result(const std::string& name, 
                                     const std::vector<long long>& latencies,
                                     long long total_time_ms, 
                                     int total_tasks) {
        TestResult result;
        result.name = name;
        result.total_time_ms = total_time_ms;
        result.qps = total_tasks * 1000.0 / total_time_ms;
        
        std::vector<long long> sorted_latencies = latencies;
        std::sort(sorted_latencies.begin(), sorted_latencies.end());
        
        result.avg_latency_us = std::accumulate(sorted_latencies.begin(), sorted_latencies.end(), 0LL) / sorted_latencies.size();
        result.p95_latency_us = sorted_latencies[sorted_latencies.size() * 95 / 100];
        result.p99_latency_us = sorted_latencies[sorted_latencies.size() * 99 / 100];
        
        std::cout << name << " - QPS: " << result.qps 
                  << ", Total: " << total_time_ms << "ms"
                  << ", Avg Latency: " << result.avg_latency_us << "μs"
                  << ", P95: " << result.p95_latency_us << "μs"
                  << ", P99: " << result.p99_latency_us << "μs" << std::endl;
        
        return result;
    }

```

再看两个模板函数`test_balanced_workload`和`test_imbalanced_workload`所需要的参数以及实现。参数很简单，就是一个std::string即可，表示进行测试的线程池类型。接着看实现：首先构造线程池，声明一个名为latency的vector，用来存储每个任务的执行时长。开始记录总时长，进入控制单个任务的for循环，开始记录单个任务的执行时长，往latency可变数组里push一个pool.enqueue函数。enqueue什么？enqueue一个lambda表达式，其参数为单个任务的开始时间，任务复杂度（任务本身for循环的次数），并指定其返回值为long long，开始执行计算任务，记录结束时间，再返回结束时间减去开始时间的值即可。退出控制单个任务的for循环后，计算结束时间，即可得到总时长。最后调用calculate_test，传入线程池名字、latency数组、总时长、任务数量，并返回calculate_test返回的结果

这时候发现异常了：

1.enqueue函数返回的是std::future<return_type>，而latency的元素类型为long long，类型不匹配

2.控制单个任务的for循环结束之后，也许任务还在线程池内执行，此时直接计算结束时间，误差太大了

所以我们应该再创建一个名为futures且元素类型为`std::future<long long>`的vector，应该是往futures可变数组里推入pool.enqueue函数。退出控制单个任务的for循环后，遍历futures可变数组的元素fut，把fut.get()的结果push到latency数组中，因为.get()保证了任务必然完成。最后再计算总的结束时间

```c++
    template<typename PoolType>
    static TestResult test_balanced_workload(const std::string& name) {
        constexpr int TOTAL_TASKS = 10000;
        constexpr int TASK_COMPLEXITY = 1000;
        
        PoolType pool(std::thread::hardware_concurrency());
        std::vector<long long> latencies;
        latencies.reserve(TOTAL_TASKS);
        
        auto start_time = std::chrono::high_resolution_clock::now();
        
        std::vector<std::future<long long>> futures;
        futures.reserve(TOTAL_TASKS);
        
        for (int i = 0; i < TOTAL_TASKS; ++i) {
            auto task_start = std::chrono::high_resolution_clock::now();
            futures.push_back(pool.enqueue([task_start, complexity = TASK_COMPLEXITY]() -> long long {
                // 模拟计算任务
                int result = 0;
                for (int j = 0; j < complexity; ++j) {
                    result += j * j;
                }

                do_not_optimize(result); // 防止优化掉计算
                auto end_time = std::chrono::high_resolution_clock::now();
                return std::chrono::duration_cast<std::chrono::microseconds>(end_time - task_start).count();
            }));
        }
        
        // 收集结果
        for (auto& fut : futures) {
            latencies.push_back(fut.get());
        }
       
        auto end_time = std::chrono::high_resolution_clock::now();
        auto total_duration = std::chrono::duration_cast<std::chrono::milliseconds>(end_time - start_time);
        
        return calculate_result(name, latencies, total_duration.count(), TOTAL_TASKS);
    }

```

三个点：

1.complexity都没声明过，为什么能直接complexity=TASK_COMPLEXITY？

C++14之后支持初始化捕获，编译器会自动推导类型，相当于auto complexity=TASK_COMPLEXITY

不过在某些情况下，我们可能需要显式指定类型，比如说需要const引用，或者复杂的表达式类型不明确时

2.为什么要调用.count()？

因为total_duration的类型是`std::chrono::milliseconds`，而calculate_result第三个参数需要的是long long，所以要用.count()方法提取出纯粹的数值部分

事实上，`std::chrono::milliseconds`不仅包含数值，还包含时间单位。就好比我们定义了一个类`Ruler`，里面有两个成员`std::string material`和`int length`，我们不能说一个Ruler对象ruler就是int类型，因为ruler里包含了材质、长度两个信息

3.防止编译器优化：因为我们最终返回的东西和result一点关系都没有，所以编译器可能会优化掉计算任务，这会对计算时间造成误差。一开始我使用volatile，但是编译器发出了警告，所以就换成使用do_not_optimize了，方法如下：

```c++
    template<typename T>
    static void do_not_optimize(T& value) {
        asm volatile("" : "+r,m"(value) : : "memory");
    }
```

```c++
asm volatile("汇编指令": 输出操作数 : 输入操作数 : 被破坏的寄存器 : 内存约束);
```

我们的汇编指令为空：因为我们并不生成任何实际的CPU指令，目的只是欺骗编译器

输出操作数：格式为`"约束"(变量)`

`+` ：表示读写操作数。变量在汇编指令执行前作为输入，执行后作为输出。当编译器看到这一行时，它必须在执行前确保value的值已经计算完成，并准备好（在寄存器或内存中）。执行后假设value可能已被修改，必须重新读取。如果没有加号，编译器知道value只读，不会被修改，但可能仍然进行某些优化，有加号，编译器必须假设value可能被修改，阻止更多的优化机会

`r` ：约束，要求编译器将变量放在通用寄存器中

`m` ：约束，要求编译器将变量放在内存中

`r,m` ：多个约束，让编译器选择最合适的

内存约束：`"memory"`告诉编译器这段汇编可能会读写任何内存地址，强制编译器重新加载所有缓存的变量值

等价的理解方式：

1.强制编译器计算value的当前值

2.将value加载到寄存器或确保在内存中可用

3.假装可能会修改value

总之，它的原理是：

1.通过操作数约束强制编译器确保变量可用

2.通过输入输出标记让编译器认为变量可能被修改

3.通过内存屏障防止内存访问优化

4.并不生成实际使用变量的代码



平衡负载的函数写完了，非平衡负载和混合负载其实也差不多

```c++
    template<typename PoolType>
    static TestResult test_imbalanced_workload(const std::string& name) {
        constexpr int TOTAL_TASKS = 8000;
        
        PoolType pool(std::thread::hardware_concurrency());
        std::vector<long long> latencies;
        latencies.reserve(TOTAL_TASKS);
        
        auto start_time = std::chrono::high_resolution_clock::now();
        std::vector<std::future<long long>> futures;
        
        for (int i = 0; i < TOTAL_TASKS; ++i) {
            auto task_start = std::chrono::high_resolution_clock::now();
            
            // 制造不均衡：70%的任务是轻量级，30%是重量级
            if (i % 10 < 7) {
                futures.push_back(pool.enqueue([task_start]() -> long long {
                    std::this_thread::sleep_for(std::chrono::microseconds(50));
                    auto end_time = std::chrono::high_resolution_clock::now();
                    return std::chrono::duration_cast<std::chrono::microseconds>(end_time - task_start).count();
                }));
            } else {
                futures.push_back(pool.enqueue([task_start]() -> long long {
                    std::this_thread::sleep_for(std::chrono::microseconds(500));
                    auto end_time = std::chrono::high_resolution_clock::now();
                    return std::chrono::duration_cast<std::chrono::microseconds>(end_time - task_start).count();
                }));
            }
        }
        
        for (auto& fut : futures) {
            latencies.push_back(fut.get());
        }
        
        auto end_time = std::chrono::high_resolution_clock::now();
        auto total_duration = std::chrono::duration_cast<std::chrono::milliseconds>(end_time - start_time);
        
        return calculate_result(name, latencies, total_duration.count(), TOTAL_TASKS);
    }
```

```c++
    template<typename PoolType>
    static TestResult test_mixed_tasks(const std::string& name) {
        constexpr int TOTAL_TASKS = 6000;
        
        PoolType pool(std::thread::hardware_concurrency());
        std::vector<long long> latencies;
        latencies.reserve(TOTAL_TASKS);
        
        auto start_time = std::chrono::high_resolution_clock::now();
        std::vector<std::future<long long>> futures;
        
        for (int i = 0; i < TOTAL_TASKS; ++i) {
            auto task_start = std::chrono::high_resolution_clock::now();
            
            // 混合不同大小的任务
            int complexity = 100 + (i % 10) * 200; // 100到1900的复杂度
            futures.push_back(pool.enqueue([task_start, complexity]() -> long long {
                int result = 0;
                for (int j = 0; j < complexity; ++j) {
                    result += j * j;
                }
                do_not_optimize(result); // 防止优化掉计算
                
                auto end_time = std::chrono::high_resolution_clock::now();
                return std::chrono::duration_cast<std::chrono::microseconds>(end_time - task_start).count();
            }));
        }
        
        for (auto& fut : futures) {
            latencies.push_back(fut.get());
        }
        
        auto end_time = std::chrono::high_resolution_clock::now();
        auto total_duration = std::chrono::duration_cast<std::chrono::milliseconds>(end_time - start_time);
        
        return calculate_result(name, latencies, total_duration.count(), TOTAL_TASKS);
    }

```

对于直接创建线程的案例，我们也单独写一个函数

```c++
    static TestResult test_direct_threads_balanced(const std::string& name) {
        constexpr int TOTAL_TASKS = 1000; 
        constexpr int TASK_COMPLEXITY = 1000;
        
        std::vector<long long> latencies;
        latencies.reserve(TOTAL_TASKS);
        
        auto start_time = std::chrono::high_resolution_clock::now();
        std::vector<std::future<long long>> futures;
        
        auto task_start = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < TOTAL_TASKS; ++i) {
            futures.push_back(std::async(std::launch::async, [task_start,complexity = TASK_COMPLEXITY]() -> long long {
                int result = 0;
                for (int j = 0; j < complexity; ++j) {
                    result += j * j;
                }
                do_not_optimize(result); // 防止优化掉计算

                auto end_time = std::chrono::high_resolution_clock::now();
                return std::chrono::duration_cast<std::chrono::microseconds>(end_time - task_start).count();
            }));
        }
        
        for (auto& fut : futures) {
            latencies.push_back(fut.get());
        }
        
        auto end_time = std::chrono::high_resolution_clock::now();
        auto total_duration = std::chrono::duration_cast<std::chrono::milliseconds>(end_time - start_time);
        
        return calculate_result(name, latencies, total_duration.count(), TOTAL_TASKS);
    }
```

一个点：`std::async`

`std::async`是一个函数模板，它接受一个可调用对象，在后台异步执行这个可调用对象，返回一个`std::future`用于获取执行结果

基本语法：

```c++
#include<future>

template<class Function,class ... Args>
std::future<std::invoke_result_t<Function,Args...>>async(std::launch policy,
                                                        Funtion&& f,Args&&... args);
```

`std::launch policy`表示启动策略，有两个选项：

1.`std::launch::async`

保证任务会在独立的线程中执行，并且是立即执行

2.`std::launch::deferred`

任务不会立即开始，只在调用future.get()时执行



这几个函数最终返回的值是经过calculate_result函数处理后的result（结构体），最终我们还要把结果打印出来。考虑到我们要进行平衡负载、非平衡负载和混合负载三个测试，所以我们把结果都存进一个vector里面，最后打印这个vector即可

```c++
  static void print_summary(const std::vector<TestResult>& results) {
        std::cout << "\n=== Performance Summary ===" << std::endl;
        for (const auto& result : results) {
            std::cout << result.name << ": " << result.qps << " QPS" << std::endl;
            std::cout<<  result.name << ": "<<  result.avg_latency_us << "μs Avg Latency" << std::endl;
            std::cout<<  result.name << ": "<<  result.p95_latency_us << "μs P95 Latency" << std::endl;
            std::cout<<  result.name << ": "<<  result.p99_latency_us << "μs P99 Latency" << std::endl;
            std::cout << "---------------------------" << std::endl;
        } 
    }
```

最后写一个总的测试函数

```c++
    static void run_comprehensive_test() {
        const int num_threads = std::thread::hardware_concurrency();
        std::cout << "Testing with " << num_threads << " threads\n" << std::endl;

        std::vector<TestResult> results;
        
        std::cout << "=== Test 1: Balanced Workload ===" << std::endl;
        results.push_back(test_balanced_workload<StealingThreadPool>("Work Stealing Pool"));
        results.push_back(test_balanced_workload<StandardThreadPool>("Standard Thread Pool"));
        results.push_back(test_direct_threads_balanced("Direct Threads"));
        
        std::cout << "\n=== Test 2: Imbalanced Workload ===" << std::endl;
        results.push_back(test_imbalanced_workload<StealingThreadPool>("Work Stealing Pool"));
        results.push_back(test_imbalanced_workload<StandardThreadPool>("Standard Thread Pool"));
        
        std::cout << "\n=== Test 3: Mixed Task Sizes ===" << std::endl;
        results.push_back(test_mixed_tasks<StealingThreadPool>("Work Stealing Pool"));
        results.push_back(test_mixed_tasks<StandardThreadPool>("Standard Thread Pool"));
        
        print_summary(results);
    }
```

在main里面调用：

```c++
int main() {
    PerformanceTest::run_comprehensive_test();
    return 0;
}
```

编译的时候，报了一长串错误：

![4d3325ac1894f383a36176291aeb1fd6](C:\Users\31604\xwechat_files\wxid_1wfmihdo10ne22_3149\temp\RWTemp\2025-09\9e20f478899dc29eb19741386f9343c8\4d3325ac1894f383a36176291aeb1fd6.png)

看了一下，大概是`std::mutex`和`std::condition_variable`不能移动或复制，而我这里对它们的vector使用了.resize()

`std::vector::resize()`方法在需要增加元素时，要求元素类型必须是**可默认构造**和**可移动构造**的

实际上当vector需要扩容时，它并不是真的扩容，而是分配一个更大的内存块，将旧内存中的元素移动到新内存中，对新增加的元素进行默认构造，再销毁旧vector中的元素并释放内存

**即：**

**1.分配新内存**

**2.移动现有元素**

**3.构造新元素**

**4.销毁旧内存**

锁的状态与特定的线程和内存位置绑定，移动一个锁，它的所有权如何转移？这会导致未定义行为

如果我们复制锁，那么就会有逻辑矛盾了：mtx1和mtx2究竟是一个锁还是两个锁？如果某个线程拿到了mtx1，那究竟是哪个线程拥有mtx2？那如果一个线程拿到了mtx1，另一个线程拿到了mtx2，那就变成两个对象操作同一个系统锁了。这违反了互斥量的唯一性

条件变量则是和特定的mutex和等待队列绑定，如果允许复制条件变量，那么等待cv1的线程应该自动转移到等待cv2的等待队列吗？原来等待的线程应该等待cv1还是cv2？如果允许移动条件变量，那么等待cv1的线程现在属于谁？这会破坏等待线程与条件变量之间的归属关系

所以我们使用`unique_ptr`对`std::mutex`和`std::condition_variable`进行管理，`unique_ptr`是可以移动的

举个例子：

```c++
std::vector<std::unique_ptr<std::mutex>> mutexes;

// 初始有2个元素
mutexes.push_back(std::make_unique<std::mutex>());
mutexes.push_back(std::make_unique<std::mutex>());

// 扩容到5个元素
mutexes.resize(5); 
```

resize()内部：

1.分配新内存（容量5）

2.移动构造现有元素

```c++
// 对每个现有元素调用移动构造函数
new_memory[0] = std::move(old_memory[0]); 
new_memory[1] = std::move(old_memory[1]); 
```

3.默认构造新元素

```c++
new_memory[2] = std::unique_ptr<std::mutex>();
new_memory[3] = std::unique_ptr<std::mutex>();  
new_memory[4] = std::unique_ptr<std::mutex>();
```

4.销毁旧vector



修改Day09代码，结果如下：

```c++
#include<iostream>
#include <stdexcept>
#include<vector>
#include<deque>
#include<functional>
#include<thread>
#include<memory>
#include<future>
#include<mutex>
#include<condition_variable>
#include<atomic>
#include<random>

class StealingThreadPool {
public:
    StealingThreadPool(size_t threads);

    template<class F,class ...Args>
    auto enqueue(F&&f,Args&& ... args);
    
    ~StealingThreadPool();

private:

    void work(size_t my_index);
    std::function<void()> steal(size_t my_index);

    std::vector<std::deque<std::function<void()>>>queues;
    std::vector<std::unique_ptr<std::mutex>>queues_mutexes;

    std::vector<std::thread>workers;

    std::atomic<bool>stop;
    std::vector<std::unique_ptr<std::condition_variable>>condition_queue;

};

inline StealingThreadPool::StealingThreadPool(size_t threads) {
    stop.store(false,std::memory_order_relaxed);
    queues.resize(threads);
    queues_mutexes.reserve(threads);
    condition_queue.reserve(threads);
    for(size_t i=0;i<threads;++i) {
        queues_mutexes.emplace_back(std::make_unique<std::mutex>());
        condition_queue.emplace_back(std::make_unique<std::condition_variable>());
    }
    for(size_t i=0;i<threads;++i) {
        workers.emplace_back(
            [this,i](){
                this->work(i);
            }
        );
    }
}

inline void StealingThreadPool::work(size_t my_index) {
    while(!this->stop.load(std::memory_order_relaxed)) {
        std::function<void()>task;
        bool should_wait=true;

        {
            std::unique_lock<std::mutex>lock(*this->queues_mutexes[my_index]);
            if(!this->queues[my_index].empty()) {
                task=std::move(this->queues[my_index].back());
                this->queues[my_index].pop_back();
                lock.unlock();
                task();
                should_wait=false;//has task to be finished,no need to wait
            }
        } 

        if(!should_wait) {
            continue;//go back to check whether a new task was pushed immediately
        }

        //local deque is empty,try to steal
        task=this->steal(my_index);
        if(task) {
            //successfully
            task();
            continue;
        }

        //if failed to steal,go to wait
        {
            std::unique_lock<std::mutex>lock(*this->queues_mutexes[my_index]);
            if(this->stop.load(std::memory_order_relaxed)) {
                break; //in case of the signal turns to true suddenly
            }
            if(!this->queues[my_index].empty()) {
                // a task was pushed suddenly
                continue;
            }
            this->condition_queue[my_index]->wait(lock,
            [this,my_index](){
                return this->stop.load(std::memory_order_relaxed) || !this->queues[my_index].empty();});
        }
    }
}

inline std::function<void()> StealingThreadPool::steal(size_t my_index) {
    std::function<void()> task;
    thread_local std::random_device rd;
    thread_local std::mt19937 gen(rd());
    thread_local std::uniform_int_distribution<size_t>dis(0,this->queues.size()-1);
    for(size_t attempt=0;attempt<2;++attempt) { // two rounds
        for(size_t cnt=0;cnt<this->queues.size();++cnt) { //should be random
            size_t i=dis(gen);
            if(i==my_index || this->queues[i].empty()) {
                continue;
            }

            //if the deque is not empty,try to acquire the lock 
            {
                std::unique_lock<std::mutex>lock(*this->queues_mutexes[i],
                    std::try_to_lock);//if fails to acquire,it won't wait here
                if(lock.owns_lock()) {
                    task=std::move(this->queues[i].front());
                    this->queues[i].pop_front();
                    return task;//steal successfully
                } else {
                    continue;//if fails to steal,try to steal another deque
                }
            }
        }
    }
    return {};
}

template<class F,class ...Args>
auto StealingThreadPool::enqueue(F&&f,Args&& ... args) {
    using return_type=std::invoke_result_t<F,Args...>;
    auto task=std::make_shared<std::packaged_task<return_type()>>( 
        std::bind(std::forward<F>(f),std::forward<Args>(args)...));
    std::future<return_type>res=task->get_future();
    static std::atomic<size_t>next_index{0};
    size_t candidate=next_index.fetch_add(1,std::memory_order_relaxed)%this->queues.size();

    {
        std::unique_lock<std::mutex>lock(*this->queues_mutexes[candidate]);
        //no matter how,enqueue must get the lock so that it can push task
        //so it has to wait here
        if(stop.load(std::memory_order_relaxed)) {
            throw std::runtime_error("enqueue on stopped ThreadPool");
        }
        this->queues[candidate].emplace_back([task](){(*task)();});
        this->condition_queue[candidate]->notify_one();
    }
    return res;
}

inline StealingThreadPool::~StealingThreadPool() {
    this->stop.store(true,std::memory_order_relaxed);
    for(size_t i=0;i<this->condition_queue.size();++i) {
        condition_queue[i]->notify_one();
    }
    for(std::thread &worker:workers) {
        worker.join();//make sure all work finished
    }
}
```

编译、运行，结果如下图：

![image-20250925152209132](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250925152209132.png)

![image-20250925152856242](C:\Users\31604\AppData\Roaming\Typora\typora-user-images\image-20250925152856242.png)

执行了多次，结果都差不多：

平衡负载：标准线程池和窃取线程池难分伯仲，但都优于直接创建线程（直接创建线程比前面两位慢了将近10倍）

非平衡负载：标准线程池和窃取线程池难分伯仲，差得不多

混合负载：窃取线程池 一般来说QPS高于标准线程池，但相对慢了一点点

工作窃取算法在复杂、动态的负载环境下确实能有效提升吞吐量

另外我们还可以注意到，每个测试中工作窃取线程池的极差普遍比较大，看起来我们还可以进一步优化窃取策略

