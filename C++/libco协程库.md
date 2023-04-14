# libco协程库

[源码地址](https://github.com/Tencent/libco)

## libco库（非对称协程）

libco库是腾讯于2013年在GitHub上开源的C++协程库，也是微信底层所使用的协程库。

​	协程不过是一段子程序(其实也就是个函数)罢了，因此只要保存下当前的**函数栈状态、寄存器值**，就可以描述出这个协程的全部状态，这个结构被称为协程上下文。

​	**协程上下文其实就是保存了当前所有寄存器的值**，这些寄存器描述了当前的函数栈，程序执行状态等信息。



## 主要代码文件

### [coctx_swap.S](https://github.com/Tencent/libco/blob/master/coctx_swap.S)

​	汇编代码，用来完成协程切换过程的寄存器和函数栈的保存和载入。通过 coctx.h 中定义的coctx_t保存和读取寄存器和函数栈的信息

```cpp
struct coctx_t
{
	void *regs[ 14 ];       // 一个数组，保存了14个寄存器的值 
	size_t ss_size;         // 协程栈大小
	char *ss_sp;            // 协程栈指针
};
```

​	可以看到，在coctx_t这个结构体中。ss_size和ss_sp则描述了协程的栈空间大小。regs是一个数组，保存着14个寄存器当前的值。



##### 文献参考

https://zhuanlan.zhihu.com/p/463549090



### co_routine.cpp

负责协程创建、运行、切换的实现

#### 相关函数

1、 创建协程

```cpp
int co_create( stCoRoutine_t **ppco,const stCoRoutineAttr_t *attr,pfn_co_routine_t pfn,void *arg );
```



2、 启动协程（Resume a coroutine）

​	 co_resume 用于第一次启动协程或者是暂停之后恢复启动，可以多次调用

```cpp
void co_resume( stCoRoutine_t *co );
```



3、协程的挂起（Yield to another coroutine）

co_yield_env从协程调用链的栈中弹出协程控制块，然后进行协程上下文切换。

```cpp
void co_yield( stCoRoutine_t *co )
{
	co_yield_env( co->env );
}

void co_yield_ct()
{

	co_yield_env( co_get_curr_thread_env() );
}

void co_yield_env( stCoRoutineEnv_t *env )
{
	
	stCoRoutine_t *last = env->pCallStack[ env->iCallStackSize - 2 ];
	stCoRoutine_t *curr = env->pCallStack[ env->iCallStackSize - 1 ];

	env->iCallStackSize--;

	co_swap( curr, last);
}
```





#### 数据结构

`stCoRoutine_t`：libco 的协程控制块

```c++
struct stCoRoutine_t {
    stCoRoutineEnv_t *env;
    pfn_co_routine_t pfn;
    void *arg;
    coctx_t ctx;

    char cStart;
    char cEnd;
    char cIsMain;
    char cEnableSysHook;
    char cIsShareStack;

    void *pvEnv;

    //char sRunStack[ 1024 * 128 ];
    stStackMem_t* stack_mem;

    //save satck buffer while confilct on same stack_buffer;
    char* stack_sp; 
    unsigned int save_size;
    char* save_buffer;

    stCoSpec_t aSpec[1024];
};
```



`stCoRoutineEnv_t`：协程执行环境

```cpp
struct stCoRoutineEnv_t {
    stCoRoutine_t *pCallStack[128];
    int iCallStackSize;
    stCoEpoll_t *pEpoll;

    // for copy stack log lastco and nextco
    stCoRoutine_t* pending_co;
    stCoRoutine_t* ocupy_co;
};
```

​	这个 env，即同属于一个线程所有协程的执行环境，包括了当前运行协程、上次切换挂起的协程、嵌套调用的协程栈，和一个 epoll 的封装结构（TBD）。不同于 Go 语言，libco 的协程一旦创建之后便跟创建时的那个线程绑定了的，是不支持在不同线程间迁移（migrate）的。



`CallStack`：保存协程调用关系

​	变量CallStack（stCoRoutineEnv_t结构体中）模拟了保存协程调用链的栈。非对称协程机制下的**被调协程只能返回到调用者协程（类似于函数调用，只能返回上一级，而不是随意跳转）**，这种调用关系不能乱，因此必须将调用链保存下来，这即是 pCallStack 的作用。

​	每当启动（resume）一个协程时，就将它的协程控制块 `stCoRoutine_t` 结构指针保存在 pCallStack 的“栈顶”，然后“栈指针” `iCallStackSize` 加 1，最后切换 context 到待启动协程运行。当协程要让出（yield）CPU 时，就将它的 `stCoRoutine_t`从`pCallStack` 弹出，“栈指针” `iCallStackSize` 减 1，然后切换 context 到当前栈顶的协程（原来被挂起的调用者）恢复执行。这个“压栈”和“弹栈”的过程具体是在`co_resume()` 和 `co_yield()` 函数中完成。



`stCoEpoll_t `结构与定时器

​	作为 stCoRoutineEnv_t 的成员，这个结构也是一个全局性的资源，被同一个线程上所有协程共享。从命名也看得出来，stCoEpoll_t 是跟 epoll 的事件循环相关的。**和主协程中co_eventloop()循环相关**

```cpp
struct stCoEpoll_t {
    int iEpollFd;
    static const int _EPOLL_SIZE = 1024 * 10;
    struct stTimeout_t *pTimeout;
    struct stTimeoutItemLink_t *pstTimeoutList;
    struct stTimeoutItemLink_t *pstActiveList;
    co_epoll_res *result; 
};
```



#### 具体实现

##### 挂起协程与恢复的执行

先来看 yield，实际上在 libco 中共有 3 种调用 yield 的场景：

	1. 用户程序中主动调用 `co_yield_ct()`；
 	2. 程序调用了 `poll()` 或 `co_cond_timedwait()` 陷入“阻塞”等待；
 	3. 程序调用了 `connect()`, `read()`, `write()`, `recv()`, `send()` 等系统调用陷入“阻塞”等待。



相应地，重新 resume 启动一个协程也有3种情况：

	1. 对应用户程序主动 yield 的情况，这种情况也有赖于用户程序主动将协程 `co_resume() `起来；
 	2. `poll() `的目标文件描述符事件就绪或超时，`co_cond_timedwait() `等到了其他协程的 `co_cond_signal() `通知信号或等待超时；
 	3. `read()`, `write()` 等 I/O 接口成功读到或写入数据，或者读写超时。



​	如果调用 read 或 write 等系统调用陷入真正的阻塞（让当前线程被内核挂起）的话，那么不光当前协程被挂起了，其他协程也得不到执行的机会（**本质上协程调度就是在一个内核线程上的实现用户态的切换。如果线程由于阻塞导致不能占用cpu，那么内部的协程也无法执行**）。

 

###### `co_cond_timedwait`函数

​	co_cond_timedwait() 函数内部会将当前协程挂入条件变量的等待队列上，并设置一个回调函数，该回调函数是用于未来“唤醒”当前协程的（即 resume 挂起的协程）。

​	如果 wait 的 timeout 参数大于 0 的话，还要向当前执行环境的定时器上注册一个定时事件（即挂到时间轮上）。

```cpp
int co_cond_timedwait( stCoCond_t *link,int ms )
{
	stCoCondItem_t* psi = (stCoCondItem_t*)calloc(1, sizeof(stCoCondItem_t));
	psi->timeout.pArg = GetCurrThreadCo();	//保存当前协程控制块
	psi->timeout.pfnProcess = OnSignalProcessEvent;		//回调函数
	...
	AddTail( link, psi);	//添加到条件变量的等待队列上

	co_yield_ct();


	RemoveFromLink<stCoCondItem_t,stCoCond_t>( psi );
	free(psi);

	return 0;
}

//恢复ap中保存协程的调用
static void OnSignalProcessEvent( stTimeoutItem_t * ap )
{
	stCoRoutine_t *co = (stCoRoutine_t*)ap->pArg;
	co_resume( co );
}
```



###### `co_cond_signal`函数

​	co_cond_signal 函数内部其实也简单，就是将条件变量的等待队列里的协程拿出来，然后挂到当前执行环境的 pstActiveList，对应co_cond_timedwait中的操作。

```cpp
int co_cond_signal( stCoCond_t *si )
{
	stCoCondItem_t * sp = co_cond_pop( si );
	if( !sp ) 
	{
		return 0;
	}
	RemoveFromLink<stTimeoutItem_t,stTimeoutItemLink_t>( &sp->timeout );

	AddTail( co_get_curr_thread_env()->pEpoll->pstActiveList,&sp->timeout );

	return 0;
}
```



###### `poll `函数

​	poll 函数内部实际上做了两件事。首先，将自己作为一个定时事件注册到当前执行环境的定时器，注册的时候设置了 1 秒钟的超时时间和一个回调函数（仍是一个用于未来“唤醒”自己的回调）。然后，就调用 co_yield_env() 让出cpu。

poll中调用了co_poll_inner

```cpp
int co_poll_inner( stCoEpoll_t *ctx,struct pollfd fds[], nfds_t nfds, int timeout, poll_pfn_t pollfunc)
{
       stPoll_t& arg = *((stPoll_t*)malloc(sizeof(stPoll_t)));
	memset( &arg,0,sizeof(arg) );
	arg.pfnProcess = OnPollProcessEvent;
	arg.pArg = GetCurrCo( co_get_curr_thread_env() );
    ...
        
        //add timeout
    int ret = AddTimeout( ctx->pTimeout,&arg,now );
	int iRaiseCnt = 0;
	if( ret != 0 )
	{
		co_log_err("CO_ERR: AddTimeout ret %d now %lld timeout %d arg.ullExpireTime %lld",
				ret,now,timeout,arg.ullExpireTime);
		errno = EINVAL;
		iRaiseCnt = -1;

	}
    else
	{
		co_yield_env( co_get_curr_thread_env() );
		iRaiseCnt = arg.iRaiseCnt;
	}
}

//poll的回调函数
void OnPollProcessEvent( stTimeoutItem_t * ap )
{
	stCoRoutine_t *co = (stCoRoutine_t*)ap->pArg;
	co_resume( co );
}
```



##### 主协程

​	libco 的第一个协程，即执行 main 函数的协程，是一个特殊的协程。这个协程又可以称作主协程，它负责协调其他协程的调度执行（后文我们会看到，还有网络 I/O 以及定时事件的驱动），它自己则永远不会 yield，不会主动让出 CPU。不让出（yield）CPU，不等于说它一直霸占着 CPU。

​	主协程是跟` stCoRoutineEnv_t `一起创建的。主协程也无需调用 resume 来启动，它就是程序本身，就是 main 函数。主协程是一个特殊的存在，可以认为它只是一个结构体而已。

​	在程序首次调用 co_create() 时，此函数内部会判断当前进程（线程）的 `stCoRoutineEnv_t `结构是否已分配，如果未分配则分配一个，同时分配一个 stCoRoutine_t 结构，并将 pCallStack[0] 指向主协程。此后如果用 `co_resume() `启动协程，又会将 resume 的协程压入 pCallStack 栈。



###### co_evectloop

​	当所有协程都挂起了，cpu控制流就回到了主协程上了。**当控制流回到主协程上时，主协程在干些什么呢？**

​	main 函数中程序最终调用了 co_eventloop()。该函数是一个基于 epoll/kqueue 的事件循环，**负责调度其他协程运行**，stCoRoutineEnv_t 结构中的 pEpoll 即使在这里用的就够了。

​	它周而复始地 epoll_wait()，唤醒挂起的工作协程去处理定时器与 I/O 事件。这里的逻辑看起来跟所有基于 epoll 实现的事件驱动网络框架并没有什么特别之处，更没有涉及到任何协程调度算法，由此也可以看到 libco 其实是一个很典型的非对称协程机制。

执行流程：

第 6 行：调用 epoll_wait() 等待 I/O 就绪事件，为了配合时间轮工作，这里的 timeout 设置为 1 毫秒。

第 8~10 行：active 指针指向当前执行环境的 pstActiveList 队列，注意这里面可能已经有“活跃”的待处理事件。timeout 指针指向 pstTimeoutList 列表，其实这个 timeout 完全是个临时性的链表， pstTimeoutList 永远为空。

第 12～19 行：处理就绪的文件描述符。如果用户设置了预处理回调，则调用 pfnPrepare 做预处理（15行）；否则直接将就绪事件 item 加入 active 队列。实际上，pfnPrepare() 预处理函数内部也是会将就绪item加入 active 队列，**最终都是加入到 active 队列等到 32~40 行统一处理。**

第 21~22 行：从时间轮上取出已超时的事件，放到 timeout 队列。

第 24~28 行：遍历 timeout 队列，设置事件已超时标志（bTimeout 设为 true）。 

第30行：将 timeout 队列中事件合并到 active 队列。 

第32~40行：遍历 active 队列，**调用工作协程设置的 pfnProcess() 回调函数** resume 挂起的工作协程，处理对应的 I/O 或超时事件。

```C++
  1 void co_eventloop(stCoEpoll_t *ctx, pfn_co_eventloop_t pfn, void *arg)
  2 {
  3     co_epoll_res *result = ctx->result;
  4
  5     for (;;) {
  6         int ret= co_epoll_wait(ctx->iEpollFd, result, stCoEpoll_t::_EPOLL_SIZE, 1);
  7
  8         stTimeoutItemLink_t *active = (ctx->pstActiveList);
  9         stTimeoutItemLink_t *timeout = (ctx->pstTimeoutList);
 10         memset(timeout, 0, sizeof(stTimeoutItemLink_t));
 11
 12         for (int i=0; i<ret; i++) {
 19         }
 20
 21         unsigned long long now = GetTickMS();
 22         TakeAllTimeout(ctx->pTimeout, now, timeout);
 23
 24         stTimeoutItem_t *lp = timeout->head;
 25         while (lp) {
 26             lp->bTimeout = true;
 27             lp = lp->pNext;
 28         }
 29
 30         Join<stTimeoutItem_t, stTimeoutItemLink_t>(active, timeout);
 31
 32         lp = active->head;
 33         while (lp) {
 34             PopHead<stTimeoutItem_t, stTimeoutItemLink_t>(active);
 35             if (lp->pfnProcess) {
 36                 lp->pfnProcess(lp);
 37             }
 38             lp = active->head;
 39         }
 40     }
 41 }
```





## 实际使用

### [生产者消费者](https://github.com/Tencent/libco/blob/master/example_cond.cpp)

​	消费者、生产者工作协程调用 `co_cond_timedwait()` 或 `poll()` 陷入“阻塞”等待，本质上即是通过 `co_yield_env `函数让出了 CPU；而主协程则负责在事件循环中“唤醒”这些“阻塞”的协程，所谓“唤醒”操作即调用工作协程注册的回调函数，这些回调内部使用 `co_resume()` 重新恢复挂起的工作协程。



### 系统走读

https://zhuanlan.zhihu.com/p/51078499

https://zhuanlan.zhihu.com/p/51081816



## 总结

1. libco是一种非对称协程，类似函数调用，当协程通过yield让出cpu后，通过自定义的栈结构，cpu只能回到上一层（调用者的协程中）。而最顶层的协程是主协程，当cpu控制权回到主协程后：会循环 co_eventloop() 函数。**在 co_eventloop() 中，主协程周而复始地调用 epoll_wait()**，当有就绪的 I/O 事件就处理 I/O 事件，当定时器上有超时的事件就处理超时事件，pstActiveList 队列中已有活跃事件就处理活跃事件。这里所谓的“处理事件”，其实就是调用其他工作协程注册的各种回调函数而已。