# THREAD_LOCAL
当在多线程环境，我们希望在访问一个变量时能让线程各自持有一个该变量的副本，而 C++11 关键字 `thread_local` 貌似给我们提供了期望的功能，但就目前而言它的实现并不理想。

## thread_local 是如何工作的？
我们通过 pthread 来了解 thread_local 的行为
```C++
#include<pthread.h>
#include<stdio.h>

thread_local int a = 10;

void * task(voidp arg){
    int * p = & a;
    printf("%p %p %d\n", & arg, p, a);
    return nullptr;
}

int main(){
    pthread_t p0, p1;
    pthread_create(& p0, nullptr, task, nullptr);
    pthread_create(& p1, nullptr, task, nullptr);
    pthread_join(p0, nullptr);
    pthread_join(p1, nullptr);
    return 0;
}

// 输出：
// 00000000029cfe80 0000000000021f88 10
// 0000000002bcfe80 0000000000021f38 10
```

每个线程读到的 `a` 的值都是 `10`，实际上只是存放的值时一样的，但它们的地址是不同的。线程库会为每一个线程分配一个 `a`，
实际上 `a` 是一个类似句柄的东西，在访问该 `a` 中内容时，我们通过当前线程 id 和 `a` 句柄可以唯一确定一个属于当前线程的变量。  
但没有这么简单，调用层层嵌套，在 windowws 平台实际上还是调用 TlsGetValue/TlsSetValue，比较繁杂。

## 新方法
**表象**
```C++
thread_local int data0 = 0;
thread_local int data1 = 1;

voidp task(voidp){
    int d0  = data0;
    int d1  = data1;
    data0   = d0 + 1;
    data1   = d1 + 1;
    return nullptr;
}

```
  
**实质**
```C++
struct local_data{
    int data0 = 0;
    int data1 = 1;
};

voidp task(voidp){
    // 这里只是示意将线程私有数据放到栈顶
    // 我们在分配线程栈内存时可以多分配一点内存用于放这些数据
    // 我们也可以通过当前线程占用的 cpu 编号获取当前线程控制块(tcb)，不过这要求线程切换时
    // 内核需要同时切换 cpu 编号对应的 tcb 指针
    // 接着通过 tcb 获取栈顶私有数据起始地址，自然也就可以获取到我们想要的线程私有数据
    // 我们无需再去分配一块内存或者创建新的结构去存放这些数据
    // 因为它们本身是静态的，占用整个线程生命周期的
    local_data local;
    int d0      = local.data0;
    int d1      = local.data1;
    local.data0 = d0 + 1;
    local.data1 = d1 + 1;
    return nullptr;
}

```