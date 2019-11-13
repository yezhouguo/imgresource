# 《软件需求工程》实验二实验报告

* 姓名：

* 班级：

* 学号：

* 邮箱：

* 报告阶段：

* 完成日期：

* 本次实验，我完成了部分内容

* markdown插入代码，换行写：

  "```python"

## 目录

[TOC]

## 实验目的

​        在很多场景下，几个进程需要共用某个全局共享的变量，或者几个进程可能要协作完成某些特定的工作，这就需要进程之间不能随意地各执行各的，而是需要协调和同步。本实验就是通过实现PV操作的系统调用，并自己编写一个函数测试这些系统调用，来学习进程同步的相关知识。



## 实验内容

### 1.1. 实现`SEM_INIT`、`SEM_POST`、`SEM_WAIT`、`SEM_DESTROY`系统调用

#### `SEM_INIT`

首先可以观察到 `sem_init` 系统调用已经实现。在 `syscall.c` 中是该系统调用的函数，其中第三个参数 `value `通过保存现场中的 `edx` 寄存器传送（通过观察 `syscall` 函数和参考别的中断处理函数得出）。而在 `irqHandle.c` 中的 `syscallSemInit` 函数是具体实现。其中首先在 `sem` 数组中查找还未使用的信号量，用于存放新申请的信号量。如果找不到，该部分代码会返回-1：

```c
pcb[current].regs.eax = -1;
```

假设找到了空闲信号量表项，就执行以下代码。代码含义写在如下注释中：

```c
sem[i].state = 1;//表示该表项已被占用来表示信号量
sem[i].value = (int32_t)sf->edx;//信号量初始值设置为用户要求的初始值
sem[i].pcb.next = &(sem[i].pcb);//循环链表头部节点初始化，next指向自身
sem[i].pcb.prev = &(sem[i].pcb);//循环链表头部节点初始化，prev指向自身
pcb[current].regs.eax = i;//init成功，返回给用户进程表项下标
```



#### `SEM_POST`

`sem_post` 系统调用相当于V操作。其流程是：将其参数信号量的 `value` 值增加1，如果 `value` 值大于0，则设置返回值后结束不做任何处理。

如果 `value` 值小于等于0，就说明**仍有进程阻塞在该信号量**上等待使用（一般有 `|value| + 1` 个进程阻塞）。因此应当唤醒其中的一个等待进程执行，具体操作就是将循环链表首部的 `pcb` 中表示进程状态的 `state` 从阻塞状态 `STATE_BLOCKED` 改变为 `STATE_RUNNABLE` ，并将 `sleepTime` 置为0（不必要，只求严谨），然后将该 `pcb` 出循环队列。

在本实验框架中，细节上，循环链表的各个节点是各个 `pcb` 的 `blocked` 成员，通过一些**地址偏移**的操作就可以找到 `blocked` 成员所在的 `pcb` 表项，偏移方式如下：

```c
ProcessTable * pcb_pointer = (ProcessTable *)((uint32_t)sem[tmp].pcb.next + sizeof(struct ListHead) - sizeof(ProcessTable));
```

此时 `pcb_pointer` 变量就指向节点所在 `pcb` 表项。

执行 `sem_post` 时，要求返回值在调用成功时返回0，反复思考认为成功的情况即该 `sem` 表项的 `state` 为1，即信号量正在使用，才认为调用成功并进行V操作。失败情况就是 `sem` 表项的 `state` 为0。返回值通过设置 `pcb[current].regs.eax` 为0或-1来实现。

循环链表从链表头部出队，将头结点和首部的下一个节点相接：

```c
sem[tmp].pcb.next->next->prev = &sem[tmp].pcb;//首个pcb节点下一个节点前驱指向头结点
sem[tmp].pcb.next = sem[tmp].pcb.next->next;//头节点后继指向首个pcb节点下一个节点
```

V操作中，若 `value` 值小于等于0，释放被阻塞进程后执行进程调度，`int $0x20` 即可。



#### `SEM_WAIT`

`sem_wait` 系统调用相当于P操作。其流程是：将参数信号量的 `value` 值减少1，如果 `value` 值大于等于0，则不做任何处理。

如果 `value` 值小于0，就说明该信号量被使用（一般有别的进程正在使用信号量并已经用完）。因此当前执行了该系统调用的进程就应该阻塞住，等待某一次信号量释放从而被唤醒重新执行。阻塞的过程是将该进程 `pcb` 的 `blocked` 成员插入循环链表尾部，将其 `pcb` 的 `state` 成员设置为 `STATE_BLOCED` 。此外，还需要设置 `sleepTime` 成员。因为实际上**因P操作阻塞的进程必须由V操作来唤醒而不能由时钟中断的调度唤醒**，否则会带来未知错误，比如导致进程不同步或互斥访问被破坏等问题。**因此在时钟中断时，P操作造成的阻塞态进程的sleepTime不应该减1**。猜想助教的框架代码已经考虑了这个问题，发现框架代码的 `timerHandle` 函数中，对 `sleepTime` 为-1的，不做减一处理：

```c
if (pcb[i].state == STATE_BLOCKED && pcb[i].sleepTime != -1)
	pcb[i].sleepTime --;
```

因此**应该将P操作造成阻塞的进程 `sleepTime` 设置为-1**。

接下来就是将本进程插入信号量对应的阻塞进程循环队列尾部。做如下操作即可：

```c
sem[tmp].pcb.prev->prev->next = &pcb[current].blocked;//原最后一项后继指向新加入节点
sem[tmp].pcb.prev = &pcb[current].blocked;//头节点前驱指向新加入节点
pcb[current].blocked.prev = tmpprev;//新加入节点前驱指向原最后一项
pcb[current].blocked.next = &sem[tmp].pcb;//新加入节点后继指向头结点
```

设置返回值的判断标准同 `sem_post` 。



#### `SEM_DESTROY`

`sem_destroy` 系统调用销毁信号量。在本框架代码中，需要释放信号量占用的 `sem` 表项。为了保证严谨性，还需将阻塞住的进程 `pcb` 循环链表清空。但在清理信号量的时候**如果还有进程在等待该信号量**，用户就会收到一个**错误**（返回-1且**不销毁**）。因此操作如下：

```c
sem[tmp].state = 0;//释放占用的sem表项
sem[tmp].pcb.prev = &sem[tmp].pcb;//头结点后继指向自身，用于清空阻塞进程循环链表
sem[tmp].pcb.next = &sem[tmp].pcb;//头结点前驱指向自身，用于清空阻塞进程循环链表
```

```c
if (sem[tmp].state == 0 || sem[tmp].pcb.next != &sem[tmp].pcb)//循环链表不空
    pcb[current].regs.eax = -1;//调用失败
```



### 1.2. 多进程进阶

需要修改框架代码而使得框架能够支持同时8个进程并发执行。观察到 `pcb` 共有 `MAX_PCB_NUM` 个表项，通过 `MAX_PCB_NUM` 的定义找到：

```c
#define MAX_PCB_NUM ((NR_SEGMENTS-2)/2)
```

`NR_SEGMENTS` 变量为：

```c
#define NR_SEGMENTS      10
```

需要支持8个进程，就需要将 `pcb` 表扩展为8项（原先为(10 - 2) / 2 等于4项）。将 `NR_SEGMENTS` 改为18即可。

```c
#define NR_SETMENTS 18
```

由于原来每个进程占用2 * 0x100000个字节内存空间，而系统最大寻址空间4GB，故不需要缩减每个进程的内存。观察 `gdt` 变量的初始化，发现其大小是 `NR_SEGMENTS`，故更改过 `NR_SEGMENTS` 后， `gdt` 的大小也随之变化。



### 1.3. 信号量进程同步进阶

#### `GETPID` 系统调用

要求返回调用进程的 `pid` 。系统调用的过程是根据中断号判断中断类型（时钟中断/系统调用），然后根据系统调用号进行处理。

首先在 `syscall.c` 中实现 `int getpid` ，并在 `lib.h` 中声明:

```c
#define SYS_PID 7
int getpid(void) {
	return syscall(SYS_PID, 0, 0, 0, 0, 0);
}
```

然后在 `irqHandle.c` 的系统调用选择中，增加一条 `switch` 语句的 `case`，进入中断处理函数。完成中断处理函数，函数的返回值放入 `eax` 寄存器：

```c
pcb[current].regs.eax = pcb[current].pid;
```



#### 生产者消费者实现

##### `main` 函数

首先通过1号进程 `fork` 出6个子进程，前两个为生产者，后四个为消费者。为使得 `fork` 不多余，要求只有1号进程可以 `fork`，即返回值大于0才能 `fork`。故 `fork` 过程如下：

```c
int count = 0;
while (ret > 0 && count < 6){
    ret = fork();
    ++count;
}
```

 `fork` 结束后，此时共有7个进程正在执行这段代码（`uEntry` 部分的代码）。应当让生产者执行循环的生产函数，而消费者执行消费者函数：

```c
if (pid == 2 || pid == 3) { //生产者
	/*执行生产函数*/
}
else if (pid >= 4 && pid <= 7) { //消费者
	/*执行消费函数*/
}
```

进程结束应该用 `exit` 系统调用而非 `return 0` ，否则会报错退出。



##### `生产者producer函数` 

题目要求一个生产者要生产八个产品，因此构造循环共8次的循环体。循环体中，

* 生产者应该先生产产品

* 试图将产品放入缓冲区

  ```c
  /*try lock*/
  P(mutex);
  ```

* 成功占用并放产品入缓冲区

* 增加产品数信号量

  ```c
  V(bufsem);//有可能会唤醒一个消费者
  ```

* 释放缓冲区访问信号量

  ```c
  V(mutex);//有可能会唤醒一个生产者/消费者进入缓冲区访问
  ```

  

##### `消费者consumer函数`

题目要求一个消费者要生产八个产品，因此构造循环共8次的循环体。循环体中，

* 消费者应该先"预订"产品（即不去缓冲区拿）

  （假设消费者不预订产品，直接进入缓冲区，然后要求取产品，发现没有产品，他就会**阻塞**住，但同时缓冲区**仍被他占用**，引发**死锁**）

  ```c
  P(bufsem);//预订产品，如果产品不足，会阻塞等待生产者的生产来唤醒
  ```

* 成功预订了产品，尝试进入缓冲区取产品

  ```c
  P(mutex);//如果有其他进程正在访问缓冲区，则他阻塞
  ```

* ### 进入缓冲区取产品

* 离开缓冲区，释放缓冲区访问互斥锁

  ```c
  V(mutex);
  ```



## 实验结果

### 原测试用例

![1558943006295](C:\Users\1091378351\AppData\Roaming\Typora\typora-user-images\1558943006295.png)

### 自己的测试用例

![1558706239094](C:\Users\1091378351\AppData\Roaming\Typora\typora-user-images\1558706239094.png)

经过分析，输出符合逻辑。



## 实验中的问题

* 在消费者函数中，先P缓冲区互斥锁，后P产品，导致程序卡住

  原因：假设无产品，会阻塞，同时不释放缓冲区。导致**死锁**。

* 事实上，本实验中的P操作和V操作**并非原语**，故有可能发生信号量使用上的错误。



## 一些细节

* 对消费者和生产者，每生产/消费一个产品，我都选择sleep(64)，以方便观察输出。

* 注意生产者是**先生产产品再进入缓冲区**而不是进入缓冲区才生产产品。



## 实验总结

本实验中，通过自己实现PV操作的系统调用，更直观深刻地理解和观察到了PV操作的内部机制，加深了记忆。
