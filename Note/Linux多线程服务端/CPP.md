### const 成员函数
---
- 特点
1. **不可修改类成员变量**：  
    使用`const`成员函数声明的返回类型和参数列表，保证了该函数在调用时不会修改类的成员变量。这意味着在`const`成员函数内，成员变量被视为只读的。
    
2. **可以被`const`对象调用**：  
    只有`const`成员函数能够被`const`对象调用。如果一个对象被声明为`const`，则只能调用`const`成员函数，而不能调用非`const`成员函数。
    
3. **加`const`关键字声明**： 
	`const`成员函数在函数声明和函数定义中都需要加上`const`关键字。
4. 加`mutable`修饰：
    在`const`成员函数中修改某些成员变量，此时可以使用`mutable`关键字修饰这些成员变量。`mutable`允许变量在`const`成员函数中被修改。
> sss
> ss





### 禁用拷贝构造
---
``` c++
# 禁用拷贝构造和赋值 
 MyClass(const MyClass&) = delete; // 删除拷贝构造函数
 MyClass& operator=(const MyClass&) = delete; // 删除拷贝赋值运算符

```

shard_ptr  在最外层拥有一个即可， 之后都使用const reference的方式使用，就不会发生拷贝，消耗资源


## BlockingQueue
---
`BlockingQueue` 是一个多线程编程中常用的数据结构，用于在生产者线程和消费者线程之间安全地传递数据。它结合了队列（Queue）和线程同步机制，提供了以下主要特性：

1. **队列特性**：
    
    - `BlockingQueue` 具备队列的基本特性，支持在队尾插入元素，队头删除元素的操作。通常实现为 FIFO（先进先出）结构。
2. **阻塞操作**：
    
    - 名称中的 "Blocking" 指的是队列在特定条件下会阻塞（即线程暂停执行，直到条件满足）。
    - 当消费者线程试图从空队列中取出元素时，队列会阻塞直到有元素可用。
    - 当生产者线程试图向满队列中插入元素时，队列会阻塞直到有空间可用。
3. **线程安全**：
    
    - 多线程环境下，`BlockingQueue` 提供了线程安全的操作。这意味着可以安全地在多个线程之间进行数据交换，而无需额外的显式锁定（例如互斥锁）。
4. **用途**：
    
    - `BlockingQueue` 通常用于生产者-消费者模型中，其中生产者线程生成数据并将其放入队列，消费者线程从队列中取出数据进行处理。
    - 其他应用包括任务调度器、线程池中的任务队列管理等。
5. **实现方式**：
    
    - `BlockingQueue` 可以使用不同的底层数据结构实现，如数组（循环队列）、链表等。
    - 基于条件变量（Condition Variable）的实现常见，通过条件变量来实现线程的阻塞和唤醒操作。


## 可变参数模板（Variadic Templates）
---
可变参数模板是C++中的一种强大特性，允许模板接受任意数量的模板参数。它在处理不定数量和不同类型参数的场景下非常有用。

#### 特性

- **灵活性**: 可变参数模板允许定义接受任意数量和类型参数的模板函数或类，提高了代码的通用性和灵活性。
- **递归展开**: 可以使用折叠表达式（fold expression）或递归展开来处理参数包中的参数，执行各种操作。
- **标准库支持**: 标准库中的许多工具，如 `std::tuple` 和 `std::make_tuple`，利用可变参数模板来处理不定数量的元素。

#### 示例

``` cpp
#include <iostream>
#include <functional>
using namespace std;

template<typename T, typename ...Args>
void expand(const T &func, Args&&... args) {
	// 这里用到了完美转发
	int arr[] = { (func(std::forward<Args>(args)), 0)... };
	// initializer_list<int>{ (func(std::forward<Args>(args)), 0)... };
}

// @note 递归终止函数
void print() {
    cout << "empty" << endl;
}

// @note 展开函数
template<typename T, typename... Args>
void print(T head, Args... rest) {
    cout << "parameter " << head << endl;
    print(rest...);
}


int main() {
	// C++ 11
	expand([](int i)->void{cout << i << endl;}, 1, 2, 3);
	// C++ 14 泛型lamda
	expand([](auto i)->void{cout << i << endl;}, 1, 2.2, "hello");
	
	print(1,2,3,4);
	return 0;
}
```


## 同步原语
---
### 互斥锁`std::mutex`,`RAII` (`std::locak_guard`,`std::unique_lock`) 

`Mutex`（互斥锁）是一种同步原语，用于保护共享资源，确保在任何时候只有一个线程可以访问该资源。在多线程编程中，使用互斥锁可以避免竞态条件（race condition）。在 C++11 中，使用 `std::mutex` 类来创建互斥锁：

``` cpp
#include <mutex>

std::mutex mutex_;

void func() {
	{
	    std::lock_guard<std::mutex> lock(mutex_); // 自动锁定互斥锁 mtx
	    // 访问共享资源
	}// lock 在这个作用域结束时自动释放互斥锁 mutex_
	{
	    std::unique_lock<std::mutex> lc(mutex_); // 自动锁定互斥锁 mtx
	    // 访问共享资源
	    lc.unlock();// uniquelock可以手动释放锁
	    lc.lock();//手动上锁
	}//在这个作用域结束时自动释放互斥锁 mutex_

}
```


### 条件变量（Condition Variables）

条件变量是多线程编程中用于线程间同步的重要机制，允许线程在某些条件满足时继续执行，否则进入等待状态。

条件变量通常与互斥锁（`mutex`）结合使用，用于解决等待条件的问题。它的基本原理是：
- 一个或多个线程等待条件满足，调用 `wait()` 进入等待状态。
- 当其他线程满足了条件，调用 `notify_one()` 或 `notify_all()` 唤醒等待的线程。

条件变量用于在以下情况下有效：
- 当一个线程需要等待某个条件的发生，才能继续执行。
- 多个线程之间需要协调操作，确保操作的顺序和条件满足。


使用 `std::condition_variable` 和 `std::mutex` 类来创建条件变量和互斥锁：

``` cpp
#include <mutex>
#include <condition_variable>

std::mutex mutex_;
std::condition_variable cond_var;

{// condition是条件表达式
	std::unique_lock<std::mutex> lock(mutex_);
	cond_var.wait(lock, []{ return condition; });
	while (condition){
		cond_var.wait(lock);// 自动释放互斥锁,唤醒后再次竞争
	}
	//执行操作
}//释放互斥锁

{ 
	std::lock_guard<std::mutex> lock(mtx); condition = true; 
} // 释放互斥锁 cv.notify_one(); // 唤醒一个等待的线程

```

- 控制条件变量使用的是std::unique_lock而不是std::lock_guard。原因是在wait()函数之前，使用互斥锁保护了，wait()函数会先调用互斥锁的unlock()函数，然后再将自己睡眠，在被唤醒后，又会继续尝试竞争持有锁，保护后面的队列操作。而lock_guard没有lock和unlock接口，而unique_lock提供了。这就是必须使用unique_lock的原因。
- 被唤醒的线程先尝试竞争互斥锁，获得锁之后在判断条件表达式。
- 使用while（）判断条件表达式和执行wait()函数;避免`spurious wakup`(虚假唤醒)。


## xxx
---
### xxx


``` cpp



```


