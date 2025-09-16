## ThreadPool Learning Day03

:question:3个问题：

:one:enqueue函数如何实现？

:two:析构函数做了什么？

:three:整体分析：这几个函数是如何配合的？

一、enqueue函数

由第一天的分析，我们知道enqueue函数的功能就是将任务提交到任务队列，并且确保这个任务是一个返回值为void，参数列表为空的可调用对象。

```c++
	template<class F, class...Args>
	auto enqueue(F&& f, Args&&...args)
		-> std::future<typename std::result_of<F(Args...)>::type>;
```

```c++
// add new work item to the pool
template<class F, class... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args)
    -> std::future<typename std::result_of<F(Args...)>::type>
{
    using return_type = typename std::result_of<F(Args...)>::type;

    auto task = std::make_shared< std::packaged_task<return_type()> >(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...)
    );

    std::future<return_type> res = task->get_future();
    {
        std::unique_lock<std::mutex> lock(queue_mutex);

        // don't allow enqueueing after stopping the pool
        if(stop)
            throw std::runtime_error("enqueue on stopped ThreadPool");

        tasks.emplace([task](){ (*task)(); });
    }
    condition.notify_one();
    return res;
}
```

:one:using return_type = typename std::result_of<F(Args...)>::type

有了第一天的分析，我们知道typename std::result_of<F(Args...)>::type是为了在编译器推导出用户提交的函数F在用参数Args...调用后的返回类型是什么。假设返回值为T，那么T::type会被编译器认为是成员变量，因此要加上typename告诉编译器T::type是一个类型名

using return_type实际上是在起别名。这样我们就不用每次都写typename std::result_of<F(Args...)>::type这么一大长串了。

根据第一天的分析，我们要返回一个std::future给用户，让用户能够异步地获取他们提交的任务的执行结果。std::future是一个模板类，它需要知道它将来要获取的值的类型，所以我们需要创建一个std::future<return_type>对象。例如，用户提交的任务返回值为int，我们就需要返回一个std::future<int>；用户提交的任务返回值为void，我们就需要返回一个std::future<void>

所以这行代码就是为了能正确声明std::future<return_type>

一个具体例子：

```c++
int add(int a, int b) { return a + b; }

ThreadPool pool(4);
// 用户提交一个任务，希望得到结果
std::future<int> result = pool.enqueue(add, 2, 3);
```

由第一天分析可知，在enqueue函数模板实例化时：

F被推导为int (*)(int,int)

Args...被推导为(int,int)

std::result_of<F(Args...)>::type就变成了：std::result_of<int (*)(int,int)>::type

编译器会计算这个表达式，得到::type就是int

进而return_type就是int

所以enqueue函数的返回类型就是std::future<int>

需要注意的是，在C++17中，std::result_of已被弃用，在C++20中已被移除。现在使用std::invoke_result_t

```c++
using return_type = std::invoke_result_t<F, Args...>;
```

联想到昨天学习的std::function<void()>，它和模板enqueue函数都体现了多态性，但是它们属于不同种类的多态。

std::function<void()>   ->   运行时多态

机制：基于继承、虚函数和动态绑定

特点：在运行时决定调用哪个具体的函数。对方究竟是什么对象我并不在乎，只要长得像void()，我就能用

代价：通常伴随一些运行时开销，如虚函数表查找、动态分配

模板   ->   编译时多态

机制：基于模板和编译期的代码生成

特点：编译时，编译器会为每一种被使用的类型组合生成一份特定的代码。类型信息在编译期就完全确定。对方究竟是什么类型，传进来我再判断，并为特定类型生成一份专属的代码

代价：可能导致代码膨胀，但几乎没有运行时开销

**总结：std::function<void()>像通用的遥控器，模板函数enqueue像万能工厂**

**这行using return_type代码体现了编译时多态，它使得这个enqueue函数模板能够：**

**1.自动推导出用户提交任务的返回类型**

**2.根据这个类型，构造并返回一个正确类型的std::future对象**

**3.让线程池能够以一种类型安全的方式，统一处理并返回任何类型的任务**



:two:std::bind\std::packaged_task\std::make_shared

```c++
    auto task = std::make_shared< std::packaged_task<return_type()> >(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...)
    );
```

先看内层：std::bind -参数绑定器

作用：将一个可调用对象与其参数绑定在一起，生成一个新的可调用对象

在线程池中的作用：将用户传入的函数f和参数args提前打包好，创建一个不需要再传参的可调用对象

举个例子：

```c++
// 假设用户调用pool.enqueue([](int a, int b){ return a+b; }, 2, 3)

// std::bind 会做这样的事：
auto bound_func = std::bind( [](int a, int b){ return a+b; }, 2, 3 );

// 现在 bound_func 是一个无参的可调用对象，调用 bound_func() 等价于调用 lambda(2, 3)，返回 5
```

std::forward<F>(f)和std::forward<Args>(args)...是完美转发，我们在第一天已经分析过了，它可以确保传递的参数保持其原始的值类别

追问1：std::bind生成的新的可调用对象，其参数列表是否必然为空？

答：不一定。只不过我们在线程池项目中，让它变成了空的。

std::bind的核心机制是参数绑定和占位符。

如果我们为可调用对象的所有参数都提供了具体的值，那么std::bind生成的新可调用对象就是一个无参对象

```c++
auto f1 = std::bind(func, 1, 2); // 绑定了所有参数
f1(); // 调用时不需要再提供参数，等价于 func(1, 2)
```

如果我们使用占位符std::placeholders::_1 那么生成的可调用对象就需要在调用时提供这些参数

```c++
auto f2 = std::bind(func, std::placeholders::_1, 2); // 只绑定了第二个参数
f2(10); // 调用时需要提供第一个参数，等价于 func(10, 2)
```

在线程池的上下文中：

```c++
std::bind(std::forward<F>(f), std::forward<Args>(args)...)
```

我们使用(args)...包展开了所有参数，这意味着我们为函数f的每一个形参都提供了一个具体的实参，因此这里std::bind生成的结果就是一个无参的可调用对象

追问2：std::bind的返回值类型？

答：std::bind返回的是一个未指定的、实现定义的函数对象模型，这意味着我们可以用auto来接受它（编译器知道具体类型，只是这个类型名字又长又复杂，而且是编译器内部的，对程序员不可见），但我们几乎不能在实际工程中直接使用这个类型，因为它不通用。这正是std::function存在的主要原因，它提供了类型擦除的包装器，可以存储任何签名相符的可调用对象

```c++
// 正确且通用的做法：用 std::function 来存储和传递
std::function<int()> stored_func = std::bind(myFunction, 42, "hello");
// 现在 stored_func 可以被传递、放入容器等
```



再看中层：std::packaged_task 承诺任务包装器

```c++
// add new work item to the pool
template<class F, class... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args)
    -> std::future<typename std::result_of<F(Args...)>::type>
{
    using return_type = typename std::result_of<F(Args...)>::type;

    auto task = std::make_shared< std::packaged_task<return_type()> >(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...)
    );

    std::future<return_type> res = task->get_future();
    {
        std::unique_lock<std::mutex> lock(queue_mutex);

        // don't allow enqueueing after stopping the pool
        if(stop)
            throw std::runtime_error("enqueue on stopped ThreadPool");

        tasks.emplace([task](){ (*task)(); });
    }
    condition.notify_one();
    return res;
}
```

作用：将一个可调用对象包装起来，并将其返回值与一个std::future对象关联

```c++
// 用 bind 的结果创建一个 packaged_task
std::packaged_task<return_type()> task( std::bind(...) );

// 可以从 packaged_task 中获取一个 future，用于将来获取结果
std::future<return_type> future = task.get_future();

// 将来在某个线程中执行这个任务
task(); // 执行绑定的函数，并将结果存入内部的共享状态
// 此时，future.get() 就可以拿到结果了
```

return_type()是一个函数签名，表示这个packaged_task包装的是一个无参数且返回return_type类型值的可调用对象。这正是我们需要的，因为std::bind已经帮我们把参数都绑定好了，现在只需要一个无参的调用。



最后看外层：std::make_shared  共享指针创建器

```c++
// add new work item to the pool
template<class F, class... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args)
    -> std::future<typename std::result_of<F(Args...)>::type>
{
    using return_type = typename std::result_of<F(Args...)>::type;

    auto task = std::make_shared< std::packaged_task<return_type()> >(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...)
    );

    std::future<return_type> res = task->get_future();
    {
        std::unique_lock<std::mutex> lock(queue_mutex);

        // don't allow enqueueing after stopping the pool
        if(stop)
            throw std::runtime_error("enqueue on stopped ThreadPool");

        tasks.emplace([task](){ (*task)(); });
    }
    condition.notify_one();
    return res;
}
```

作用：在动态内存（堆）上创建一个对象，并返回一个指向它的std::shared_ptr（共享指针）

为什么需要这么做？因为我们要让packaged_task对象的生命周期超越当前作用域（enqueue函数）

:question: 追问：为什么要让其生命周期超越当前作用域？

答：因为我们需要实现异步执行。enqueue函数和真正执行任务的线程是两个不同的线程流，它们的生命周期是分离的。

1.enqueue函数中（生产者线程）：

创建packaged_task

获取与之关联的future

将任务放入队列

enqueue函数返回，它的栈空间即将被销毁

2.未来的某个时刻（消费者线程）：

工作线程从队列中取出任务

执行任务，需要调用packaged_task对象

如果packaged_task是创建在enqueue函数的栈上，那么当enqueue函数返回时，这个packaged_task对象就会被销毁。当工作线程稍后尝试执行它时，它访问的是一个已经被销毁的对象，会导致未定义行为。



此外，Lambda表达式：

```c++
[task](){ (*task)(); }
```

需要捕获这个package_task

如果直接按值捕获，会涉及到拷贝，**而packaged_task是不可拷贝的**。

因此，我们需要将packaged_task放在堆上，并用一个shared_ptr管理它的生命周期。这样lambda表达式只需捕获这个轻量的shared_ptr的副本即可。

:question: 追问：为什么packaged_task不可拷贝？

答：如果被拷贝，会有如下问题：

1.该执行哪个副本？如果两个副本都被执行了operator()，那么用户的一个任务就会被执行两次。而且两次执行会竞争同一个future的共享状态。

如何理解两次执行会竞争同一个future的共享状态？什么是共享状态？

共享状态：通信的桥梁

std::packaged_task和它通过.get_future()获取的std::future并不是两个相互独立的对象。它们通过一个共同的、在堆上分配的共享状态连接在一起。

比喻理解：线上会议

std::packaged_task就像发言人，它负责说话，并将结果告诉会议系统

std::future就像听众，它连接到会议系统，等待听取发言人的结果

共享状态：会议系统本身

可以这么理解：共享状态是一块在堆上动态分配的内存区域，充当了std::packaged_task和std::future之间通信的桥梁和中间缓存。

```c++
std::packaged_task<int()> task([]{ return 42; });
// 此时，库实现会在堆上分配一块内存作为“共享状态”
// 这块内存可能包含：结果值、异常指针、状态标志（是否就绪）、引用计数等

std::future<int> fut = task.get_future();
// 现在，`task` 和 `fut` 内部都持有一个指向这块堆内存的指针（或类似的东西）
```

执行：

```c++
task(); // 执行任务
// 内部发生：
//   1. 调用绑定的函数，得到结果 42。
//   2. 通过内部的指针，找到堆上的共享状态内存。
//   3. 将结果 42 写入这块内存的特定位置。
//   4. 将一个标志位（如 `ready_flag`）设置为 true。
//   5. 可能还会通知等待中的线程。
```

获取：

```c++
int result = fut.get();
// 内部发生：
//   1. 通过内部的指针，找到堆上的共享状态内存。
//   2. 检查标志位，如果为 false 则阻塞等待。
//   3. 如果为 true，则从内存中读取结果值 42。
//   4. 返回这个值。
```

std::packaged_task拥有写权限，它负责向共享状态写入结果（或异常）并将状态标记为就绪

std::future拥有读权限。它负责从共享状态读取结果（或异常）并检查状态。它可以被多次拷贝，多个future副本可以共享读权限。

共享状态本身：由标准库在堆上创建和管理。它的生命周期由引用计数控制，当最后一个指向它的packaged_task或future被销毁时，这块内存才会被释放。

理解了共享状态，我们回到刚刚的问题，为什么两个pakcaged_task副本会指向同一个共享状态？

如果std::packaged_task有拷贝构造函数，大概率也是浅拷贝。因为如果是深拷贝，创建一份共享状态的完整副本，相当于任务执行了两次。

如果是浅拷贝，只复制内部的指针，新副本和原对象指向同一个共享状态，可能发生数据竞争，如同两个发言人同时对一个麦克风说话。即使没有数据竞争，最终也只有一个结果被设置，另一个结果会被覆盖或丢失

这也引出了问题2：结果该放到哪里？两个副本都尝试设置结果，或者一个设置结果，另一个设置异常，共享状态会被破坏。

因此，std::packaged_task禁止拷贝，但允许移动

```c++
std::packaged_task<int()> task1([]{ return 42; });
// std::packaged_task<int()> task2 = task1; // 错误：拷贝构造被禁用
std::packaged_task<int()> task2 = std::move(task1); // 正确：移动构造
// 现在 task1 变为无效状态（可安全析构但不可再使用），task2 获得了所有权
```



:question:追问：为什么是shared_ptr？

答：首先，为什么不是原始指针？因为我们不知道工作线程什么时候完成任务。如果工作线程执行完后删除了对象，但其它地方还持有这个指针，又会导致悬空指针。原始指针无法回答“这个对象应该活多久”的问题。

其次，为什么不是unique_ptr？std::unique_ptr代表独占所有权，一个资源只能被一个unique_ptr拥有。而下面的lambda表达式：

```c++
[task](){ (*task)(); }
```

有按值捕获指针的操作，会复制指针副本，意味着副本和原对象管理同一个对象，那就必然不可能是unique_ptr

能否移动它呢？

```c++
tasks.emplace([task = std::move(task)](){ (*task)(); }); // 移动捕获
```

致命问题：移动后，enqueue函数中的原始task指针变成了nullptr，后面获取future的操作task->get_future()实际上是在对一个空指针进行操作。虽然说在lambda表达式之后没有对task的操作，但是我们必须十分小心，不能再使用task，扩展性不如shared_ptr

所以shared_ptr的共享所有权语义更直观地表达了“这个对象需要被多个地方调用”

时间线：

1. make_shared()           ->    引用计数 = 1 (enqueue局部变量)
2. lambda捕获                ->    引用计数 = 2 (enqueue变量 + lambda副本)
3. enqueue函数返回      ->    引用计数 = 1 (只有lambda副本在队列中)
4. 工作线程取出副本       ->    副本从队列移到线程栈，引用计数仍为1
5. 工作线程执行副本       ->    执行 (*task)()，引用计数仍为1
6. 副本执行完毕               ->    副本销毁 -> shared_ptr销毁 -> 引用计数 = 0 -> packaged_task销毁

**总结：**
**这行代码的执行顺序与效果：**

**1.最内层 std::bind**

**输入：用户函数f及其参数args...**

**输出：一个无参的可调用对象，调用它等价于用args...参数调用f**

**2.中间层 std::packaged_task<return_type()>(...)**

**输入：bind产生的无参可调用对象**

**输出：一个packaged_task对象。它包装了第一步的无参可调用对象，并提供了get_future接口，便于异步获取结果**

**3.最外层 std::make_shared<...>(...)**

**输入：第二步创建的packaged_task对象**

**输出：一个shared_ptr，指向在堆上创建的packaged_task对象**

**最终结果：变量task是一个智能指针，它指向一个在堆上的对象，这个对象包装了用户想要执行的函数和参数，提供了可异步获取其执行结果的接口，可以被安全地传递和捕获，不会造成内存泄漏。**



:three:锁

```c++
// add new work item to the pool
template<class F, class... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args)
    -> std::future<typename std::result_of<F(Args...)>::type>
{
    using return_type = typename std::result_of<F(Args...)>::type;

    auto task = std::make_shared< std::packaged_task<return_type()> >(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...)
    );

    std::future<return_type> res = task->get_future();
    {
        std::unique_lock<std::mutex> lock(queue_mutex);

        // don't allow enqueueing after stopping the pool
        if(stop)
            throw std::runtime_error("enqueue on stopped ThreadPool");

        tasks.emplace([task](){ (*task)(); });
    }
    condition.notify_one();
    return res;
}
```

接着，调用task的get_future接口，便于以后通过res.get()异步获取任务结果

开辟临界区，unique_lock自动上锁

如果已经下班了，就不允许再接收任务了，抛出runtime_error

往任务队列里放置任务

临界区结束

为什么要用锁？

因为操作了stop和tasks这两个共享资源。如果不加锁，可能会有多个线程同时修改队列结构，对于stop，可能会有线程读到过期的数据，因为读操作可能不是原子操作。

**总之，在多线程环境中，对任何非线程安全的共享资源的访问（包括读写），都需要通过适当的同步机制（互斥锁、内存屏障、原子操作）来保护。**



:four:emplace？push？

```c++
// add new work item to the pool
template<class F, class... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args)
    -> std::future<typename std::result_of<F(Args...)>::type>
{
    using return_type = typename std::result_of<F(Args...)>::type;

    auto task = std::make_shared< std::packaged_task<return_type()> >(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...)
    );

    std::future<return_type> res = task->get_future();
    {
        std::unique_lock<std::mutex> lock(queue_mutex);

        // don't allow enqueueing after stopping the pool
        if(stop)
            throw std::runtime_error("enqueue on stopped ThreadPool");

        tasks.emplace([task](){ (*task)(); });
    }
    condition.notify_one();
    return res;
}
```

如果使用push，就要先将其构造为std::function<void()>对象，再将其push进去

```c++
auto lambda = [task](){ (*task)(); };  // 构造lambda
std::function<void()> func(lambda);    // 构造function（可能分配内存）
tasks.push(func);                      // 拷贝function到容器中
// 或者 tasks.push(std::move(func));   // 移动function

tasks.emplace([task](){ (*task)(); }); // 直接在容器内存中构造function
```

如果我们不使用std::move，那还增加了昂贵的拷贝操作，即传进去的func实际上是个副本

如果使用std::move，依然需要在push函数的栈上构造对象，然后放到容器中

哪怕用下面这种传入右值的方法：

```c++
tasks.push(std::function<void()>([task](){ (*task)(); }));
```

比喻理解：做菜。如果调用push+拷贝构造，那就相当于在外面餐馆看着厨师做好了菜，自己跟着做了一遍，端回家里吃。如果调用push+移动构造，那就相当于直接把外面餐馆厨师做的菜端回家里吃。如果调用emplace，就相当于原材料直接送到家里，自己在家做好菜，直接在家吃。

看三个场景：

场景一：构造开销大的对象

```c++
class ExpensiveObject {
    std::vector<int> large_data;
    std::unique_ptr<Resource> resource;
public:
    ExpensiveObject(std::vector<int> data, std::unique_ptr<Resource> res)
        : large_data(std::move(data)), resource(std::move(res)) {}
};

// push：先外部构造，再移动
auto obj = ExpensiveObject(std::move(data), std::move(res));
tasks.push(std::move(obj));  // 仍有外部构造步骤

// emplace：直接内部构造  
tasks.emplace(std::move(data), std::move(res)); // 无外部构造
```

场景二：只能移动的对象（unique_ptr）

```c++
std::unique_ptr<MyTask> task = create_task();

// push：必须命名临时变量
tasks.push(std::move(task));  // 需要显式std::move

// emplace：更简洁
tasks.emplace(create_task());  // 直接传递，无需命名
```

场景三：需要转换类型

```c++
// 如果lambda需要转换为function
auto lambda = [](int x) { return x * 2; };

// push：需要显式转换
tasks.push(std::function<int(int)>(lambda));

// emplace：隐式转换
tasks.emplace(lambda);  // 自动推导和转换
```

:question:追问：为什么emplace能够直接在原容器中构造而push不能？

答：这涉及到了我们第一天所讲的完美转发。第一天的总结并不够好，今天再来谈谈完美转发

**完美转发保持引用类型，从而在底层维护值类别的语义后果（左值语义允许拷贝、可以取地址；右值语义允许移动、资源可转移），使得能够选择最优的参数传递方式。**

容器内部的emplace签名：

```c++
template<typename... Args>
void emplace(Args&&... args) {  // 转发引用
    // 在容器内存中直接构造
    new (memory_location) T(std::forward<Args>(args)...);
}
```

转发引用：根据传入的实参的值类别推导出不同的引用类型。传入左值/左值引用，就推导为左值引用，然后T& &&=T&形成左值引用。传入右值/右值引用，就推导为普通类型，然后T&&形成右值引用。这是引用折叠规则的体现

```c++
template<typename T>
void func(T&& arg);  // 转发引用

// 传入左值：
int x = 10;
func(x);            // T 推导为 int&, T&& → int& && → int&

// 传入右值：
func(10);           // T 推导为 int, T&& → int&&

// 传入左值引用：
int& y = x;
func(y);            // T 推导为 int&, T&& → int& && → int&

// 传入右值引用：
int&& z = 10;
func(std::move(z)); // T 推导为 int, T&& → int&&
```

:question: 追问：为什么传入左值，推导为左值引用？

答：对于转发引用T&&，C++标准规定了一套特殊的推导规则

当函数参数是转发引用时，如果传入左值，会被推导为左值引用类型。如果传入右值，会被推导为普通类型

其设计目的是区分左值和右值。如果T总是被推导为普通类型，那么就丢失了左值信息



std::forward的功能是：如果是左值引用，就返回左值引用。如果是右值引用，就返回右值引用

简化的std::forward实现：

```c++
template<typename T>
T&& forward(typename std::remove_reference<T>::type& arg) {
    return static_cast<T&&>(arg);  // 保持引用类型
}

template<typename T>
T&& forward(typename std::remove_reference<T>::type&& arg) {
    return static_cast<T&&>(arg);  // 保持引用类型
}
```

:question:追问：为什么两个返回的都是T&&类型？

答：因为模板参数T不同。如果传入左值/左值引用，转发引用将T推导为左值引用LeftType&，那么T&&=LeftType& &&=LeftType&，折叠为左值引用。如果传入右值/右值引用，转发引用将T推导为普通右值RightType，那么T&&=RightType&&，即右值引用本身



**需要特别注意的是，有名字的右值引用变量本身是左值，因为它有标识符，只有通过std::move()才能恢复为右值**

```c++
int&& rref = 10;  // rref是有名字的右值引用变量

func(rref);       // rref是左值！T推导为int&
func(std::move(rref)); // 这才是右值，T推导为int
```

**这种设计的精妙之处在于：**

**1.统一语法：无论转发什么，都写std::forward<T>(arg)**

**2.类型推导完成工作：编译器根据T的类型自动选择正确的行为**

**3.零运行时开销：全是编译期类型操作**

**4.类型安全：保持原有的const和引用属性**

回到一开始的问题，为什么emplace可以但push不行

push的实现：

```c++
void push(const T& value) { // 需要拷贝
    new (分配的内存) T(value);
}

void push(T&& value) {      // 需要移动  
    new (分配的内存) T(std::move(value));
}
```

push的设计遵循一个明确的语义：将一个已经存在的对象放入容器中

而emplace接收构造参数而不是完整的对象

**总结一下，emplace更优的原因：**

**1.更直接的构造路径：避免任何可能的临时对象**

**2.更简洁的语法：无需显示指定类型，可以隐式转换**

**3.更安全的移动语义：避免意外拷贝**

**4.更好的通用性：适用于各种复杂情景**



:five:为什么condition这个共享资源没有放在临界区？

```c++
// add new work item to the pool
template<class F, class... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args)
    -> std::future<typename std::result_of<F(Args...)>::type>
{
    using return_type = typename std::result_of<F(Args...)>::type;

    auto task = std::make_shared< std::packaged_task<return_type()> >(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...)
    );

    std::future<return_type> res = task->get_future();
    {
        std::unique_lock<std::mutex> lock(queue_mutex);

        // don't allow enqueueing after stopping the pool
        if(stop)
            throw std::runtime_error("enqueue on stopped ThreadPool");

        tasks.emplace([task](){ (*task)(); });
    }
    condition.notify_one();
    return res;
}
```

如果在锁内通知，那么通知操作必然引起线程调度，由昨天总结的condition.wait()的性质可知，被唤醒的线程会原子地获取锁，但是锁还被当前线程持有，那么wait()又阻塞了。那就相当于这个广播没人响应了。这是严重的错误，不是性能优化。

**这种理解是有误的！**

通知在锁内或在锁外都是没错的，只不过通知在锁外在性能上稍有优化

正确的解释：如果在锁内通知，那么通知操作**可能**引起线程调度（注意，是可能！）。notify_all和notify_one的作用只是唤醒线程，仅仅是将线程的状态从“等待状态”变为“就绪状态”，不涉及线程调度操作！唤醒线程后，由于在锁内，被唤醒的线程发现锁不可用，于是只能阻塞。注意，虽然线程是原子地获取锁，但是也要等到锁可用才能获取锁！此外，发现锁不可用后的阻塞，是因为锁，而不是条件变量的wait()的阻塞！是互斥锁的阻塞，不是条件变量的阻塞

**优化点：**

**锁内通知：被唤醒的线程需要等待锁释放**

**锁外通知：被唤醒的线程可能立即获得锁**

**正确性：无论是锁内通知还是锁外通知，最终锁都会被释放，所以两种方式都是正确的**

:question:追问：那是什么引起线程调度的？

答：线程调度是由操作系统的调度器决定的，条件变量与锁只是提供信息给调度器

```c++
// 条件变量的等待
condition.wait(lock, predicate);
```

相当于：

1.释放锁

2.告诉操作系统："把我挂起，等有人通知再唤醒我"

3.进入阻塞状态（不再被调度）



```c++
// 条件变量的通知  
condition.notify_one();
```

相当于告诉操作系统："有一个线程可以醒来了"。但具体何时调度，由操作系统决定



```c++
std::mutex::lock();
```

如果锁被占用：

1.线程告诉操作系统："锁忙，把我挂起"

2.进入阻塞队列

3.当锁释放时，操作系统选择某个等待线程唤醒



```c++
condition.notify_one();
```

1.操作系统将对应线程从"阻塞队列"移到"就绪队列"

2.调度器在下次调度时可能选择该线程运行

3.线程尝试获取锁，如果锁可用则继续，否则再次阻塞



最终，返回std::future<return_type>类型的变量res，可以通过res异步获取结果



二、析构函数

```c++
// the destructor joins all threads
inline ThreadPool::~ThreadPool()
{
    {
        std::unique_lock<std::mutex> lock(queue_mutex);
        stop = true;
    }
    condition.notify_all();
    for(std::thread &worker: workers)
        worker.join();
}
```

为什么inline？见第二天的笔记。C++17之后，成员函数隐式内联，此处inline可省略。

析构函数操作共享资源stop（因此要加锁），发出下班信号，并告诉所有工人（notify_all），最后等待所有工人把手头上的事情做完就下班

join()是必要的，它确保了主线程等待所有其它线程结束了才结束。如果没有join()，还在运行的工作线程变成僵尸线程，它们所拥有的资源可能无法释放，最终导致资源泄露或未定义行为

另外，如果没有notify_all()，那么所有工人都不知道要下班，该干活的干活，在等待的工人也一直阻塞在wait()，等到干活的工人干完了也阻塞在wait()，这又导致了join()一直在阻塞



三、整体把握：这些函数如何配合使用

最后，整体把握一下整个流程。

创建线程池（包含提交函数enqueue，vector工人线程，条件变量，任务队列，互斥锁，停止标志），在构造函数中通过死循环让工人一直在岗（哪怕队列为空return了也还继续死循环），如果有活了就从任务队列取并执行。

怎么才能有活？需要通过enqueue函数提交任务。enqueue函数将任务包装成无参可调用对象，并用shared_ptr管理它，利用packaged_task与future的共享资源，创建一个std::future<return_type>类型变量，用于异步获取任务结果，这样才能将任务包装成返回值为void的无参调用对象，才能将任务放到队列当中，并提醒任意一个工人。

下班时，发送下班信号，等待所有工人做完手上的活，结束。

























