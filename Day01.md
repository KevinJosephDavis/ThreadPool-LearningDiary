## ThreadPool Learning Day01

:question:3个问题：

:one:ThreadPool 类的公共接口有哪些？

:two:它的数据成员有哪些？

:three:enqueue函数的参数是什么？返回值类型是什么？为什么要这样设计？

```c++
class ThreadPool {
public:
	ThreadPool(size_t);
	template<class F, class...Args>
	auto enqueue(F&& f, Args&&...args)
		-> std::future<typename std::result_of<F(Args...)>::type>;
	~ThreadPool();
private:
	std::vector<std::thread>workers;
	std::queue<std::function<void()>>tasks;
	std::mutex queue_mutex;
	std::condition_variable condition;
	bool stop;
};
```

A1：公共接口：

构造函数、enqueue函数、析构函数。



A2：数据成员：

工作线程组：不断从任务队列取出任务并执行。

任务队列：存储待执行的任务，是一个函数对象的队列。

同步原语：std::mutex和std::condition_variable. 作用是协调工作线程和任务提交者。std::mutex保护任务队列，防止多线程同时访问导致数据竞争。std::condition_variable用于通知工作线程。当有新任务到来时，通知等待的线程。当线程池停止时，通知所有线程结束。

停止标志：用于关闭线程池。所有工作线程在执行完当前任务后都会退出。



A3：深入分析一下enqueue函数。

:one:template<class F,class...Args>

```c++
template<class F, class...Args>
auto enqueue(F&& f, Args&&...args)
		-> std::future<typename std::result_of<F(Args...)>::type>;
	~ThreadPool();
```

class F表示一个可调用对象类型，如函数、lambda、函数对象等。

class...Args表示0个或多个参数类型（这是一个可变参数模板）。

如果没有模板，我们只能接收特定类型的任务：

```c++
//只能接受无参数、无返回值的函数
void enqueue(std::function<void()>task);
    
//只能接受int参数的函数
void enqueue(std::function<void(int)>task);
```

有了模板，我们可以接受任意任务：

```c++
pool.enqueue([]{return 20;});//lambda
pool.enqueue(regular_function,arg1,arg2);//普通函数
```

例如：

```c++
int add(int a,int b){return a+b;}
pool.enqueue(add,2,3);

class MyClass {
public:
    void process(int value){/*...*/}
};
MyClass obj;
pool.enqueue(&MyClass:process,&obj,42);
```

为什么MyClass:process要加&？因为MyClass::process是成员函数，成员函数作为参数传递时必须加&.

```c++
//普通函数：两种方式都可以
int func(int x) {return x;}

pool.enqueue(func,10);//正确，函数名退化为指针
pool.enqueue(&func,10);//也正确

pool.enqueue(MyClass:process,&obj,10);//错误
pool.enqueue(&MyClass::process,&obj,10);//正确
```

pool.enqueue(func,10)的正确性基于一个事实：C++中函数或数组作为参数传递时会退化为指针。

obj前要加&，使用引用传递，使得我们创建的obj真正被改变并且避免拷贝。如果不使用引用传递，那么成员函数会作用在副本上，副本在任务完成后被销毁，原始的obj完全没有被修改。使用引用传递，成员函数通过this指针操作原始对象，原始obj被正确修改。

另外，Args是和可调用对象F绑定的参数。当编译器看到：

```c++
pool.enqueue(add,2,3);
```

它会实例化一个特定的enqueue版本：

```c++
std::future<int> enqueue(int (*f)(int,int),int&& arg1,int&& arg2)
{
    //因为函数名作为参数传递会退化为指针，所以这里推导为*f
    //...具体实现
}
```

不同类型的任务实例化不同的模板。

:two:&&

```c++
template<class F, class...Args>
auto enqueue(F&& f, Args&&...args)
		-> std::future<typename std::result_of<F(Args...)>::type>;
	~ThreadPool();
```

这里的&&不是右值引用，而是转发引用（万能引用）。

**作用：保持值类别**：如果传入左值，f和args就是左值引用；如果传入右值，就是右值引用。但是必须与std::forward结合才能保持值类别。

例：

```c++
//左值引用
int x=10;
std::string s="hello";
pool.enqueue(func,x,s);//f和args是左值引用

//右值引用
pool.enqueue(func,20,std::string("world"));//f和args是右值引用
```

**这样可以避免不必要的拷贝。左值保留原有对象，右值可以安全地移动**

涉及到了左右值，复习一下：

**左值：有持久身份的对象（有名字，可以多次使用），可以取地址**

**右值：临时对象或即将销毁的对象（通常只能用一次）**

```c++
int a=10;//a是左值，有名字：a
int b=a;//多次使用a

int c=10+20;//10+20是右值（临时结果）
int d=std::move(a);//std::move(a)产生右值引用

//不能只将左值理解为可修改的值
const int a;
a=5;//错误，但a是左值
```

一个例子理解移动（move）：

```c++
std::string s1="hello";
std::string s2=s1;//拷贝：s1的内容复制到s2，s1保持不变
std::string s3=std::move(s1);//移动：s1的内容转移到s3，s1变为空
```

在转发引用的应用：

```c++
template<typename T>
voie f(T&&arg){
    process(std::forward<T>(arg));//完美转发
}
```

场景一：传递左值

```c++
std::string s="hello";
f(s);//T推导为std::string&，arg是左值引用
	 //std::forward保持左值特性，调用process(s)-拷贝
//不是说，拷贝就一定不好。设计目的是按照调用者的意图传递参数，而非一直追求最佳性能
```

场景二：传递右值

```c++
f(std::string("hello"));
//T推导为std::string，arg是右值引用
//std::forward转换为右值，调用process(std::move(arg))-移动
```

场景三：传递std::move

```c++
std::string s="hello";
f(std::move(s));
//T推导为std::string，arg是右值引用
//std::forward转换为右值，调用process(std::move(arg))-移动
```

再看一个更能体现移动优势的例子：

```c++
std::vector<int>data(100000000);
pool.enqueue(process,data);//拷贝整个vector，成本太高
pool.enqueue(process,std::move(data));//移动更高效
poo.enqueue(process,std::vector<int>(100000000));//或者直接传个右值让模板自动推导
```

另外，上面涉及到了完美转发，复习一下：

std::forward的核心作用：保持值类别。传来的是左值，就保持左值特性。传来的是右值，就保持右值特性。

如果没有std::forward：

```c++
//转发引用会消灭右值性
void f(T&&arg) { //arg现在可能是左值引用，也可能是右值引用
    process(arg);//这里arg总是会被当成左值
}
```

**完美转发=转发引用+std::forward**

**:point_right:转发引用：在参数接收端保持值类别**

:point_right:**std::forward：在参数使用端恢复值类别**

**这个经典线程池项目里为什么没有使用std::forward呢？使用了，只不过不是在这里。**

这么看，转发引用是不是显得有点鸡肋？并不是。

如果没有转发引用：

```c++
template<typename T>
void f(T arg) { //这里不是转发引用，是值传递
    process(std::forward<T>(arg));
}

//调用：
std::string s="hello";
f(s);//拷贝构造arg
f(std::move(s));//移动构造arg
```

对于f(s)，std::forward是没有意义的：因为已经发生了拷贝：成本最高的操作已经完成了。

如果传入的是一个右值，分为两种情况讨论：

```c++
template<typename T>
void f(T arg) { //值传递
    //使用arg
}

//1.传入纯右值
f(std::string("hello"));

//2.传入将亡值
std::string s="hello";
f(std::move(s));
```

1.传入纯右值（prvalue）：

T推导为std::string，在函数f的栈上构造arg对象，临时对象std::string("hello")的内容被移动到arg.

2.传入将亡值（xvalue）：

T推导为std::string，参数构造：

```c++
std::string arg(std::move(s)) //移动构造
```

s的内容被移动到arg，s变为空字符串。

所以对于右值来说，值传递的性能是好的，因为只有廉价的移动操作。

不过，虽然性能上没问题，但可能会有潜在的风险。

```c++
template<typename T>
void f(T arg) {
    process(std::forward<T>(arg));
}

void process(std::string&&data) {
    //process认为data是右值，可以安全移动
    str=std::move(data);
}

std::string s="hello";
f(std::move(s));
```

data是arg的右值引用，但arg是f的局部变量。移动后arg变为空。f函数结束后，arg析构。如果process存储了引用，会导致悬空引用。

```c++
std::string* ptr=nullptr;
void process(std::string&&data) {
    ptr=&data;
}
f(std::move(s));

//现在ptr指向已经销毁的arg，会导致未定义行为
```

**总结一下：std::forward要与转发引用配合使用才能起到“保持值类型”的作用。如果std::forward没有转发引用，即进行值传递，那么左值会拷贝（成本高），右值虽然不会有性能问题，但是会有潜在风险。如果转发引用没有std::forward，使用参数的时候始终是左值类别。**

:three:typename

```c++
template<class F, class...Args>
auto enqueue(F&& f, Args&&...args)
		-> std::future<typename std::result_of<F(Args...)>::type>;
	~ThreadPool();
```

F(Args...)就是函数名（参数）的抽象写法，因为我们使用的是模板。

那么result_of<F(Args...)>表示这个函数的返回值。T::something一般会认为是一个值，所以这里要加typename告诉编译器T::type是一个类型，不是一个值。

:four:返回值类型：std::future

用一个直观的方式理解std::future：

```c++
//点外卖（提交任务）
std::future<Food>order=restaurant.enqueue(pizza,"大份");

//点完外卖后我们可以做其他事情，没必要干等外卖（非阻塞）
WatchTV();
Cleaning();

//饿了，取外卖（获取结果）
Food food = order.get();//这里会阻塞，如果披萨还没做好，那么就等到做好

//如果外卖员送错了（任务抛出异常）
try {
    Food food=order.get();
} catch (const WrongFoodError& e) {
    //处理异常
}
```

如果没有future：

```c++
pool.enqueue([](){
    int res=cal();
    //结果无法返回给调用者
}
```



关于阻塞，举一个极端例子：

```c++
std::future<int>res=pool.enqueue([]() {
    std::this.thread::sleep_for(std::chrono::hours(1));//模拟1h计算
    return 10;
});

//主线程继续执行
task();

//假设过了1分钟后，主线程执行到这里
int value=res.get();//会在这里阻塞59分钟
```

也就是说，future的核心功能之一就是异步获取结果。而且，能通过.get()将结果返回给调用者（如果使用全局变量，线程不安全）。此外，还能抛出异常。

此外，future_status还能与wait_for搭配使用。

情景一：检查是否完成（非阻塞）

```c++
std::future<int>res=pool.enqueue(耗时任务);

//立即检查
if(res.wait_for(std::chrono::seconds(0))==std::future_status::ready) {
    //解释：如果等待0秒钟后的状态是ready，即立即判断当前状态是否为ready
}
```

情景二：设置最大等待时间（有限阻塞）

```c++
std::future<int>res=pool.enqueue(耗时1h任务);

//最多等待5分钟
auto status=res.wait_for(std::chrono::minutes(5));
if (status==std::future_status::ready) {
    //如果任务在5分钟内完成了
} else if (status==std::future_status::timeout) {
    //如果等了5分钟还没有完成
}
```

**std::future是一个轻量级的包装器**

**通俗地说，它就是”收货凭证“**

情景：线上购物。

1.用户下单购买商品（提交任务）；

2.平台给用户一个订单号（返回std::future）；

3.用户可以继续做其他事情（主线程不阻塞）;

4.可凭订单号取货（调用.get()获取结果）；

5.如果商品有问题，可以退货/投诉（异常处理）；

**总结：std::future的核心功能：**

**1.提供线程安全的返回值存储；**

**2.实现生产者与消费者的同步；**

**3.管理异步操作的生命周期；**

**4.提供统一的异常处理机制。**



