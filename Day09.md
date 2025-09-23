## ThreadPool Learning Day09

目标：基于昨天的理解，实现工作窃取算法

先大致分析一下思路，一边分析一边优化

调用线程池的构造函数时，就为每个线程分配一个专属的双端队列。每个线程执行的函数都和之前一样包含一个死循环。在死循环中，线程判断自己的deque是否为空：如果不为空，就从队尾取出任务并执行；如果为空，就随机选择一个工作线程，尝试窃取它的队头任务。需要注意窃取的时候，或者取出任务的时候要上锁，避免一种情况：队列只有一个任务，它对应的线程想要pop，盗窃线程想要偷。因此每个线程的专属队列都要有一把锁（后续再实现无锁）

这就出现了一个问题：一个线程如何访问另一个线程的专属双端队列？

如果真的把这个deque设置为某个线程的私有财产，那么似乎就无法实现窃取了。因此我们维护一个全局的vector，其元素是deque，即它包含了每个线程对应的任务队列。这又引申出来一个事情：我们必须要给线程分配ID，注意：不能使用get_id()：一方面是它的值对我们来说是“黑盒”，不利于调试；另一方面是它通常不是一个简单的整数，如果要使用的话就必须要维护一个哈希表，这会引入不必要的开销和复杂性。分配了ID之后，就可以通过ID访问到每个线程的deque了。这就保证了每个线程在运行的时候都知道两件事：1.我的队列是哪个？2.所有其它队列的地址在哪？

此外，窃取的选择是随机的，还是尽可能选择队列任务较多的线程去窃取？我们先从简单做起：假设线程数量为N，那就尝试窃取2N次。如果窃取了2N次仍然窃取失败，才认为全局真的没有任务了，进入休眠

enqueue函数也同理，推送任务时先随机分配，也许会造成负载不均衡，但我们先从简单的实现做起

初稿：

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

class ThreadPool {
public:
    ThreadPool(size_t threads);

    template<class F,class ...Args>
    auto enqueue(F&&f,Args&& ... args);
    
    ~ThreadPool();

private:

    void work(size_t my_index);
    std::function<void()> steal(size_t my_index);

    std::vector<std::deque<std::function<void()>>>queues;
    std::vector<std::mutex>queues_mutexes;

    std::vector<std::thread>workers;

    std::atomic<bool>stop;
    std::condition_variable condition;

};

inline ThreadPool::ThreadPool(size_t threads): stop(false) {
    queues.resize(threads);
    queues_mutexes.resize(threads);
    for(size_t i=0;i<threads;++i) {
        workers.emplace_back(
            [this,i](){
                this->work(i);
            }
        );
    }
}

inline void ThreadPool::work(size_t my_index) {
    for(;;) {
        std::function<void()> task;
        {
            std::unique_lock<std::mutex>lock(queues_mutexes[my_index]);
            this->condition.wait(lock,[this,my_index](){
                return this->stop || !this->queues[my_index].empty();
            });
            if (this->stop && this->queues[my_index].empty()) {
                return; //all work finished
            }
            if(!queues[my_index].empty()) {
                task=std::move(queues[my_index].back());
                queues[my_index].pop_back();
                lock.unlock();
                task();
            } else {
                lock.unlock();
                task=this->steal(my_index);
                if(task) {
                    task();
                } else {
                    std::this_thread::yield();//fails to steal,rest for a while
                }
            }
        }
    }
}

inline std::function<void()> ThreadPool::steal(size_t my_index) {
    std::function<void()> task;
    for(size_t attempt=0;attempt<2;++attempt) { // two rounds
        for(size_t i=0;i<this->queues.size();++i) { //should be random
            if(i==my_index || this->queues[i].empty()) {
                continue;
            }

            //if the deque is not empty,try to acquire the lock 
            {
                std::unique_lock<std::mutex>lock(this->queues_mutexes[i],
                    std::try_to_lock);//if fails to acquire,it won't wait here
                if(lock.owns_lock()) {
                    task=std::move(this->queues[i].front());
                    this->queues[i].pop_front();
                    lock.unlock();
                    return task;//steal successfully
                } else {
                    continue;//if fails to steal,try to steal another deque
                }
            }
        }
    }
    return task;
}

template<class F,class ...Args>
auto ThreadPool::enqueue(F&&f,Args&& ... args) {
    using return_type=std::invoke_result_t<F,Args...>;
    auto task=std::make_shared<std::packaged_task<return_type()>>( 
        std::bind(std::forward<F>(f),std::forward<Args>(args)...));
    std::future<return_type>res=task->get_future();
    size_t candidate=0;
    size_t min_size=INT_MAX;
    for(size_t i=0;i<this->queues.size();++i) {
        size_t curr_size=this->queues[i].size();
        if(curr_size<min_size) {
            min_size=curr_size;
            candidate=i;
        }
    }

    {
        std::unique_lock<std::mutex>lock(this->queues_mutexes[candidate]);
        //no matter how,enqueue must get the lock so that it can push task
        //so it must wait here
        if(stop.load()) {
            throw std::runtime_error("enqueue on stopped ThreadPool");
        }
        this->queues[candidate].emplace_back([this](){(*task)();});
    }
    return res;
}

inline ThreadPool::~ThreadPool() {
    stop =true;//??
    this->condition.notify_all();//needs to be received in constructor 
    for(std::thread &worker:workers) {
        worker.join();//make sure all work finished
    }
}
```



我自己在写的时候，就已经感觉有点不对了，主要是条件变量的问题

1.enqueue函数是否调用notify？

我一开始写enqueue的时候的想法是，让线程持续判断自己的工作队列是否有任务（事实上，这已经违背了线程池的初衷，因为一直在忙等待，而不是有任务来了再被唤醒），如果有就执行，没有就窃取。那么只要enqueue拿到锁并且推送任务后，线程就会收到任务。但是最后写析构函数的时候，由于必须要让工人下班，所以必须要调用notify_all()，那这么一来工人也必须要接收，所以我才在构造函数那里补上了`condition.wait()`

但是，补上了之后有一个严重的问题：线程抢到锁后（因为每个队列一把锁，所以竞争很小）就会进入wait，那么如果这时候enqueue没有推送任务给它（虽然enqueue是找任务队列最短的那一个线程，但是有可能有多个线程的任务队列长度都为最小值，所以依然有随机性），它就会一直阻塞在wait()处。阻塞不是问题，问题在于，如果后续enqueue向它推送任务，那么它并不会收到消息！那这个任务就白推送了

这又引申出来一个问题，如果是有任务才被notify的话，那么就不可能执行到窃取这个环节了。因为线程被唤醒必然是因为有任务，有任务就不会进入else分支，这就完全与工作窃取不沾边了

2.一个小问题：窃取的时候最好随机选择，不然窃取的时候总是从第一个线程开始窃取



解决方案：

针对问题一：enqueue暂时不调用notify，降低每个线程wait的优先级，确保每个线程都先处理本地队列，只有为空时才窃取，最后如果窃取不到才wait

此外为了缓解对condition的竞争，我们可以创建一个condition_variable的vector，让每个线程都有自己的条件变量，这样notify就实现了精确唤醒了

针对问题二：写个随机数即可



第二稿：

```c++
class ThreadPool {
public:
    ThreadPool(size_t threads);

    template<class F,class ...Args>
    auto enqueue(F&&f,Args&& ... args);
    
    ~ThreadPool();

private:

    void work(size_t my_index);
    std::function<void()> steal(size_t my_index);

    std::vector<std::deque<std::function<void()>>>queues;
    std::vector<std::mutex>queues_mutexes;

    std::vector<std::thread>workers;

    std::atomic<bool>stop;
    std::vector<std::condition_variable>condition_queue;

};

inline ThreadPool::ThreadPool(size_t threads): stop(false) {
    queues.resize(threads);
    queues_mutexes.resize(threads);
    for(size_t i=0;i<threads;++i) {
        workers.emplace_back(
            [this,i](){
                this->work(i);
            }
        );
    }
}

inline void ThreadPool::work(size_t my_index) {
    while(!this->stop.load()) {
        std::function<void()>task;
        bool should_wait=true;

        {
            std::unique_lock<std::mutex>lock(this->queues_mutexes[my_index]);
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
            std::unique_lock<std::mutex>lock(this->queues_mutexes[my_index]);
            if(this->stop.load()) {
                break; //in case of the signal turns to true suddenly
            }
            if(!this->queues[my_index].empty()) {
                // a task was pushed suddenly
                continue;
            }
            this->condition_queue[my_index].wait(lock,
            [this,my_index](){
                return this->stop.load() || !this->queues[my_index].empty();});
        }
    }
}

inline std::function<void()> ThreadPool::steal(size_t my_index) {
    std::function<void()> task;
    for(size_t attempt=0;attempt<2;++attempt) { // two rounds
        for(size_t cnt=0;cnt<this->queues.size();++cnt) { //should be random
            std::random_device rd;
            std::mt19937 gen(rd());

            std::uniform_int_distribution<>dis(0,this->queues.size());
            size_t i=dis(gen);
            if(i==my_index || this->queues[i].empty()) {
                continue;
            }

            //if the deque is not empty,try to acquire the lock 
            {
                std::unique_lock<std::mutex>lock(this->queues_mutexes[i],
                    std::try_to_lock);//if fails to acquire,it won't wait here
                if(lock.owns_lock()) {
                    task=std::move(this->queues[i].front());
                    this->queues[i].pop_front();
                    lock.unlock();
                    return task;//steal successfully
                } else {
                    continue;//if fails to steal,try to steal another deque
                }
            }
        }
    }
    return task;
}

template<class F,class ...Args>
auto ThreadPool::enqueue(F&&f,Args&& ... args) {
    using return_type=std::invoke_result_t<F,Args...>;
    auto task=std::make_shared<std::packaged_task<return_type()>>( 
        std::bind(std::forward<F>(f),std::forward<Args>(args)...));
    std::future<return_type>res=task->get_future();
    size_t candidate=0;
    size_t min_size=INT_MAX;
    for(size_t i=0;i<this->queues.size();++i) {
        size_t curr_size=this->queues[i].size();
        if(curr_size<min_size) {
            min_size=curr_size;
            candidate=i;
        }
    }

    {
        std::unique_lock<std::mutex>lock(this->queues_mutexes[candidate]);
        //no matter how,enqueue must get the lock so that it can push task
        //so it has to wait here
        if(stop.load()) {
            throw std::runtime_error("enqueue on stopped ThreadPool");
        }
        this->queues[candidate].emplace_back([this](){(*task)();});
        this->condition_queue[candidate].notify_one();//wake the thread precisely
    }
    return res;
}

inline ThreadPool::~ThreadPool() {
    this->stop.store(true);//should I appoint memory order?
    for(size_t i=0;i<this->condition_queue.size();++i) {
        condition_queue[i].notify_one();
    }
    for(std::thread &worker:workers) {
        worker.join();//make sure all work finished
    }
}
```



依然有几个问题

1.随机数生成器的性能问题与边界问题

```c++
std::random_device rd;  // 每次循环都创建随机设备
std::mt19937 gen(rd()); // 每次循环都初始化随机引擎
std::uniform_int_distribution<>dis(0,this->queues.size()); // 边界错误
```

std::random_device是一个真正的随机数生成器，它通常读取硬件熵源，涉及系统调用（速度很慢，可能需要数千个CPU周期），在某些系统上甚至可能阻塞，等待足够的熵积累

假设有8个线程，每个窃取尝试2轮，那么每轮尝试8次，这意味着要调用std::random_device 128次，每次调用都可能涉及系统调用，这是性能上的灾难

std::met19937是梅森旋转算法，它有19937位的内部分状态，初始化时需要填充这个巨大的状态数组，初始化成本相对较高（每次都要从random_device获取种子，初始化19937位的状态数组）

std::uniform_int_distribution<>dis(0,this->queues.size())会产生[0,queues.size()]的随机数，但有效的索引最大值是queues.size()-1

2.enqueue中的竞态条件

读取queues[i].size()时没有加锁，可能读到旧的值

```c++
    for(size_t i=0;i<this->queues.size();++i) {
        size_t curr_size=this->queues[i].size();
```

3.内存序问题：

```c++
    this->stop.store(true);//should I appoint memory order?
```

4.任务包装问题：不应该捕获this指针

```c++
this->queues[candidate].emplace_back([this](){(*task)();});
```

5.最严重的：加了condition_queue后，忘记将它初始化了！

解决方案：

1.使用thread_local：每个线程第一次进入函数时初始化第一次，后续直接调用已初始化的对象，且线程安全

2.不找最小队列了，不然要加锁。直接改成简单轮询

解释：简单轮询就是第一次给线程0，第二次给线程1，第三次给线程2...我们把下一个要找的线程编号next_index设置为静态原子变量，这样所有线程都可以共用它。调用fetch_add使它自增1，并接受它自增1之前的值candidate，作为当前找的线程编号。fetch_add的用法在Day05末尾有提到，此处不再赘述

3.简单的布尔变量，用relaxed即可

4.改为按值捕获task指针，后续原对象被销毁后还能操作副本。这也是在Day03中提到的，要让这个对象的生命周期超越enqueue函数的生命周期



第三稿：

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

class ThreadPool {
public:
    ThreadPool(size_t threads);

    template<class F,class ...Args>
    auto enqueue(F&&f,Args&& ... args);
    
    ~ThreadPool();

private:

    void work(size_t my_index);
    std::function<void()> steal(size_t my_index);

    std::vector<std::deque<std::function<void()>>>queues;
    std::vector<std::mutex>queues_mutexes;

    std::vector<std::thread>workers;

    std::atomic<bool>stop;
    std::vector<std::condition_variable>condition_queue;

};

inline ThreadPool::ThreadPool(size_t threads) {
    stop.store(false,std::memory_order_relaxed);
    queues.resize(threads);
    queues_mutexes.resize(threads);
    condition_queue.resize(threads);
    for(size_t i=0;i<threads;++i) {
        workers.emplace_back(
            [this,i](){
                this->work(i);
            }
        );
    }
}

inline void ThreadPool::work(size_t my_index) {
    while(!this->stop.load(std::memory_order_relaxed)) {
        std::function<void()>task;
        bool should_wait=true;

        {
            std::unique_lock<std::mutex>lock(this->queues_mutexes[my_index]);
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
            std::unique_lock<std::mutex>lock(this->queues_mutexes[my_index]);
            if(this->stop.load(std::memory_order_relaxed)) {
                break; //in case of the signal turns to true suddenly
            }
            if(!this->queues[my_index].empty()) {
                // a task was pushed suddenly
                continue;
            }
            this->condition_queue[my_index].wait(lock,
            [this,my_index](){
                return this->stop.load(std::memory_order_relaxed) || !this->queues[my_index].empty();});
        }
    }
}

inline std::function<void()> ThreadPool::steal(size_t my_index) {
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
                std::unique_lock<std::mutex>lock(this->queues_mutexes[i],
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
auto ThreadPool::enqueue(F&&f,Args&& ... args) {
    using return_type=std::invoke_result_t<F,Args...>;
    auto task=std::make_shared<std::packaged_task<return_type()>>( 
        std::bind(std::forward<F>(f),std::forward<Args>(args)...));
    std::future<return_type>res=task->get_future();
    static std::atomic<size_t>next_index{0};
    size_t candidate=next_index.fetch_add(1,std::memory_order_relaxed)%this->queues.size();

    {
        std::unique_lock<std::mutex>lock(this->queues_mutexes[candidate]);
        //no matter how,enqueue must get the lock so that it can push task
        //so it has to wait here
        if(stop.load(std::memory_order_relaxed)) {
            throw std::runtime_error("enqueue on stopped ThreadPool");
        }
        this->queues[candidate].emplace_back([task](){(*task)();});
        this->condition_queue[candidate].notify_one();
    }
    return res;
}

inline ThreadPool::~ThreadPool() {
    this->stop.store(true,std::memory_order_relaxed);
    for(size_t i=0;i<this->condition_queue.size();++i) {
        condition_queue[i].notify_one();
    }
    for(std::thread &worker:workers) {
        worker.join();//make sure all work finished
    }
}
```

因为虚拟机还没安装中文输入法，所以注释都暂时用一些简单的英文写了

| **如何访问其他线程的队列？** | 使用全局的 `vector` 来存储所有队列，线程通过索引 (`my_index`) 访问 |
| ---------------------------- | ------------------------------------------------------------ |
| **`enqueue` 任务如何分配？** | 使用原子变量 `next_index` 进行轮询（Round-Robin）            |
| **随机数生成性能瓶颈**       | 使用 `thread_local` 存储随机数生成器                         |
| **窃取时的锁竞争**           | 使用 `std::try_to_lock` 尝试窃取，失败立即放弃，不阻塞       |
| **条件变量的使用**           | 将条件变量等待设为**最低优先级**，先处理本地任务，再窃取，最后才等待 |

















