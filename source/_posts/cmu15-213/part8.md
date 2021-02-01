---
title: CMU 15-213 Part8 Exceptional Control Flow
date: "2020/2/10"
categories:
- CMU 15-213
---

### 异常控制流

处理器读取并执行内存中的连续指令，称为**控制流(Control Flow)**。默认情况下，控制流一条接着一条指令执行。跳转和函数调用是程序改变控制流的方式，而系统改变控制流的方式则叫作**异常控制流(Exceptional Control Flow)**，在磁盘/网络数据到达，除0，系统计时器过期，键盘按下Ctrl+C等情况引发。异常控制流的应用包括**异常(Exception)**，**进程上下文切换(Process Contex Switch)**，**信号(Signals)**(纯软件实现)和**非本地跳转(Nonlocal Jumps)**(纯软件实现)。

### 异常

**异常**发生时，控制流从用户程序转移到内核，响应系统事件。注意内核不是单独存在的一个进程，而是进程的一部分，常驻在内存空间的顶部，用于管理系统资源。异常由硬件和操作系统协作实现，一般异常由硬件引发，但由操作系统提供软件来处理。内存的某处存储了一个异常跳转表，记录异常编号到处理函数的映射。

* **中断(Interrupt)**：来自处理器外部的异常称作**异步异常(Asynchronous Exception)**，也称作中断。引发中断的事件一般由处理器引脚电平的变化导致，比如数据到达或键盘按下Ctrl+C。系统无法预测中断何时到来，所以是异步的。

* **陷入(Traps)**：程序执行**系统调用(syscall)**的方式。内核的代码与资源是用户级程序无法访问的。为了请求系统资源，用户级程序主动发起一个异常，使控制流切换到内核状态，调用系统资源。系统库提供的函数，如open，read等，实际上是系统调用的封装。

* **故障(Faults)**：用户级程序执行某条指令时发生错误，比如除0，缺页，段错误(Segmentation Fault)等引发异常，有些是可恢复的，比如缺页，恢复异常时，这条指令重新执行。
；有些则不可，如段错误，会导致程序中止(Abort)。 

下图是通过陷入进行open系统调用的例子。把系统调用的编号写入%rax，`syscall`指令引发陷入，返回时文件描述符放在%rax中。

![open系统调用](https://i.loli.net/2020/02/08/V5XgKGTFRQ6OLiP.png)

下图是缺页引发可恢复故障的例子。程序访问地址0x8049d10，但是该地址并不可用，因为该地址没有从磁盘交换到内存(关于交换，参考虚拟内存)。引发缺页故障后，异常处理程序尝试交换。成功后，回到该指令重新执行，这次就不会引发故障了。

![缺页](https://i.loli.net/2020/02/08/igl4hcIG3xm56VC.png)

下图是访问无效内存地址引发不可恢复故障的例子。程序访问地址0x804e360，由于该地址不可用，同样引发了缺页故障。但是异常处理程序的交换尝试失败，于是向程序发送SIGSEGV中止。

![无效地址访问](https://i.loli.net/2020/02/08/gEHuakZBdjS1V5M.png)


### 进程上下文切换

进程是程序可执行文件的实例化，因为可执行文件只存储一份在磁盘中，而可执行文件可以被加载到内存的不同位置产生多个进程。操作系统为进程提供了两个抽象：**逻辑控制流(Logical Control Flow，即貌似独占的CPU资源)**和**私有的地址空间(虚拟内存)**。

每个进程拥有一套相互独立且私有的内存空间，寄存器值的上下文环境。上下文切换指的是处理器在这些环境间切换。由于上下文切换，多个进程可以在处理器上**并发(Concurrent)**运行。我们将并发定义为逻辑控制流的重叠，与核心数无关。即使在单核处理器上，进程也可以并发运行。下面第二张图，进程A与进程B是并发的，进程A与进程C是并发的，但进程B与进程C不是并发的。

![上下文环境](https://i.loli.net/2020/02/08/rtYCQ7ENJvBFnVk.png)

![并发的逻辑控制流](https://i.loli.net/2020/02/08/n4ZBOxcTRzv9mPJ.png)

上下文切换通过硬件和操作系统软件协作完成。硬件定时器过期时向处理器发送中断，为了接收定时器中断，操作系统将进程切换到内核态。此时，操作系统可以决定是否切换上下文。而进程之间的执行顺序是不可预测的，将多个进程的各条语句进行拓扑排序，每种排序结果都是一种执行顺序的可能性。

![上下文切换](https://i.loli.net/2020/02/08/bxUdZfSDFI4twAy.png)

### 进程编程实践

Linux系统用于进程操作的函数如下：

```cpp
pid_t fork(void);
void exit(int status);
pid_t wait(int *status);
pid_t waitpid(pid_t pid, int *status, int options);e
int execve(const char *filename, char *const argv[], char *const envp[]);
```

子进程执行完后变成**僵尸进程(Zombie)**，需要父进程回收资源。如果父进程比子进程更早退出，子进程成为init进程的子进程，被自动回收。如果父进程一直不退出，僵尸子进程在内存中越积越多，将导致内存泄漏。 


### 信号

信号是一种系统发送给进程的消息，告知进程系统中发生了某种事件。信号的编号(一个整数)决定它的种类，由内核发送给用户程序，在用户态执行信号处理代码。信号处理有捕获，忽略和中止三种方式。如果用户程序没有捕获并定义信号处理函数，则使用预定义的默认处理方式。信号从发送到接收分成了以下几个阶段：

1. 内核传递信号

2. 检查进程是否阻塞信号

3. 进程状态标记某些位，表示信号待处理

4. 上下文切换到进程接收信号，调用信号处理函数

5. 清除进程状态中标记的位

由于待处理信号只通过进程状态中的一个位表示，所以信号是不会排队的，即新的同种信号在原有信号被处理之前来到，该种信号也只被处理一次。

被阻塞的信号将暂时无法在进程状态中标记，标记进程状态的操作被延迟到阻塞取消后进行。某种信号的处理函数执行期间，该种信号被阻塞，但其他信号不受影响，称作隐式的信号阻塞。所以，在执行某种信号处理函数时，可能跳转到另一种信号的处理函数，但不可能再一次进入同种的信号处理函数。

![嵌套的逻辑控制流](https://i.loli.net/2020/02/08/m8cWuX6aszPUMv9.png)

可以通过如下几种方式发送信号：

* `kill -<signum> <pid>`，当pid为负数，表示gpid

* 键盘按下Ctrl+C给前台进程组发送SIGINT，Ctrl+Z发送SIGSTP

* `kill`函数

进程主函数的逻辑控制流与信号处理函数也是并发的。但是与多进程并发不同，进程的主函数与信号处理函数是共用内存空间的。所以信号处理函数也存在并发编程中死锁，同步一致性等问题。为了防止这些问题发生，可以遵守以下规则写安全的信号处理函数：

* 使信号处理函数尽量简单，比如设置一个全局标记后马上返回

* 在信号处理函数中只调用**可重入函数(Reentrant Function)**。异步安全函数指(没有使用全局/函数静态变量，所用变量存储在栈上)，或无法被信号中断的函数。异步安全的函数包括`_exit`，`write`，`wait`，`waitpid`，`sleep`，`kill`等；而常用的异步不安全函数有`printf`，`sprintf`，`exit`，`malloc`等。信号处理函数中不应该使用异步不安全函数；

* 在进入和退出信号安全函数时分别保存和恢复全局变量errno

* 访问共享数据前阻塞信号实现保护读/写

* 共享的全局变量声明为`volatile`，声明后变量将不会缓存到寄存器中而导致不一致问题。典型应用是在信号处理函数设置全局标记，主函数中while循环忙等该标记，则该标记要声明为`volatile`。如果该标记缓存到寄存器中，则主函数永远不能停止循环。

* 共享的全局标记声明为`volatile sig_atomic_t`，大多数系统的`sig_atomic_t`类型大小等同`int`，但是读写操作是不可中断的。

### 信号编程实践

因为信号不会排队的，不能使用信号来统计事件发生的次数。 接受到信号，说明事件**至少**发生了一次，比如接收到SIGCHLD信号说明**至少**一个子进程需要回收。printf不是异步安全的，因为printf在向 终端输出前需要获取终端的锁。假设main函数正在进行大量printf输出，这时接受到一个信号，处理函数中也调用了printf，而main函数的printf正在锁内，所以处理函数将永远阻塞等待，造成死锁。所以在信号处理函数中输出内容只能使用`write`函数。

```cpp
int ccount = 0;
void child_handler(int sig) {
    int olderrno = errno;
    pid_t pid;
    if ((pid = wait(NULL)) < 0)
        Sio_error("wait error");
    ccount--;
    Sio_puts("Handler reaped child ");
    Sio_putl((long)pid);
    Sio_puts(" \n");
    sleep(1);
    errno = olderrno;
}

void main() {
    pid_t pid[N];
    int i;
    ccount = N;
    Signal(SIGCHLD, child_handler);

    for (i = 0; i < N; i++) {
        if ((pid[i] = Fork()) == 0) {
            Sleep(1);
            exit(0);  /* Child exits */
        }
    }
    while (ccount > 0) /* Parent spins */
        ;
    return 0;
}
```

修改信号处理函数的实现，每次回收多个子进程，并且使用安全的`write`函数输出内容：

```cpp
void child_handler2(int sig)
{
    int olderrno = errno;
    pid_t pid;
    while ((pid = wait(NULL)) > 0) {
        ccount--;
        Sio_puts("Handler reaped child ");
        Sio_putl((long)pid);
        Sio_puts(" \n");
    }
    if (errno != ECHILD)
        Sio_error("wait error");
    errno = olderrno;
}
```

`read`函数属于**慢系统调用(slow syscall)`，进程调用read等待磁盘资源时，系统会调度执行其他进程，资源到达时才重新切换到该进程。但是信号也会导致系统切换回原来的进程，这时会中止系统调用，`read`函数会返回一个错误(EINTR)。使用`sigaction`函数注册信号时设置`SA_RESTART`标记可以自动重新开始系统调用，解决这个问题。`sigaction`函数也可以解决各个系统实现的注册信号行为不一致的问题。

```cpp
handler_t *Signal(int signum, handler_t *handler)
{
    struct sigaction action, old_action;

    action.sa_handler = handler;
    sigemptyset(&action.sa_mask); /* Block sigs of type being handled */
    action.sa_flags = SA_RESTART; /* Restart syscalls if possible */

    if (sigaction(signum, &action, &old_action) < 0)
        unix_error("Signal error");
    return (old_action.sa_handler);
}
```

另一个信号导致的异步问题是**竞争(Race)**。比如下面的代码，信号可能在`addjob`函数调用前发送，导致`addjob`函数添加无效的工作进程。

```cpp
void handler(int sig)
{
    int olderrno = errno;
    sigset_t mask_all, prev_all;
    pid_t pid;

    Sigfillset(&mask_all);
    while ((pid = waitpid(-1, NULL, 0)) > 0) { /* Reap child */
        Sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
        deletejob(pid); /* Delete the child from the job list */
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    if (errno != ECHILD)
        Sio_error("waitpid error");
    errno = olderrno;
}

int main(int argc, char **argv)
{
    int pid;
    sigset_t mask_all, prev_all;

    Sigfillset(&mask_all);
    Signal(SIGCHLD, handler);
    initjobs(); /* Initialize the job list */

    while (1) {
        if ((pid = Fork()) == 0) { /* Child */
            Execve("/bin/date", argv, NULL);
        }
        Sigprocmask(SIG_BLOCK, &mask_all, &prev_all); /* Parent */
        addjob(pid);  /* Add the child to the job list */
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    exit(0);
}
```

通过阻塞实现同步来解决这个问题。

```cpp
int main(int argc, char **argv)
{
    int pid;
    sigset_t mask_all, mask_one, prev_one;

    Sigfillset(&mask_all);
    Sigemptyset(&mask_one);
    Sigaddset(&mask_one, SIGCHLD);
    Signal(SIGCHLD, handler);
    initjobs(); /* Initialize the job list */

    while (1) {
        Sigprocmask(SIG_BLOCK, &mask_one, &prev_one); /* Block SIGCHLD */
        if ((pid = Fork()) == 0) { /* Child process */
            Sigprocmask(SIG_SETMASK, &prev_one, NULL); /* Unblock SIGCHLD */
            Execve("/bin/date", argv, NULL);
        }
        Sigprocmask(SIG_BLOCK, &mask_all, NULL); /* Parent process */
	addjob(pid);  /* Add the child to the job list */
        Sigprocmask(SIG_SETMASK, &prev_one, NULL);  /* Unblock SIGCHLD */
    }
    exit(0);
}
```

另一种竞争发生的情景是主函数忙等由信号处理函数设置的全局标记。在循环体中使用`pause`函数则会发生竞争。

```cpp
volatile sig_atomic_t pid;

void sigchld_handler(int s)
{
    int olderrno = errno;
    pid = Waitpid(-1, NULL, 0); /* Main is waiting for nonzero pid */
    errno = olderrno;
}

void sigint_handler(int s) {}

int main(int argc, char **argv) {
    sigset_t mask, prev;
    Signal(SIGCHLD, sigchld_handler);
    Signal(SIGINT, sigint_handler);
    Sigemptyset(&mask);
    Sigaddset(&mask, SIGCHLD);

    while (1) {
	Sigprocmask(SIG_BLOCK, &mask, &prev); /* Block SIGCHLD */
	if (Fork() == 0) /* Child */
            exit(0);
	/* Parent */
	pid = 0;
	Sigprocmask(SIG_SETMASK, &prev, NULL); /* Unblock SIGCHLD */

	/* Wait for SIGCHLD to be received (wasteful!) */
	while (!pid)
        pause();
	/* Do some work after receiving SIGCHLD */
        printf(".");
    }
    exit(0);
}
```

解决方法是使用`sigsuspend`函数暂时阻塞其他信号，等待特定信号发生，`sigsuspend`函数等效为下列代码的原子版本：

```cpp
sigprocmask(SIG_BLOCK, &mask, &prev);
pause();
sigprocmask(SIG_SETMASK, &prev, NULL);
```

利用`sigsuspend`函数解决上面例子出现的问题：

```cpp
int main(int argc, char **argv) {
    sigset_t mask, prev;
    Signal(SIGCHLD, sigchld_handler);
    Signal(SIGINT, sigint_handler);
    Sigemptyset(&mask);
    Sigaddset(&mask, SIGCHLD);

    while (1) {
        Sigprocmask(SIG_BLOCK, &mask, &prev); /* Block SIGCHLD */
        if (Fork() == 0) /* Child */
            exit(0);
 
       /* Wait for SIGCHLD to be received */
        pid = 0;
        while (!pid)
            Sigsuspend(&prev);
 
       /* Optionally unblock SIGCHLD */
        Sigprocmask(SIG_SETMASK, &prev, NULL);
	/* Do some work after receiving SIGCHLD */
        printf(".");
    }
    exit(0);
}
```
