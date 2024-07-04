

## 1.9 定制析构

shared_ptr 的构造函数（ reset 方法）额外接收一个参数，可以传入一个函数指针或者仿函数 d(ptr)，ptr为 shared_ptr 保存的对象指针。

``` c++
void f(int * x);
shared_ptr<int> x(new int, f);

class Stock{/*...*/};
class StockFactory{
	void deleteStock(Stock *);
	/*...*/
};

shared_ptr<Stock> ptr;
ptr.reset(new Stock(key),bind(&StockFactory::deleteStock,this,_1));

```

## 1.10 enable_shared_from_this

继承 enable_shared_from_this class，可以使该 class 使用 shared_ptr 管理 this 指针。

```c++
class Foo : public boost:enable_shared_from_this<Foo>{/*..*/}
```

另外要注意的是，为了使用 shared_from_this(), 对象不能是 stack object ，必须是 heap object 且由 shared_ptr 管理生命周期。

```c++
ptr.reset(new Stock(key),bind(&StockFactory::deleteStock,shared_from_this(),_1));
```

## 1.11 弱回调

使用 enable_shared_from_this 方法传递对象的 shared_ptr 有一个缺陷，虽然这个方法是安全的，但这同时延长了对象的生命周期。有时我们需要“如果对象还活着就调用它的成员函数，否则忽略”这样的语境，我称之为“弱回调”。这是就可以使用，weak_ptr 这样对象的生命周期就不会延长，如果 weak_ptr 能提升成 shared_ptr 那就调用，如果不能就忽略。

```c++
class Stock{/*...*/};
class StockFactory{
	static void weakDeleteCallback(const boost:weak_ptr<StockFactory>& ,Stock*);
	/*...*/
};
shared_ptr<Stock> pStock;
pStock.reset(new Stock(key),boost::bind(&StockFactory::weakDeleteCallback,boost::weak_ptr<StockFactory>(shared_from_this()) ,_1));

```

## 1.12 心得与小结

### 1.12.1 心得

虽然本章写的是任何安全的使用跨线程对象，但实际上尽量减少使用跨线程对象，不使用跨线程对象，自然不会遇到本章描述的各种险态。  
_**“用流水线，生产者消费者，任务队列这些有规律的机制，最低限度地共享数据。这是我所知的最好的多线程编程的建议了”**_

### 1.12.2 小结

- 原始指针暴露给多个线程会导致各种问题。
- 统一用 shared_ptr / weak_ptr 管理对象的生命周期，在多线程中尤为重要。
- shared_ptr 是值语义，当心意外延长对象的生命周期。
- weak_ptr 是 shared_ptr 的好搭档，可以用作弱回调、对象池等。
- 认真阅读 boost::shared_ptr 的文档，能学到很多东西。

# 2 线程同步精要

并发编程有两种基本模型

- message passing
- shared memory  
    在分布式系统中，只有 message passing 这一种实用模型，message passing 也更容易保证程序的正确性。

线程同步的四项原则，按重要性排列：

1. 首要原则是降低共享对象，一个对象能不暴露给其他线程就不暴露，实在要暴露，优先考虑 immutable 对象，实在不行才暴露可修改的对象，并且使用同步措施充分进行保护。
2. 其次是使用高级的并发编程构件，如 TaskQueue 、 Producer-Consumer Queue 、CountDownLatch 等等。
3. 最后不得已必须使用底层同步原语时，只用非递归的互斥器和条件变量，慎用读写锁，不要用信号量。
4. 除了使用 atomic 整数之外，不自己编写 lock-free 代码，也不要用“内核级”同步原语。

## 2.1 互斥器（mutex）

互斥器恐怕是使用得最多的同步原语。粗略的说，任何时间只能有一个线程在互斥器划分出的临界区中活动。  
使用互斥器的原则：

- 使用 RAII 手法封装的 mutex 的创建、销毁、加锁、解锁这四个操作。
- 只用非递归的 mutex （即不可重入的 mutex）。
- 不手工调用 lock() 和 unlock() 函数，保证一切交给栈上的 Guard 对象的构造和析构函数负责。
- 在每一次构造 Guard 对象的时候，思考一路上已经持有的锁，防止因加锁顺序导致的死锁。

次要原则有：

- 不使用跨进程的 mutex，进程间通信只使用 TCP sockets。
- 加锁、解锁在同一个线程（RAII自动保证）。
- 记得解锁（RAII自动保证）。
- 不重复解锁（RAII自动保证）。
- 必要的时候可以考虑使用 PTHREAD_MUTEX_ERRORCHECK 来排错。

### 2.1.1 只使用非递归的 mutex

mutex 分为：

- 递归（recursive）
- 非递归（non-recursive）

或者称为：

- 可重入（reentrant）
- 非可重入

它们唯一的区别就是：同一个线程可以重复对 recursive mutex 加锁，但是不能重复对 non-recursive mutex 加锁。  
多次对 non-recursive mutex 加锁会立刻导致死锁，而 recursive mutex 不会，毫无疑问 recursive mutex 使用起来更为方便，但正因为它的方便，recursive mutex 可能会隐藏代码中的一些问题。典型情况就是你以为拿到一个锁就可以修改对象了，没想到外层代码已经拿到了锁，正在修改同一个对象呢。  
使用 non-recursive mutex 的优越性在于，如果出现了这种情况，non-recursive mutex 会出现死锁比较便于 debug，如果使用 recursive mutex 则会正常执行。

### 2.1.1 死锁

一个经典的死锁模型：带锁的对象 A 有一个可以调用 B 的方法，带锁的对象 B 有一个可以调用 A 的方法，有两个线程 t1 、 t2 分别执行两个方法，线程 t1 执行 A 的方法，先将自己加锁，然后 t2 线程执行 B 的方法，也将自己加锁， t1 线程继续执行，调用 B 等待 B 解锁，t2 线程也继续执行调用 A，也在等待 A 解锁，两个线程互相等待对方，死锁形成。  
在有两个对象互相调用的情况下要考虑这种死锁。

## 2.2 条件变量（condition variable）

在使用 mutex 的时候我们一般会希望加锁不要阻塞，总是能立刻拿到锁，然后尽快访问数据，用完后尽快解锁。  
如果我们需要等待某个条件成立，我们应该使用**条件变量（condition variable）** ，条件变量顾名思义有一个或者多个线程等待某个布尔值为真，等待别的线程“唤醒”它。条件变量学名**管程（monitor）**。

互斥器和条件变量构成了多线程编程的全部必备同步原语，用它们可以完成任何多线程同步任务，二者不能互相替代。

### 2.2.1 倒计时 （CountDownLatch）

倒计时是一种常用且易用的同步手段，它主要有两个用途：

1. 主线程发起多个子线程，等待多个线程都完成一定任务后，主线程才继续执行。
2. 主线程发起多个子线程，子线程等待主线程执行一些任务后，通知所有子线程开始执行。  
    当然也可以使用条件变量来实现这两种同步，不过用倒计时的话，逻辑更清晰。

## 2.3 不要使用读写锁和信号量

### 2.3.1 读写锁

- 从正确性来讲，容易犯的错误是在持有 read lock 的时候修改了共享数据，这种情况通常发生在程序的维护阶段，把 read lock 的程序调用了会修改数据的函数。
- 从性能三来讲，读写锁不一定比 mutex 要高效，如果临界区小，锁竞争不激烈，那么 mutex 往往更快。
- 通常 reader lock 是可重入的，writer lock 是不可重入的，为了防止 writer 饥饿 writer lock 通常会阻塞后来的 reader lock ，因此 reader lock 重入的时候可能死锁。
- 在最求低延迟读取的场合也不适用读写锁。

### 2.3.2 信号量

条件变量配合互斥器可以完全替代其功能，而且更不易用错

## 2.4 线程安全的 Singleton 实现

Singleton （单例模式）顾名思义，Singleton 保证了程序中同一时刻最多存在该类的一个对象。  
Singleton 一般有两种实现方式：

- eager initialization （饿汉）
    - 定义：
        - eager initialization 顾名思义就是在进入程序时直接实例化。
    - 优点：
        - 不用考虑多线程安全，因为即使是多线程程序，在进入的时候一般都是单线程。
        - 因为预先创建好了，所以调用时反应速度快。
    - 缺点：
        - 资源效率，所有实例在程序开始时创建，可能会造成卡顿。
- lazy initialization （懒汉）
    - 定义：
        - lazy initialization 顾名思义，等到要用的时候再实例化。
    - 优点：
        - 资源利用率高，要用的时候再实例化，很好的节省了资源。
    - 缺点：
        - 在多线程的情况下容易产生线程安全问题。
        - 第一次加载时不够快。

这里主要讲讲 lazy initialization 的线程安全实现。  
以下是一个 Singleton 的实现。

```c++
template<typename T>
class Singleton {
public:
	static T &instance() {
		if (!instance_) {
			instance_ = new T();
		}
		return *instance_;
	}
private:
	Singleton()=default;
	Singleton(const Singleton&) = delete;
	Singleton &operator=(const Singleton&) = delete;
	~Singleton() = default;
	static T * instance_;
};

template<typename T> 
T* Singleton<T>::instance_ = nullptr;
```

这个 Singleton 实现了它的基本功能，在第一次调用 instance 的时候构造对象，并且只构造一次，但很容易看出它是线程不安全的，如果有多个线程同时调用 instance 的时候，有可能会构造多个对象，造成内存泄漏。当然这个代码还有一个问题，就是 new 的对象没释放，也会造成内存泄漏，但由于这时一个 Singleton 就是一个长期存在直到系统关闭才销毁的对象，所以没用必要，当然解决这个问题也简单，可以使用 shared_ptr 或者 静态的嵌套类对象。

```c++
//将nstance修改为这样
static T &instance() {
	if (!instance_) {
		Sleep(1000);	//为了稳定出现内存泄漏
		instance_ = new T();
	}
	return *instance_;
}
// 运行两个线程，在vs的
void f() {
	Singleton<bitset<10000000>>::instance();
}
void vf() {
}
void test0() {
	thread t1(f);	//调用instance
	thread t2(vf);	//空进程，用于控制变量
	t1.join();
	t2.join();
	return 0;
}

void test1() {
	thread t1(f);	//调用instance
	thread t2(f);	//同时调用instance
	t1.join();
	t2.join();
	return 0;
}
```

运行 test0 的代码，增加的内存大概是 2 MB 运行 test1 的代码内存增加大概是 4 MB 很明显出现了内存泄漏，instance 申请了两次内存。  
为了处理线程安全的问题，很容易想出加锁解决。比如：

```c++
template<typename T>
class Singleton {
public:
	static T &instance() {
		lock_guard<mutex> lock(mutex_);
		if (!instance_) {
			instance_ = new T();
		}
		return *instance_;
	}
private:
	Singleton()=default;
	Singleton(const Singleton&) = delete;
	Singleton &operator=(const Singleton&) = delete;
	~Singleton() = default;
	static mutex mutex_;
	static T * instance_;
};

template<typename T> 
T* Singleton<T>::instance_ = nullptr;
template<typename T>
mutex Singleton<T>::mutex_;

```

### 2.4.1 DCL（Double Checked Locking）

但加锁的开销不小，每一次获取数据都要加一次锁属实是不明智，因为这个函数实际上只有第一次访问才会造成 race condition ，自然有人就想到了这种优化方法：

```c++
static T &instance() {
	
	
	if (!instance_) {
		lock_guard<mutex> lock(mutex_);
		instance_ = new T();
	}
	
	return *instance_;
}
```

但很明显这种优化方法是错误的，一样会造成 race condition ，这时我们可以采用 DCL（Double Checked Locking），顾名思义两次检查。

```c++
static T &instance() {
	if (!instance_) {
		lock_guard<mutex> lock(mutex_);
		if (!instance_) {
			instance_ = new T();
		}
	}
	return *instance_;
}
```

这种方式看似高枕无忧了，实际上很长时间，人们也认为这种方式是正确的。但是后来有人指出由于**乱序执行**（包括**编译乱序**和**执行乱序**，一个是编译器层面的一个是核心层面）的影响，DCL 也是靠不住的。

```c++
instance_ = new T(); 	//这个操作实际上是3个步骤
//如下
tmp= new (sizeof(T));	//申请内存
new(tmp) T();			//构造
instance_ =tmp;			//赋值
```

是结合之前的指令重排可以知道，编译器并不会被约束去执行这些步骤，很多时候第二步和第三步会交换，也就是先给 instance_ 赋值然后再构造。这时候如果还没有进行构造时线程被挂起，另一个线程访问单例就会认为 instance_ 已经构造完毕进而使用了未构造的对象，我们的程序就会 crash 。那么怎么写一个线程安全的双重检查？这需要用到**内存屏障（memory barriers）**或者**原子操作**。  
在 C++11 中这两种方式均被封装进标准库中可以直接调用，但先不讨论这两种方法，实际上如果使用 C++11 我们可以用一种更优美的方式实现 Singleton 。

### 2.4.2 Meyers' Singleton

```c++
template<typename T>
class Singleton {
public:
	static T &instance() {
		static T instance_;
		return *instance_;
	}
private:
	Singleton()=default;
	Singleton(const Singleton&) = delete;
	Singleton &operator=(const Singleton&) = delete;
	~Singleton() = default;
};
```

该 Singleton 使用一个静态变量来储存数据，非常的简单明了，在 C++ 11 中，它是线程安全的。根据标准，§6.7 [stmt.dcl] p4：

> If control enters  
> the declaration concurrently while the variable is being initialized, the concurrent execution shall wait for completion of the initialization.

gcc 和 vs 对该特性(动态初始化和并发销毁，也称为 msdn 上的 magic statics)的支持如下：

- Visual Studio：自 Visual Studio 2015 以来支持 。
- GCC：自 GCC 4.3 起支持。】

### 2.4.3 (linux)pthread_once / std::call_once 实现

```c++
template<typename T>
class Singleton {
public:
	static T &instance() {
		call_once(onceFlag_, [&]{instance_ = new T(); });
		return *instance_;
	}
private:
	Singleton()=default;
	Singleton(const Singleton&) = delete;
	Singleton &operator=(const Singleton&) = delete;
	~Singleton() = default;
	static once_flag onceFlag_;
	static atomic<T *> instance_;
};

template<typename T> 
atomic<T *> Singleton<T>::instance_ = nullptr;
template<typename T>
once_flag Singleton<T>::onceFlag_;
```

```c++
template<typename T>
class Singleton : boost::noncopyable {
public:
    static T& instance() {
        pthread_once(&ponce_, &Singleton::init);
        return *value_;
    }
    
    static void init() {
        value_ = new T();
    }
private:
    static pthread_once_t ponce_;
    static T* value_;
};
 
template<typename T>
pthread_once_t Singleton<T>::ponce_ = PTHREAD_ONCE_INIT;
 
template<typename T>
T* Singleton<T>::value_ = nullptr;

```

(linux)pthread_once / std::call_once 函数的特点是只会运行一次，并且线程安全，pthread_once 是 linux 自己线程库的东西，std::call_once 是 C++11 线程库的东西。

## 2.5 sleep 不是同步原语

sleep 只能出现在测试代码之中，正常的执行中，如果需要等待一段已知的时间，应该往 event loop 里注册一个 timer，然后在 timer 的回调函数中继续干活，如果是等待某个事情发生应该使用条件变量或者 IO 事件回调，不能使用 sleep。使用 sleep 低效且浪费资源。

## 2.6 借 shared_ptr 实现 copy-on-write

### 2.6.1 用普通 mutex 替换读写锁

读写冲突的对象，我们可以使用 shared_ptr 来实现类似读写锁的机制，将数据使用 shared_ptr 储存，但读取的时候使用一个栈上的局部 shared_ptr 变量当作“观察者”，将它指向数据，使数据的计数器增加，这时只需要加锁构建“观察者”的那部分,缩小了临界区，并且读操作不会互相冲突，增加 read 的速度，减少 read 的延时。

```c++
void read(){
	shared_ptr<DataType> obs;
	{
		lock_gruad<mutex> lock(mutex_);
		obs=data_;
 	}
	obs->doit();
}
```

write 在写入的时候判断一次数据是否正在被读取，也就是 shared_pte 的计数器是否为一，如果不为一就拷贝一份并且替代原本的数据，再进行修改。这个方法的缺点是 write 需要额外开销，如果进行频繁的写操作时内存中可能会出现多个数据的副本（由于是使用 shared_ptr 不会内存泄漏）,适合多读少写的需求情况。

```c++

void write(const WriteType &x){	
	lock_gruad<mutex> lock(mutex_);
	if(!data_.unique()){
		data_.reset(new DataType(*data_));
	}
	data_.write(x);
}

```

## 2.7 归纳总结

- 线程同步的四项原则，尽量使用高层同步设施（线程池、队列、倒计时）。
- 使用普通互斥器和条件变量完成剩余的同步内容，采用 RAII 惯用手法和 Scoped Locking。
- 读写冲突的时候可以复制并替换原本内容来解决。

# 3 多线程服务器的适用场合与常用编程模型

## 3.1 进程与线程

### 3.1.1 进程

粗略的说，一个进程是“内存中正在运行的程序”，每一个进程都有自己的独立地址空间。《Erlang程序设计》把“进程”比喻为“人”，为我们提供了个思考框架。  
每一个人有自己的记忆（内存），人与人之间通过谈话（消息传递）来交流，谈话可以是面对面（同一台机器中），也可以是通过电话（网络），面谈可以知道他是不是死了，而电话只能根据周期性的心跳来判断他是否活着。  
有了这个比喻，设计分布式系统时就可以采用“角色扮演”形式思考了：

- 容错      万一人死了
- 扩容      新人招募
- 负载均衡    把甲的活分担给别人
- 退休      甲要去培训 or 治病（修bug），先别派任务，等他做完手上的事情就重启

### 3.1.2 线程

线程的特点是可以共享地址空间，可以更高效的共享数据。  
多线程的价值是为了更好的发挥多核心处理器的效能，在单核时代，多线程没有多大价值。  
Alan Cox 说过：

> A Computer is a state machine. Threads are for people who can't program state machines.  
> (计算机是一个状态机，线程是给那些不能编写状态机程序的人准备的)

如果只有一块 CPU、一个执行单元，那么的确和 Alan Cox 所说的，按状态机的思路写程序是最高效的。

## 3.2 单线程服务器的常用编程模型

在高性能网络程序中，使用的最广泛的是“non-blocking IO + IO multiplexing”（非阻塞式 IO + IO 多路复用）这种模型，即 **Reactor** 模式，如：

- lighttpd，单线程服务器。
- libevent，libev。
- ACE，Poco C++ libraries。
- Java NIO。
- POE（perl）。
- Twisted（python）

相反 Boost.Asio 和 Windows I/O Completion Ports 实现了 **Proactor** 模式。应用面似乎要窄一些。

### 3.2.1 **Reactor**

- 实现：
    - Reactor 模型中。程序的基本结构就是一个事件循环（event loop），以事件驱动和事件回调的方式实现的业务逻辑。
- 优点：
    - 编程不难。
    - 效率不错。
    - 对 IO 密集形的应用是不错的选择。
- 缺点：
    - 要求事件回调函数必须是非阻塞式。
    - 容易割裂业务逻辑，相对不容易理解和维护。

## 3.3 多线程服务器常用编程模型

常见的模型大概有：

- 每一个请求创建一个线程，使用阻塞式 IO 操作。
- 使用线程池，同样使用阻塞式 IO 操作。
- 使用 non-blocking IO + IO multiplexing。
- Leader / Follower 等高级模式。

默认情况推荐使用第三种，即 non-blocking IO + one loop per thread 模式。

### 3.3.1 one loop per thread

在这种模型下，每一个 IO 线程有一个 event loop（Reactor），用于处理读写的定时事件。  
这种方法的好处是：

- 线程数目固定，不会频繁创建和销毁。
- 可以方便的在线程间调配负载。
- IO 事件发生的线程是固定的，同一个 TCP 连接不比考虑事件并发。

Event loop 代表了线程的主循环，需要让哪一个线程干活就把 timer 或 IO channel 注册到哪一个线程的 loop 里即可。

### 3.3.2 线程池

对于光有计算任务没有 IO 任务的线程，使用 event loop 有点浪费，使用一种补充方案，即用 blocking queue 实现的任务队列。

```c++
template<typename T>
class BlockingQueue {
public:
	BlockingQueue() = default;
	BlockingQueue(const BlockingQueue&) = delete;
	BlockingQueue& operator=(const BlockingQueue&) = delete;
	void push(const T& val) {
		lock_guard<mutex> lock(mutex_);
		data_.push(val);
		cond_.notify_one();
	}
	T pop() {
		unique_lock<mutex> lock(mutex_);
		while (data_.empty()) {
			cond_.wait(lock);
		}
		T tmp = data_.front();
		data_.pop();
		return tmp;
	}
	
private:
	queue<T> data_;
	mutex mutex_;
	condition_variable cond_;
};




class TharedPool {
public:
	using Functor = function<void()>;
	TharedPool(): running_(true), taskQueue_(){
		for (int i = 0; i < num_of_computing_thread; ++i) {
			threads_.push_back(thread(&workerThread,this));
		}
	}
	void post(Functor functor) {
		taskQueue_.push(functor);
	}
	
	void stop() {
		running_ = false;
		for (int i = 0; i < threads_.size(); ++i) {
			post([] {});
		}
		for (int i = 0; i < threads_.size(); ++i) {
			threads_[i].join();
		}
	}

private:
	static void workerThread(TharedPool* tp) {
		while (tp->running_) {
			Functor task = tp->taskQueue_.pop();
			task();
		}
	}

	BlockingQueue<Functor> taskQueue_;
	vector<thread> threads_;
	bool running_;
};

```

手动实现的阻塞队列来实现的简易线程池，要使用的时候使用前文的 Singleton 包装。  
除了任务队列，还可以使用 blocking queue 来实现生产者消费者队列，当然里面存储的就不是可调用对象了，而是数据。

### 3.3.3 推荐模式

总结，推荐使用 one（event） loop per thread + thread pool。

- event loop 用作 IO multiplexing ，配合 non-blocking IO 和定时器。
- thread pool 用于计算，具体可以是任务队列和生产者消费者队列。

## 3.4 进程间通信只用 TCP

Linux 下进程间通信（IPC）的方式数不胜数：

- 匿名管道（pipe）
- 具名管道（FIFO）
- POSIX 消息队列
- 共享内存
- 信号 （signals）
- Sockets

同步原语也有很多

- 互斥器
- 条件变量
- 读写锁
- 文件锁
- 信号量

贵精不贵多，进程间通信首选 Sockets ，好处在于：

- 可以跨主机，有伸缩性。
- 双向通信，方便。
- TCP port 由操作系统自动回收，即使程序意外退出也不会产生垃圾。
- TCP port 由程序独占，防止程序重复启动。
- 两个 TCP 通信，如果一个崩溃了，可以通过心跳，快速感应到另一个程序崩溃。
- 可记录，可重现。
- 跨语言。
- 实现简单。

## 3.5 多线程服务器的适用场合

### 3.5.1 处理并发连接

开发服务端程序的一个基本任务是处理并发连接，主要有两种方式：

- 当“线程”廉价时，一台机器可以创建远高与 CPU 数目的“线程”。这时一个线程只处理一个 TCP 连接，通常使用阻塞 IO。
- 但“线程”很宝贵的时候，通常创建和 CPU 数目相当的线程。这时一个线程要处理多个 TCP 连接上的 IO，通常使用非阻塞 IO 和 IO multiplexing 。

### 3.5.1 处理模式

在一个多核机器上提供一种服务或者执行一个任务，可用的模式有：

1. 运行一个单线程的进程
    - 这种模式是不可伸缩的，不能发挥多核心的优势
2. 运行一个多线程的进程
    - 这种模式被很多人鄙视，认为多线程难写，并且比起模式 3 没什么优势。
3. 运行多个单线程的进程
    - 3a 简单的把模式 1 中的进程运行多份。
    - 3b 主进程 + worker 进程。
4. 运行多个多线程的进程
    - 更是被千夫所指，不但没有结合 2 和 3 的优点，反而汇集了二者的缺点。

### 3.5.2 实例

比方说：使用速率为 50 MB / s 的数据压缩库、在进程创建和销毁的开销是 800 us、线程创建和销毁的开销是 50 us 的前提下，如何执行压缩任务：

- 如果要偶尔压缩 1 GB 的文本文件，预计的运行时间是 20 s ，那么起一个进程去做是合理的，因为启动进程的开销远远小于实际任务开销。
- 如果要经常压缩 500 KB 的文本文件，预计的运行时间是 10 ms ，那么每次起一个进程似乎有点浪费，可以单独起一个线程去做。
- 如果要频繁压缩 10 KB 的文本文件，预计的运行时间是 200 us ，那么每次起一个线程也很浪费，不如交给现在的线程搞定，或者用线程池，避免阻塞当前线程。

### 3.5.3 必须用单线程的场合

有两种场合必须使用单线程：

1. 程序可能会 fork(2)。
    - 这么做会出现很多麻烦，而且没有一定要这么做的理由。
2. 限制 CPU 占用率。
    - 多核心机器中，单线程程序最多只占用一个核心。

### 3.5.4 单线程程序的优缺点

- 优点：
    - 简单，程序的结构一般是一个基于 IO multiplexing 的 event loop。
    - 单核心下的性能优势。
- 缺点：
    - event loop 是非抢占的，容易造成优先级反转，没办法控制优先级。
    - 多核心下 CPU 利用率低。

### 3.5.5 适用多线程程序的场景

多线程的应用场景是：提高响应速度，让 IO 和计算互相重叠。  
一个程序要做成多线程大概要满足：

1. 有多个 CPU 可用。
2. 线程中有共享数据，如果没有共享数据，那使用多进程单线程模型就好。
3. 共享的数据可以修改，而不是静态的常量表。如果不能修改，那我们可以直接使用共享内存。
4. 提供非均质的服务，即，事务间有优先级。
5. 延迟和吞吐量同样重要。
6. 利用异步操作。
7. 可扩展。
8. 具有可预测的性能。
9. 多线程能有效的划分责任与功能，让每一个线程的逻辑比较简单，任务单一，便于编码。

线程的分类：

1. IO线程，这里线程的主循环是 IO multiplexing，也可以做一些简单的计算，比如消息的编码或者解码。
2. 计算线程，这类线程的主循环是 blocking queue，一般要避免任何阻塞操作。
3. 第三方库使用的线程。

## 3.6 答疑

### 3.6.1 Linux 能同时启用多少线程？

32 位 Linux 300 左右是上限，64 位就更多了，实际上在一台机器中最多只用到几十个用户线程。

### 3.6.2 多线程如何增加效率的？

线程不能减少工作量，反而会增加工作量，它做到的是资源的统筹调配，从而使各个资源的利用率提升，来提高效率。

### 3.6.3 什么是线程池大小的阻抗匹配原则？

如果线程池中线程在执行任务时间，密集计算所占时间比例为 P （0 << P ≤≤ 1）,而系统一共有 C 个 CPU ，为了让 CPU 跑满又不过载， 线程池的大小 T = C / P 。考虑到 P 不会很准确，T 的最佳值可上下浮动 50% ，当 P 小于 0.2 时，公式不适用，取一个固定的倍数会更好，比如 T = 5 * C 。

# 4 C++ 多线程系统编程精要

学习多线程最大的思维方式转变有两点：

1. 当前线程随时会被切换出去。
2. 多线程程序中事件发生的顺序不再有全局统一的先后关系。

## 4.1 基本线程原语的选用

11 个最基本的 Pthreads 函数是：

- 2 个：线程的创建和等待结束。封装为 muduo::Thread。
- 4 个：mutex 的创建、销毁、加锁、解锁。封装为 muduo::MutexLock。
- 5 个：条件变量的创建、销毁、等待、通知、广播。封装为 muduo::Condition。

除此之外还有一些其他原语，可以酌情使用：

- pthread_once，封装为 muduo::Singlenton<T>。其实不如直接使用全局变量。
- pthread_key*,封装为 muduo::ThreadLocal<T>。可以考虑用 __thread 替换。

不推荐使用的：

- pthread_rwlock。
- sem_*。
- pthread_{cancel,kill}。

## 4.2 C/C++ 系统库的线程安全性

- 一个线程安全的基本原则：如果一个对象自始至终只被一个线程用到，那么它就是安全的。
- 另一个事实标准是：共享对象的 read-only 操作是安全的，前提是不能有并发写的操作。即多个线程读取一个常量对象是安全的。一旦有了写操作，那么读操作也不安全了。
- 绝大多的泛型算法都是安全的，因为这些都是无状态函数。
- C++ 的 iostream 不是线程安全的，printf 是线程安全的，但相当于使用了全局锁，恐怕不太高效。

## 4.3 Linux 的线程标识

在 Linux 上建议使用 gettid(2) 系统调用的返回值作为线程 id，这么做的好处是：

- 它的类型是 pid_t，通常是一个小整数。
- 它直接表示内核的任务调度 id。
- 在其他系统工具中也容易定位到具体某一个线程。
- 任何时刻都是全局唯一的、
- 0 是非法值。

muduo::CurrentThread::tid() 使用了 __thread 变量缓存 gettid(2) 的返回值，更加高效。

## 4.4 线程的创建与销毁的守则

### 4.4.1 线程的创建

几条简单的原则：

- 程序库不应该在未提前告知的情况下创建自己的“背景线程”。
- 尽量使用相同的方法创建线程。
- 在进入 main() 之前不要启动线程。
- 程序中线程的创建最好能在初始化阶段全部完成。

### 4.4.2 线程的销毁

线程的销毁一般有几种方式：

- 自然死亡。线程从主函数返回，正常退出。
- 非正常死亡。从线程主函数抛出异常或线程触发 segfault 信号等非法操作。
- 自杀。在线程中调用函数退出线程。
- 他杀。其他线程调用函数强制终止某个线程。

线程正常退出的方式只有一种，即自然死亡。任何从外部强行终止线程的做法和想法都是错误的，不应该使用 pthread_cancel 或者 exit(3)。  
如果能做到程序中线程的创建在初始化阶段全部完成，则线程是不必销毁的，伴随进程一直执行，彻底避开了线程销毁的一系列问题。

## 4.5 善用 __thread 关键字

- 是什么？
    - __thread 是 GCC 内置的线程局部存储设施。
- 怎么用？
    - 使用 __thread 可以修饰 POD 类型的变量，不能修饰 class 类型，因为无法自动调用构造和析构。
    - __thread 只能修饰全局和函数内静态变量。
- 有什么用？
    - __thread 变量是每一个线程都有一份独立的实体，各线程不会互相干扰。
    - 还可以修饰那些“值可能会变，带有全局性，但又不值得用锁保护”的变量。

## 4.6 多线程与 IO

多线程共用一个 IO 没意义，难以安全实现，并且不会增加效率，每一个文件操作符只由一个线程操作。  
有两个例外：

1. 对于磁盘文件，在有必要的时候多个线程可以同时调用 pread(2) / pwrite(2) 来读写一个文件。
2. 对于 UDP，在适当条件下，可以多线程同时读写一个 UDP 文件描述符。

## 4.7 用 RAII 包装文件描述符

程序不要只记住文件描述符，要使用 RAII 的方式包装文件描述符，并且使用 shared_ptr ，来确保对象的生命周期。  
linux 的文件描述符在程序刚启动的时候，0 是标准输入，1 是标准输出，2 是标准错误，如果你新打开一个文件，那么它的文件描述符会是 3，如果你关闭这个文件，然后再打开它，还是 3。因为，POSIX 标准中规定，每一次打开文件的时候要使用当前最小可用文件描述符。

## 4.8 RAII 与 fork()

某些资源在使用 fork() 后不会复制，这会导致包装那种资源的对象，析构两次。在程序设计开始的时候就要考虑是否能用 fork() 了，不要在没有做好使用 fork() 准备的程序中使用 fork()。

## 4.9 多线程与 fork()

别在多线程程序使用 fork()，唯一安全的做法是：使用 fork() 后直接调用 exec() 来执行另一个程序，彻底隔断与父进程的联系。