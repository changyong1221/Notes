# C++ STL - ConcurrencySupportLibrary

## std::thread

Defined in header `<thread>`  

|               |      |               |
| ------------- | ---- | ------------- |
| class thread; |      | (since C++11) |
|               |      |               |

The class `thread` represents [a single thread of execution](https://en.wikipedia.org/wiki/Thread_(computing)). Threads allow multiple functions to execute concurrently.

Threads begin execution immediately upon construction of the associated thread object (pending any OS scheduling delays), starting at the top-level function provided as a [constructor argument](https://en.cppreference.com/w/cpp/thread/thread/thread). The return value of the top-level function is ignored and if it terminates by throwing an exception, [std::terminate](https://en.cppreference.com/w/cpp/error/terminate) is called. The top-level function may communicate its return value or an exception to the caller via [std::promise](https://en.cppreference.com/w/cpp/thread/promise) or by modifying shared variables (which may require synchronization, see [std::mutex](https://en.cppreference.com/w/cpp/thread/mutex) and [std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic))

`std::thread` objects may also be in the state that does not represent any thread (after default construction, move from, [detach](https://en.cppreference.com/w/cpp/thread/thread/detach), or [join](https://en.cppreference.com/w/cpp/thread/thread/join)), and a thread of execution may not be associated with any `thread` objects (after [detach](https://en.cppreference.com/w/cpp/thread/thread/detach)).

No two `std::thread` objects may represent the same thread of execution; `std::thread` is not [*CopyConstructible*](https://en.cppreference.com/w/cpp/named_req/CopyConstructible) or [*CopyAssignable*](https://en.cppreference.com/w/cpp/named_req/CopyAssignable), although it is [*MoveConstructible*](https://en.cppreference.com/w/cpp/named_req/MoveConstructible) and [*MoveAssignable*](https://en.cppreference.com/w/cpp/named_req/MoveAssignable).

### Member types

| Member type                              | Definition               |
| ---------------------------------------- | ------------------------ |
| `native_handle_type`(not always present) | *implementation-defined* |

### Member classes

| [id](https://en.cppreference.com/w/cpp/thread/thread/id) | represents the *id* of a thread (public member class) |
| -------------------------------------------------------- | ----------------------------------------------------- |
|                                                          |                                                       |

### Member functions

| [(constructor)](https://en.cppreference.com/w/cpp/thread/thread/thread) | constructs new thread object (public member function)        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [(destructor)](https://en.cppreference.com/w/cpp/thread/thread/~thread) | destructs the thread object, underlying thread must be joined or detached (public member function) |
| [operator=](https://en.cppreference.com/w/cpp/thread/thread/operator%3D) | moves the thread object (public member function)             |
| Observers                                                    |                                                              |
| [joinable](https://en.cppreference.com/w/cpp/thread/thread/joinable) | checks whether the thread is joinable, i.e. potentially running in parallel context (public member function) |
| [get_id](https://en.cppreference.com/w/cpp/thread/thread/get_id) | returns the *id* of the thread (public member function)      |
| [native_handle](https://en.cppreference.com/w/cpp/thread/thread/native_handle) | returns the underlying implementation-defined thread handle (public member function) |
| [hardware_concurrency](https://en.cppreference.com/w/cpp/thread/thread/hardware_concurrency)[static] | returns the number of concurrent threads supported by the implementation (public static member function) |
| Operations                                                   |                                                              |
| [join](https://en.cppreference.com/w/cpp/thread/thread/join) | waits for the thread to finish its execution (public member function) |
| [detach](https://en.cppreference.com/w/cpp/thread/thread/detach) | permits the thread to execute independently from the thread handle (public member function) |
| [swap](https://en.cppreference.com/w/cpp/thread/thread/swap) | swaps two thread objects (public member function)            |

### Non-member functions

| [std::swap(std::thread)](https://en.cppreference.com/w/cpp/thread/thread/swap2)(C++11) | specializes the [std::swap](https://en.cppreference.com/w/cpp/algorithm/swap) algorithm (function) |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
|                                                              |                                                              |

### See also

| [jthread](https://en.cppreference.com/w/cpp/thread/jthread)(C++20) | **std::thread** with support for auto-joining and cancellation (class) |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
|                                                              |                                                              |

### 注意事项

**join和detach的区别**

操作

- - join 等待线程完成其执行(公开成员函数) 主线程等待子线程的终止。即主线程的代码块中，使用join()方法时主线程需要等待（阻塞）子线程结束了才能继续执行join()之后的代码块。
  - detach容许线程从线程句柄独立开来执行(公开成员函数)



c++ 11 之后有了标准的线程库：std::thread。

之前一些编译器使用 C++11 的编译参数是 -std=c++11

```
g++ -std=c++11 test.cpp 
```



### std::thread 构造函数

| 默认构造函数           | thread() noexcept;                                           |
| :--------------------- | ------------------------------------------------------------ |
| 初始化构造函数         | template <class Fn, class... Args> explicit thread(Fn&& fn, Args&&... args); |
| 拷贝构造函数 [deleted] | thread(const thread&) = delete;                              |
| Move 构造函数          | thread(thread&& x) noexcept;                                 |

- 默认构造函数，创建一个空的 `std::thread` 执行对象。
- 初始化构造函数，创建一个 `std::thread` 对象，该 `std::thread` 对象可被 `joinable`，新产生的线程会调用 `fn` 函数，该函数的参数由 `args` 给出。
- 拷贝构造函数(被禁用)，意味着 `std::thread` 对象不可拷贝构造。
- Move 构造函数，move 构造函数(move 语义是 C++11 新出现的概念，详见附录)，调用成功之后 `x` 不代表任何 `std::thread` 执行对象。

> 注意：可被 `joinable` 的 `std::thread` 对象必须在他们销毁之前被主线程 `join` 或者将其设置为 `detached`.

**std::thread** 各种构造函数例子如下：

```cpp
#include <iostream>
#include <utility>
#include <thread>
#include <chrono>
#include <functional>
#include <atomic>

void f1(int n)
{
    for (int i = 0; i < 5; ++i) {
        std::cout << "Thread " << n << " executing\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}

void f2(int& n)
{
    for (int i = 0; i < 5; ++i) {
        std::cout << "Thread 2 executing\n";
        ++n;
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}

int main()
{
    int n = 0;
    std::thread t1; // t1 is not a thread
    std::thread t2(f1, n + 1); // pass by value
    std::thread t3(f2, std::ref(n)); // pass by reference
    std::thread t4(std::move(t3)); // t4 is now running f2(). t3 is no longer a thread
    t2.join();
    t4.join();
    std::cout << "Final value of n is " << n << '\n';
}
```

### std::thread 赋值操作

| Move 赋值操作          | thread& operator=(thread&& rhs) noexcept;  |
| :--------------------- | ------------------------------------------ |
| 拷贝赋值操作 [deleted] | thread& operator=(const thread&) = delete; |

- Move 赋值操作(1)，如果当前对象不可 `joinable`，需要传递一个右值引用(`rhs`)给 `move` 赋值操作；如果当前对象可被 `joinable`，则会调用 `terminate`() 报错。
- 拷贝赋值操作(2)，被禁用，因此 `std::thread` 对象不可拷贝赋值。

请看下面的例子：

```cpp
#include <stdio.h>
#include <stdlib.h>

#include <chrono>    // std::chrono::seconds
#include <iostream>  // std::cout
#include <thread>    // std::thread, std::this_thread::sleep_for

void thread_task(int n) {
    std::this_thread::sleep_for(std::chrono::seconds(n));
    std::cout << "hello thread "
        << std::this_thread::get_id()
        << " paused " << n << " seconds" << std::endl;
}

int main(int argc, const char *argv[])
{
    std::thread threads[5];
    std::cout << "Spawning 5 threads...\n";
    for (int i = 0; i < 5; i++) {
        threads[i] = std::thread(thread_task, i + 1);
    }
    std::cout << "Done spawning threads! Now wait for them to join\n";
    for (auto& t: threads) {
        t.join();
    }
    std::cout << "All threads joined.\n";

    return EXIT_SUCCESS;
}
```

### 其他成员函数

**get_id**: 获取线程 ID，返回一个类型为 std::thread::id 的对象。请看下面例子：

```cpp
#include <iostream>
#include <thread>
#include <chrono>

void foo()
{
  std::this_thread::sleep_for(std::chrono::seconds(1));
}

int main()
{
  std::thread t1(foo);
  std::thread::id t1_id = t1.get_id();

  std::thread t2(foo);
  std::thread::id t2_id = t2.get_id();

  std::cout << "t1's id: " << t1_id << '\n';
  std::cout << "t2's id: " << t2_id << '\n';

  t1.join();
  t2.join();
}
```

**joinable**: 检查线程是否可被 join。检查当前的线程对象是否表示了一个活动的执行线程，由默认构造函数创建的线程是不能被 join 的。另外，如果某个线程 已经执行完任务，但是没有被 join 的话，该线程依然会被认为是一个活动的执行线程，因此也是可以被 join 的。

```cpp
#include <iostream>
#include <thread>
#include <chrono>

void foo()
{
  std::this_thread::sleep_for(std::chrono::seconds(1));
}

int main()
{
  std::thread t;
  std::cout << "before starting, joinable: " << t.joinable() << '\n';

  t = std::thread(foo);
  std::cout << "after starting, joinable: " << t.joinable() << '\n';

  t.join();
}
join: Join 线程，调用该函数会阻塞当前线程，直到由 *this 所标示的线程执行完毕 join 才返回。

#include <iostream>
#include <thread>
#include <chrono>

void foo()
{
  // simulate expensive operation
  std::this_thread::sleep_for(std::chrono::seconds(1));
}

void bar()
{
  // simulate expensive operation
  std::this_thread::sleep_for(std::chrono::seconds(1));
}

int main()
{
  std::cout << "starting first helper...\n";
  std::thread helper1(foo);

  std::cout << "starting second helper...\n";
  std::thread helper2(bar);

  std::cout << "waiting for helpers to finish..." << std::endl;
  helper1.join();
  helper2.join();

  std::cout << "done!\n";
}
```

**detach**: Detach 线程。 将当前线程对象所代表的执行实例与该线程对象分离，使得线程的执行可以单独进行。一旦线程执行完毕，它所分配的资源将会被释放。

调用 detach 函数之后：

- `*this` 不再代表任何的线程执行实例。
- joinable() == false
- get_id() == std::thread::id()

另外，如果出错或者 joinable() == false，则会抛出 std::system_error。

```cpp
#include <iostream>
#include <chrono>
#include <thread>
 
void independentThread() 
{
    std::cout << "Starting concurrent thread.\n";
    std::this_thread::sleep_for(std::chrono::seconds(2));
    std::cout << "Exiting concurrent thread.\n";
}
 
void threadCaller() 
{
    std::cout << "Starting thread caller.\n";
    std::thread t(independentThread);
    t.detach();
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Exiting thread caller.\n";
}
 
int main() 
{
    threadCaller();
    std::this_thread::sleep_for(std::chrono::seconds(5));
}
```

**swap**: Swap 线程，交换两个线程对象所代表的底层句柄(underlying handles)。

```cpp
#include <iostream>
#include <thread>
#include <chrono>

void foo()
{
  std::this_thread::sleep_for(std::chrono::seconds(1));
}

void bar()
{
  std::this_thread::sleep_for(std::chrono::seconds(1));
}

int main()
{
  std::thread t1(foo);
  std::thread t2(bar);

  std::cout << "thread 1 id: " << t1.get_id() << std::endl;
  std::cout << "thread 2 id: " << t2.get_id() << std::endl;

  std::swap(t1, t2);

  std::cout << "after std::swap(t1, t2):" << std::endl;
  std::cout << "thread 1 id: " << t1.get_id() << std::endl;
  std::cout << "thread 2 id: " << t2.get_id() << std::endl;

  t1.swap(t2);

  std::cout << "after t1.swap(t2):" << std::endl;
  std::cout << "thread 1 id: " << t1.get_id() << std::endl;
  std::cout << "thread 2 id: " << t2.get_id() << std::endl;

  t1.join();
  t2.join();
}
```

执行结果如下：

```
thread 1 id: 1892
thread 2 id: 2584
after std::swap(t1, t2):
thread 1 id: 2584
thread 2 id: 1892
after t1.swap(t2):
thread 1 id: 1892
thread 2 id: 2584
```

**native_handle**: 返回 native handle（由于 std::thread 的实现和操作系统相关，因此该函数返回与 std::thread 具体实现相关的线程句柄，例如在符合 Posix 标准的平台下(如 Unix/Linux)是 Pthread 库）。

```cpp
#include <thread>
#include <iostream>
#include <chrono>
#include <cstring>
#include <pthread.h>

std::mutex iomutex;
void f(int num)
{
  std::this_thread::sleep_for(std::chrono::seconds(1));

 sched_param sch;
 int policy; 
 pthread_getschedparam(pthread_self(), &policy, &sch);
 std::lock_guard<std::mutex> lk(iomutex);
 std::cout << "Thread " << num << " is executing at priority "
           << sch.sched_priority << '\n';
}

int main()
{
  std::thread t1(f, 1), t2(f, 2);

  sched_param sch;
  int policy; 
  pthread_getschedparam(t1.native_handle(), &policy, &sch);
  sch.sched_priority = 20;
  if(pthread_setschedparam(t1.native_handle(), SCHED_FIFO, &sch)) {
      std::cout << "Failed to setschedparam: " << std::strerror(errno) << '\n';
  }

  t1.join();
  t2.join();
}
```

执行结果如下：

Thread 2 is executing at priority 0 Thread 1 is executing at priority 20

**hardware_concurrency [static]**: 检测硬件并发特性，返回当前平台的线程实现所支持的线程并发数目，但返回值仅仅只作为系统提示(hint)。

```cpp
#include <iostream>
#include <thread>

int main() {
  unsigned int n = std::thread::hardware_concurrency();
  std::cout << n << " concurrent threads are supported.\n";
}
```

### std::this_thread 命名空间中相关辅助函数介绍

**get_id**: 获取线程 ID。

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <mutex>

std::mutex g_display_mutex;

void foo()
{
  std::thread::id this_id = std::this_thread::get_id();

  g_display_mutex.lock();
  std::cout << "thread " << this_id << " sleeping...\n";
  g_display_mutex.unlock();

  std::this_thread::sleep_for(std::chrono::seconds(1));
}

int main()
{
  std::thread t1(foo);
  std::thread t2(foo);

  t1.join();
  t2.join();
}
```

**yield**: 当前线程放弃执行，操作系统调度另一线程继续执行。

```cpp
#include <iostream>
#include <chrono>
#include <thread>

// "busy sleep" while suggesting that other threads run 
// for a small amount of time
void little_sleep(std::chrono::microseconds us)
{
  auto start = std::chrono::high_resolution_clock::now();
  auto end = start + us;
  do {
      std::this_thread::yield();
  } while (std::chrono::high_resolution_clock::now() < end);
}

int main()
{
  auto start = std::chrono::high_resolution_clock::now();

  little_sleep(std::chrono::microseconds(100));

  auto elapsed = std::chrono::high_resolution_clock::now() - start;
  std::cout << "waited for "
            << std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count()
            << " microseconds\n";
}
```

**sleep_until**: 线程休眠至某个指定的时刻(time point)，该线程才被重新唤醒。

```cpp
template< class Clock, class Duration >
void sleep_until( const std::chrono::time_point<Clock,Duration>& sleep_time );
```

**sleep_for**: 线程休眠某个指定的时间片(time span)，该线程才被重新唤醒，不过由于线程调度等原因，实际休眠时间可能比 sleep_duration 所表示的时间片更长。

```cpp
#include <iostream>
#include <chrono>
#include <thread>

int main()
{
  std::cout << "Hello waiter" << std::endl;
  std::chrono::milliseconds dura( 2000 );
  std::this_thread::sleep_for( dura );
  std::cout << "Waited 2000 ms\n";
}
```

执行结果如下：

```
Hello waiter
Waited 2000 ms
```
