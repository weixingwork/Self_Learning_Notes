🔹**进程 (Process)**

**Q:** 什么是进程 (process)？

**A:** 正在运行的程序，是操作系统进行**资源分配与调度**的基本单位；拥有**独立的地址空间**（代码段/数据段/堆/栈）、打开文件表、信号处理等。

  

**Q:** 什么是 PCB (Process Control Block)？

**A:** 内核用来记录进程状态的结构，含 PID、寄存器快照、调度信息、打开文件指针、退出码、父子关系等。父进程通过它知道子进程的**return code**。

  

**Q:** 为什么执行命令（如 ls）时 shell 要先 fork()？

**A:** 为了让 shell 自己继续存在与交互。若直接 exec ls，shell 会被**替换**，交互中断；fork() 先复制出一个**子进程**，再在子进程里 exec 新程序。

---

🔹**地址空间 & 内核切换 (trap / system call)**

**Q:** 什么是 system call？

**A:** 用户态程序通过**陷入指令（trap）**进入内核态以请求服务（如 read/write/open/fork/exec/wait/exit）。库函数会在底层发出 trap 指令。

  

**Q:** 调用 write(fd, buf, len) 时发生了什么？

**A:** 用户态→trap→内核态，内核根据 fd 找到内核“文件对象”，按权限和**文件偏移(cursor)** 把数据写到目标；若错误返回 -1 并设置 errno。

  

**Q:** 用户/内核地址空间如何共存？

**A:** 每个进程有“用户部分”，同时映射**同一套内核**；陷入内核后，仍是**同一条执行线**在该进程的**内核栈**上运行，再返回用户态继续。

---

🔹**创建/替换/终止 进程：fork/exec/exit/wait**

**Q:** fork() 的作用与返回值？

**A:** 创建一个子进程（几乎父进程的拷贝）。返回两次：

- 父进程得到**子进程 PID**（>0）；
    
- 子进程返回**0**。
```
pid_t pid = fork();
if (pid == 0) { /* child path */ }
else if (pid > 0) { /* parent path, pid是孩子PID */ }
else { /* -1: fork失败 */ }
```
**Q:** if ((pid = fork()) == 0) { ... } 这句怎么读？

**A:** 先调用 fork() 并**赋值**给 pid，随后判断是否等于 0。只有**子进程**会进入 if 分支；父进程跳过 if。

  

**Q:** exec() 干什么？与 fork() 如何配合？

**A:** exec* 家族用**新程序**替换当前进程的地址空间（PID 不变）。常用模式：父 fork() -> 子 exec()。
```
execl("/bin/ls", "ls", "-l", (char*)0);  // argv[0] 通常写程序名
```
**Q:** execl 参数为什么有两个 "primes"？

**A:** 第一个是**路径**，第二个是 argv[0]（程序名），后接各参数，最后以 NULL 结束。

  

**Q:** exit(n) 与从 main 返回的关系？

**A:** main 的 return n; 最终会调用 exit(n)。在多线程程序里，exit() 会**终止整个进程**（所有线程）。

  

**Q:** wait() / waitpid() 作用与要点？

**A:** **阻塞**等待子进程结束并**回收**其 PCB，得到退出状态：
```
int st; pid_t c = wait(&st);
if (WIFEXITED(st)) printf("code=%d\n", WEXITSTATUS(st));
```
- 有多个子进程时，wait() 回来的是**任意一个**已结束的子进程。
    
- 不 wait 会产生**僵尸进程**。
    

  

**Q:** 僵尸进程 (zombie) 与 reparenting？

**A:** 子进程结束而父进程未 wait → 子留在**僵尸**状态（仅保留 PID/退出码）等被回收。若父先死，孤儿会被 PID=1（init/systemd）接管并收割（**reparenting**）。

  

**Q:** PID 会复用吗？

**A:** 会，但 OS 会尽量**延迟**复用，避免与僵尸/未收割记录混淆。

---

🔹**文件 & I/O（fd/句柄/重定向/对象/表）**

**Q:** 什么是文件描述符 (fd) 与“句柄 (handle)”？

**A:** fd 是进程可见的**整型索引**，指向进程的**文件描述符表**中的一个“文件对象”；handle 泛指由内核管理对象的**索引**。

  

**Q:** 标准 fd 是哪些？

**A:** 0=stdin，1=stdout，2=stderr。

  

**Q:** “最小空闲 fd 分配”规则？

**A:** open() 总返回该进程中**最小编号**的空闲 fd。
```
close(1);                      // 释放stdout
int fd = open("out.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644);
// 若成功，fd 通常=1，实现了 stdout 重定向
```
**Q:** 如何在子进程中把 stdout 重定向到文件再 exec？

**A:**
```
if (fork() == 0) {
  close(1);
  if (open("/path/out", O_WRONLY|O_CREAT|O_TRUNC, 0644) == -1) _exit(1);
  execl("/bin/ls", "ls", (char*)0);
  _exit(1);
}
wait(0);
```
等价 shell：ls > /path/out。**打开的 fd 会在 exec 后继续存在**（称“扩展地址空间”随 exec 保留）。

  

**Q:** open/read/write/close 基本用法？

**A:**
```
int fd = open("file", O_RDWR);              // O_RDONLY/O_WRONLY/O_RDWR
char buf[1024]; ssize_t n = read(fd, buf, sizeof(buf));
if (write(fd, buf, n) == -1) perror("write");
close(fd);
```
perror() 会基于 errno 打印出错原因。

  

**Q:** FILE*（如 printf/fprintf）与 fd 的差别？

**A:** FILE* 是**C 标准库缓冲流**；fd 是内核级**非缓冲**接口。fprintf(stdout,...) 近似 write(1,...)，但前者有用户态缓冲。

  

**Q:** 文件对象里都存什么？

**A:** 访问模式（如 O_RDONLY）、文件偏移（cursor）、引用计数、指向 inode 的指针等。**上下文信息在内核，不可被用户程序随意篡改**。

---

🔹**多线程 (POSIX Threads)**

**Q:** 如何创建线程？函数原型要求是什么？

**A:**
```
int pthread_create(pthread_t *t, const pthread_attr_t *attr,
                   void *(*start)(void*), void *arg);
```
线程入口必须是 void* (*)(void*)：
```
void *worker(void *arg) { /* ... */ return NULL; }
pthread_t tid; pthread_create(&tid, NULL, worker, arg);
```
**Q:** 传参 (void*)i 与地址传参的坑？

**A:** (void*)i 把整数强转为指针，**64 位下有风险**。更安全：为每个线程**单独分配**实参并传地址：
```
int *pi = malloc(sizeof *pi); *pi = i;
pthread_create(&tid,0,worker,pi);  // worker里用完free(pi)
```
**Q:** 多参数怎么传？

**A:** 定义结构体：
```
typedef struct { int a, b; } two_ints_t;
two_ints_t *p = malloc(sizeof *p); *p=(two_ints_t){x,y};
pthread_create(&tid,0,worker,p);
void *worker(void *arg){ two_ints_t *p=arg; /*...*/ free(p); return NULL; }
```
**Q:** 传局部变量地址为何易出**内存破坏 (memory corruption)**？

**A:** 若创建线程后函数**提前返回**，该局部变量所在的**栈帧无效**，线程里仍在用**悬空指针** → 读写随机内存导致崩溃/数据错乱。解决：在返回前 pthread_join() 或使用 malloc。

  

**Q:** pthread_join() 与线程返回值？

**A:** 阻塞等待线程结束，并可取回其 void* 返回值：
```
void *ret; pthread_join(tid,&ret);
```
一个线程**只能被 join 一次**；没人 join 的结束线程会保留返回值/ID/栈信息，类似**僵尸线程**，直到被 join 或 detach。

  

**Q:** pthread_exit() vs return vs 进程 exit()？

**A:** pthread_exit(v) 和 return (void*)v; 都结束**当前线程**并把值交给 join。exit() 会结束**整个进程**（所有线程），常由 main 的 return 隐式触发。

  

**Q:** 主线程想退出但保留其他线程继续跑？

**A:** 在 main 里调用 pthread_exit(NULL);（而不是 return），这样进程不退出，其他线程可继续。

  

**Q:** 什么是分离线程 (detached thread)？

**A:** 调 pthread_detach(tid) 的线程**无需 join**，结束即自动回收资源。适合“fire-and-forget”。**已 detach 的线程不能再 join**。

  

**Q:** 线程栈多大？如何设置？

**A:** Linux/glibc 默认常见**8MB**。可用属性设置：
```
pthread_attr_t a; pthread_attr_init(&a);
pthread_attr_setstacksize(&a, 20*1024*1024);   // 例如20MB
pthread_create(&tid, &a, worker, arg);
pthread_attr_destroy(&a);
```
**线程多**→考虑减小栈；**深递归/大局部数组**→增大栈。

  

**Q:** 栈溢出 (stack overflow) 如何产生与避免？

**A:** 递归过深或大局部数组超栈限触发溢出。避免：控制递归深度；把大数组放**堆**；必要时**增大线程栈**。

  

**Q:** 为什么多线程可能更快？

**A:** 能把**计算/IO**并行在多个 CPU 核、或在一个线程等待 IO 时让别的线程**无感切换**继续算/做 IO。注意：需要**足够的可并行工作**与**合理同步**，否则锁竞争会抵消收益。

  

**Q:** 经典并行例：矩阵乘法按行并行？

**A:** 为 C 的每一行开一个线程：
```
void* matrow(void* arg){ int row=(int)(intptr_t)arg; /* 计算C[row][*] */ return 0; }
for(int i=0;i<M;++i) pthread_create(&th[i],0,matrow,(void*)(intptr_t)i);
for(int i=0;i<M;++i) pthread_join(th[i],0);
```
（注意安全地传参：(intptr_t) 或传地址）

---

🔹**并发错误与同步 (Race / Mutex / Critical Section)**

**Q:** 什么是 race condition（竞态条件）？

**A:** 最终结果取决于多个执行单元的**交错顺序**；例如两个线程同时做 x = x + 1 会丢失更新。

  

**Q:** 什么是 data race（数据竞争）？

**A:** 多线程同时访问同一内存地址，至少一个是**写**，且无同步；在 C/C++ 内存模型里属于**未定义行为**。

  

**Q:** 什么是临界区 (critical section)？

**A:** 访问**共享可变状态**的代码片段，必须保证**互斥**。

  

**Q:** 如何用 mutex 保护临界区？

**A:**
```
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&m);
x = x + 1;             // 临界区
pthread_mutex_unlock(&m);
```
保证同一时刻**只有一个线程**执行被保护代码，消除 data race。

---

🔹**I/O 重定向（进程内实现）**

**Q:** 用系统调用把程序输出写到文件（等价 > file）怎么写？

**A:**
```
if (fork()==0) {
  close(1);
  if (open("out", O_WRONLY|O_CREAT|O_TRUNC, 0644) == -1) _exit(1);
  execl("/home/bc/bin/primes","primes","300",(char*)0);
  _exit(1);
}
while (wait(0) > 0) {}   // 回收子进程
```
要点：close(1) 先腾出 1；open 新文件拿到最小可用 fd→通常是 1；exec 后写 1 就是写文件。

---

🔹**编译/链接 pthread 程序**

**Q:** gcc 如何编译 pthread 程序？

**A:** -pthread（优选）或 -lpthread：
```
gcc -o app app.c -pthread
```
-pthread 既启用编译期的线程宏，也在链接时带上 pthread 库。

🔹**更多术语小卡**

**Q:** 返回码/退出码 (return/exit code) 存在哪里？

**A:** 存在子进程的 PCB/TCB 中，父进程通过 wait()/pthread_join() 取走。

  

**Q:** errno 是什么？

**A:** 线程局部的错误码变量，系统调用失败时设置；perror() 据此打印可读信息。

  

**Q:** 句柄能否“跨 exec”保留？

**A:** 默认**保留**已打开 fd；除非设置 FD_CLOEXEC（fcntl），使其在 exec 时自动关闭。

  

**Q:** dup2(oldfd, newfd) 有啥用？

**A:** 把 newfd 指向 oldfd 的同一对象（必要时先关闭 newfd），是**重定向**的常用原语。

  

**Q:** wait() 是阻塞调用吗？

**A:** 是。没有已结束的子进程时会睡眠，直到某个子进程结束。

  

**Q:** 线程之间有父子关系吗？

**A:** **没有**。任何线程只要拿到 pthread_t 都能 join 另一个线程；但**同一线程不能被 join 两次**。

  

**Q:** 僵尸线程与僵尸进程的相似点？

**A:** 都是**已结束但未被收割**的执行实体；保留 ID/返回值等，直到被 join/wait 清理。

--- 

🔹**Mutex 初始化**

**Q:** 如何静态初始化一个 pthread 互斥锁？

**A:**
```
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
```
这是最常见的写法，编译时就完成了初始化。初始状态：**解锁 (unlocked)**。

🔹**动态初始化**

**Q:** 如果一个 mutex 不能用 PTHREAD_MUTEX_INITIALIZER 静态初始化，怎么办？

**A:** 使用 pthread_mutex_init()：
```
pthread_mutex_t m;
pthread_mutex_init(&m, NULL);
```
第二个参数可以传 pthread_mutexattr_t 属性（一般传 NULL，表示默认属性）。


🔹**销毁 mutex**

**Q:** 为什么要调用 pthread_mutex_destroy()？

**A:** 用完互斥锁后要释放其内部资源：
```
pthread_mutex_destroy(&m);
```
🔹**额外说明**

**Q:** pthread_mutexattr_t 有什么用？

**A:** 它允许配置一些特殊属性，比如**递归锁**（同一线程可以多次加锁）、**错误检测**模式等。大多数情况用不到，所以 slides 说 _“Usually, mutex attributes are not used”_。

🔹**静态初始化**

**Q:** 静态初始化 pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER; 有什么限制？

**A:**

- 只能用于 **静态存储期变量**（如全局变量、static 局部变量）。
    
- 编译时初始化，程序启动时就已经分配并初始化好了。
    
- 不适合在**运行时动态分配的场景**。


🔹**动态初始化**

**Q:** 为什么需要动态初始化 pthread_mutex_init()？

**A:**

1. **在堆上分配的 mutex**
    
    - 比如通过 malloc 动态创建的结构体里含有 pthread_mutex_t，就必须运行时调用 pthread_mutex_init()。
```
typedef struct {
    int data;
    pthread_mutex_t lock;
} shared_t;

shared_t *p = malloc(sizeof(shared_t));
pthread_mutex_init(&p->lock, NULL);
```
2. **需要自定义属性**
    
    - 比如要创建递归锁：
```
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);

pthread_mutex_t m;
pthread_mutex_init(&m, &attr);
pthread_mutexattr_destroy(&attr);
```
3. **运行时控制资源生命周期**
    
    - 你可能需要在某个时刻创建和销毁锁（例如线程池或服务器中，连接对象的创建/销毁过程）。


🔹**销毁的必要性**

**Q:** 为什么还要 pthread_mutex_destroy()？

**A:**

- 避免内核/库资源泄漏。
    
- 静态初始化的全局 mutex 在程序结束时会由操作系统回收，但**动态创建的对象**必须手动销毁。



🔹**互斥与原子性 (Mutual Exclusion & Atomicity)**

**Q:** 为什么 x = x+1 不是原子操作？

**A:** 因为机器实际执行为：
```
ld   r1, x    // load
add  r1, 1    // add
st   r1, x    // store
```
若线程在执行中途被切换，可能和另一个线程交错执行，最终结果错误（只加 1 而不是 2）。

  

**Q:** 什么是“原子性 (atomicity)”？

**A:** 一段操作要么**全部完成**，要么**完全不执行**，中间不能被打断。对 x=x+1 来说，三个指令必须作为一个整体执行。

  

**Q:** 原子性是否意味着 CPU 不会去干别的事？

**A:** 不是。CPU 可以切换去干别的，但只要这些操作**不涉及同一个变量 x**，就不影响原子性。

---
🔹**Critical Section 与 Mutex**

**Q:** 什么是临界区 (critical section)？

**A:** 在多线程中，访问共享变量的代码片段。必须通过互斥锁保护，保证同一时间只有一个线程执行。

  

**Q:** 为什么 slide 里用“安全存储箱 (safe-deposit box)”来比喻 mutex？

**A:** 共享变量像存放在保险箱里的东西，mutex 就像钥匙。线程必须先拿到钥匙才能进入临界区，别人只能等待。

---

🔹**序列化盒子 (Serialization Box)**

**Q:** slide 为什么提到 “Serialization Box”？

**A:** 抽象模型：mutex 把所有访问共享变量的代码“放进同一个盒子里”，一次只能有一个线程进入，这样访问就被**序列化**了。

---

🔹**多个临界区**

**Q:** 一个 mutex 可以保护多个不同的临界区吗？

**A:** 可以。只要用相同的 mutex 上锁解锁，它们就是同一个“序列化盒子”，保证互斥。

---

🔹**多把锁与死锁 (Deadlock)**

**Q:** 为什么多把锁容易出问题？

**A:** 如果线程 1 先拿锁 m1 再等 m2，而线程 2 先拿 m2 再等 m1，就可能形成**循环等待** → 死锁。

  

**Q:** 如何表示死锁？

**A:** 用“等待图 (wait-for graph)”：

- 节点 = 线程持有/等待的锁。
    
- 边 = 从已持有的锁指向正在等待的锁。
    
- 出现**环**就表示死锁。

---
🔹**死锁 (Deadlock)**

**Q:** 多个线程如何产生死锁？

**A:** 如果线程 1 拿了锁 m1 等 m2，线程 2 拿了锁 m2 等 m1，就互相等待 → 死锁。可用“等待图 (wait-for graph)”表示：从持有的锁画箭头指向等待的锁，若出现环路即可能死锁。

  

**Q:** 死锁的四个必要条件是什么？

**A:**

1. 有界资源（有限的资源数量）
    
2. 请求并保持（持有部分资源还要等新的资源）
    
3. 不可抢占（不能强制剥夺资源）
    
4. 循环等待（形成一个环状等待链）
    
    → 四者同时成立才可能死锁。
    

  

**Q:** 处理死锁有哪两种思路？

**A:**

- **检测 (detection)**：判断系统是否死锁，难度大，检测后处理复杂。
    
- **预防 (prevention)**：用规则让死锁根本不可能发生（如限制加锁顺序），相对简单。课程推荐把死锁当作编程 bug，优先预防。
    

---

🔹**加锁层级 (Lock Hierarchies)**

**Q:** 什么是锁的层级 (Lock Hierarchy)？

**A:** 给每个锁分配等级。若已持有等级 ≥ i 的锁，就不能再申请等级 ≤ i 的锁，只能申请更高等级的锁。这样避免循环等待。

  

**Q:** 锁层级的实际规则例子？

**A:** 若线程已持有 level 2 和 3 的锁，就只能再申请 level 4+ 的锁，不允许申请更低等级。

---

🔹**条件加锁 (Conditional Locking)**

**Q:** 如果锁无法分层，如何避免死锁？

**A:** 用条件加锁：如果尝试第二把锁失败，就释放第一把锁重试，避免“请求并保持”。
```
while (1) {
  pthread_mutex_lock(&m2);
  if (!pthread_mutex_trylock(&m1)) break;
  pthread_mutex_unlock(&m2);
}
/* 同时持有 m1,m2 */
pthread_mutex_unlock(&m1);
pthread_mutex_unlock(&m2);
```
---
🔹**Mutex 小结**

**Q:** 什么时候共享数据不需要锁？

**A:** 当多个线程只是**只读**共享数据时。

  

**Q:** 如果多个线程读写共享数据怎么办？

**A:** 必须加锁，使用临界区访问。可以一把锁保护，也可能使用多把锁与不同线程交互。

---

🔹**超越 Mutex (Beyond Mutexes)**

**Q:** 什么时候 mutex 过于保守？

**A:**

1. 多数时候线程互不干扰，仅偶尔需要同步。
    
2. 有的线程只需要读数据，不必和写者完全互斥。
    
    例子：**生产者-消费者问题**、**读者-写者问题**、**栅栏同步 (Barrier)**。
    

---

🔹**生产者-消费者 (Producer-Consumer)**

**Q:** 为什么用一把大锁解决生产者-消费者太保守？

**A:** 因为大部分时间生产和消费可以并行，只有在缓冲区**满**或**空**时才需要阻塞。用大锁会让不冲突的操作也被锁住，降低效率。

  

**Q:** 软件里如何实现生产者-消费者？

**A:** 用**环形缓冲区 (circular buffer)**，并配合更合适的同步机制（如信号量）来在满/空时阻塞。

---

🔹**守护命令 (Guarded Commands)**

**Q:** 什么是 Guarded Command？

**A:** 一种伪代码结构：
```
when (guard) [
   // 一旦 guard 为真，原子地执行代码
]
```
**语义**：当 guard 为真时，**原子地**评估 guard 并执行命令序列。原子性表示“像瞬间完成一样”。

  

**Q:** Guarded Command 是否等于临界区？

**A:** 不同。临界区是通过锁保护的一段代码；Guarded Command 是抽象语义，表示 guard 检查+代码执行不可分割。

---

🔹**信号量 (Semaphores)**

**Q:** 信号量的定义是什么？

**A:** 一个非负整数 S，只能通过两种操作修改：

- P(S)：当 S>0 时执行 S--，否则阻塞。
    
- V(S)：执行 S++。
    
    初始化后不能直接修改。
    

  

**Q:** 如何用二值信号量实现互斥？

**A:** 初始化 S=1，进入临界区前 P(S)，退出时 V(S)。功能类似 mutex。
```
semaphore S = 1;
P(S);
x = x+1;
V(S);
```
**Q:** 计数信号量 (Counting Semaphore) 的用途？

**A:** S=N，表示同一时间允许最多 N 个线程进入。用于：生产者-消费者（记录空槽/满槽数）、连接池、限流等。

  

**Q:** 信号量和互斥锁的关键区别？

**A:** mutex 要求**同一线程 lock/unlock 成对**；semaphore 的 P 和 V 可以由**不同线程**执行（生产者 V，消费者 P），支持跨线程“配额交接”。


---
🔹**Producer-Consumer with Semaphores**

**Q:** 如何用信号量实现生产者-消费者？

**A:** 使用两个信号量：

- empty = B（空槽数，初始为缓冲区大小 B）
    
- occupied = 0（已占用槽数，初始 0）
    

  

代码骨架：
```
void Produce(char item) {
  P(empty);                // 等待至少一个空槽
  buf[nextin] = item;
  nextin = (nextin+1) % B; // 环形缓冲
  V(occupied);             // 增加已占用
}

char Consume() {
  P(occupied);             // 等待至少一个满槽
  char item = buf[nextout];
  nextout = (nextout+1) % B;
  V(empty);                // 增加空槽
  return item;
}
```
---

🔹**信号量在运行时的行为**

**Q:** 如果生产和消费速度相同，会怎样？

**A:** 双方可能不需要等待，生产者刚放进去消费者就取走，形成“流水线并行”。

  

**Q:** 如果生产者快，消费者慢，会怎样？

**A:** 当缓冲区满了，empty=0，生产者阻塞等待消费者消费。

  

**Q:** 如果消费者快，生产者慢，会怎样？

**A:** 当缓冲区空了，occupied=0，消费者阻塞等待生产者生产。

---

🔹**Mutex vs Semaphore**

**Q:** 为什么只用 mutex 不够？

**A:** 一把大锁只能保证互斥，过于粗粒度，所有操作被串行化。

信号量可以实现**条件同步**：只有在缓冲区满/空时才阻塞 → 并行度更高。

  

**Q:** 信号量最典型的用途是什么？

**A:** **生产者-消费者问题（也叫有界缓冲区问题）**。这几乎是信号量的“代表性应用”。

---

🔹**POSIX Semaphores**

**Q:** POSIX 提供了哪些信号量 API？

**A:**
```
sem_t sem;
sem_init(&sem, pshared, init_value);
sem_destroy(&sem);
sem_wait(&sem);     // P 操作
sem_trywait(&sem);  // 条件 P 操作
sem_post(&sem);     // V 操作
```
其中 pshared=0 表示只在同一进程内共享。

---

🔹**实现信号量的原理**

**Q:** 如何用 mutex 实现一个 P(S) 操作？

**A:** 伪代码：
```
while (1) {
  pthread_mutex_lock(&m);
  if (S > 0) {
    S = S - 1;
    pthread_mutex_unlock(&m);
    break;
  }
  pthread_mutex_unlock(&m);
  // 重试
}
```
这会导致 **busy waiting**，CPU 空转，不高效。

  

**Q:** 为什么 busy waiting 不好？

**A:** 线程在等待条件时不做有用工作，而且持续占用 CPU。

---

🔹**Guarded Commands 的一般实现**

**Q:** 如果 guard 很复杂怎么办？

**A:** 必须“采样”所有相关变量，并且用 mutex 保证在检查和执行之间不会被其他线程打断。

  

**Q:** Busy waiting 如何改进？

**A:** 引入 **条件变量 (Condition Variable, CV)**，让等待的线程进入睡眠队列，不占用 CPU。

---

🔹**Condition Variables**

**Q:** 什么是条件变量 (CV)？

**A:** 一种同步原语，保存等待某个条件的线程队列。

线程在 pthread_cond_wait() 里睡眠，直到被其他线程 signal 或 broadcast 唤醒。

  

**Q:** pthread_cond_wait() 做了什么？

**A:**

1. 必须在持有 mutex 时调用。
    
2. 原子地：释放 mutex → 阻塞等待事件。
    
3. 被唤醒时：重新获取 mutex，返回。
    

  

**Q:** pthread_cond_signal() vs pthread_cond_broadcast()？

**A:**

- signal() 唤醒一个等待线程。
    
- broadcast() 唤醒所有等待线程。
    

---

🔹**Guarded Commands with CV (POSIX 实现)**

**Q:** 如何用条件变量实现 guarded command？

**A:**
```
pthread_mutex_lock(&mutex);
while (!guard)
    pthread_cond_wait(&cv, &mutex);
... // guard 为真时执行代码
pthread_mutex_unlock(&mutex);
```
修改 guard 的地方：
```
pthread_mutex_lock(&mutex);
... // 修改 guard 的代码
pthread_cond_broadcast(&cv); // 唤醒等待 guard 的线程
pthread_mutex_unlock(&mutex);
```
**规则**：

- 所有 pthread_cond_wait/signal/broadcast 调用必须在 mutex 保护下进行。
    
- 如果不按规则来，可能出现 race condition（线程错过信号永远睡眠）。
    

---

✅ **总结**

- 信号量适合实现 **生产者-消费者**，能保证 buffer 满/空时正确阻塞。
    
- 但信号量用途有限，不适合更复杂的条件。
    
- 更通用的方法是 **条件变量 (CV)**，配合 mutex，实现任意复杂 guard 的 guarded command。

----
## **🔹POSIX Condition Variables**

  

Q: POSIX 条件变量是怎么实现 Guarded Command 的？

A: 使用 pthread_mutex_lock 和 pthread_cond_wait 实现：
```
pthread_mutex_lock(&mutex);
while (!guard)
    pthread_cond_wait(&cv, &mutex);
/* critical section */
pthread_mutex_unlock(&mutex);
```
修改 guard 时，需要 pthread_cond_signal() 或 pthread_cond_broadcast() 唤醒等待的线程。

  

Q: 为什么必须在持有锁时调用 pthread_cond_wait() 和 pthread_cond_signal()？

A: 避免 race condition。如果不加锁，可能 guard 刚被修改但线程还没等到信号就错过了。

---

## **🔹Readers–Writers Problem (读者–写者问题)**

  

Q: 读者写者问题的核心矛盾是什么？

A: 多个读者可以并发读取，但写者必须独占，且不能与读者同时进行。

  

Q: 基本伪代码如何实现？

A:
```
reader() {
  when (writers == 0) [ readers++; ]
  /* read */
  [ readers--; ]
}

writer() {
  when ((writers == 0) && (readers == 0)) [ writers++; ]
  /* write */
  [ writers--; ]
}
```
Q: 为什么 readers 像计数信号量 (counting semaphore)，writers 像二进制信号量 (binary semaphore)？

A:

- readers：可以多个 reader 同时增加，计数类似 counting semaphore。
    
- writers：要么有 0，要么有 1 个 writer，占用类似 binary semaphore。
    

---

## **🔹POSIX 实现 Reader 代码

  

Q: POSIX 版的 reader() 是怎么写的？

A:
```
reader() {
  pthread_mutex_lock(&m);
  while (!(writers == 0))
    pthread_cond_wait(&readersQ, &m);
  readers++;
  pthread_mutex_unlock(&m);

  /* read */

  pthread_mutex_lock(&m);
  if (--readers == 0)
    pthread_cond_signal(&writersQ);
  pthread_mutex_unlock(&m);
}
```
Q: 这个代码的执行流程？

A:

1. **进来时**：检查 writers == 0，否则在 readersQ 等待。
    
2. **进入读模式**：readers++，允许多个读者同时读。
    
3. **离开时**：readers--，如果这是最后一个读者（readers == 0），就唤醒一个写者。
    

---

## **🔹POSIX 实现 Writer 代码**

  

Q: Writer 的实现如何保证独占？

A:
```
writer() {
  pthread_mutex_lock(&m);
  while (!((readers == 0) && (writers == 0)))
    pthread_cond_wait(&writersQ, &m);
  writers++;
  pthread_mutex_unlock(&m);

  /* write */

  pthread_mutex_lock(&m);
  writers--;
  pthread_cond_signal(&writersQ);
  pthread_cond_broadcast(&readersQ);
  pthread_mutex_unlock(&m);
}
```
写者必须等到 **没有读者且没有其他写者** 才能进入。
```
#### **1.** 

#### **pthread_mutex_lock(&m);**

- 加锁，确保下面的检查条件和修改 writers 的操作是原子的，避免多个 writer/reader 并发修改共享变量。
    

---

#### **2.** 

#### **while (!((readers == 0) && (writers == 0))) pthread_cond_wait(&writersQ, &m);**

- **逻辑条件**：写者只有在 **没有读者** 且 **没有写者** 的时候才能进入。
    
- 否则就要进入等待队列 writersQ。
    
- pthread_cond_wait(&writersQ, &m)：
    
    1. 自动释放 m（让别的线程运行）。
        
    2. 把自己挂到 writersQ 队列上，阻塞等待。
        
    3. 被唤醒后会重新获得锁 m，再检查条件。
        
    
- while 是必须的，因为可能会被虚假唤醒 (spurious wakeup)。
    

---

#### **3.** 

#### **writers++;**

- 成功进入写模式后，把 writers 设为 1，表示当前有写者在写。
    

---

#### **4.** 

#### **pthread_mutex_unlock(&m);**

- 解锁，让其他线程可以继续检查条件。
    
- 注意：即使此时 writer 正在写，解锁是安全的，因为 writers = 1 会阻止其他线程（读者或写者）进入。
    

---

#### **5.** 

#### **/* write */**

- 写者对共享资源的真正写操作。
    
- 因为写是独占的，所以在这一段期间，其他读者和写者都会被阻塞。
    

---

#### **6.** 

#### **pthread_mutex_lock(&m);**

- 写完后，要修改 writers--，需要重新加锁，避免 race condition。
    

---

#### **7.** 

#### **writers--;**

- 写者完成工作，把写者数目减 1，变回 0。
    

---

#### **8.** 

#### **pthread_cond_signal(&writersQ);**

- 唤醒 **一个**等待的写者线程（如果有的话）。
    
- 这样可以让排队的写者继续执行。
    

---

#### **9.** 

#### **pthread_cond_broadcast(&readersQ);**

- 同时唤醒所有读者线程，因为一旦没有写者了，读者可以并行进入。
    

---

#### **10.** 

#### **pthread_mutex_unlock(&m);**

- 解锁，释放共享变量控制。
```

---

## **🔹Starvation Problem (饥饿问题)**

  

Q: 读者–写者问题中，写者可能被饿死吗？

A: 是的，如果一直有新的读者进来，写者可能永远得不到机会写。

  

Q: 如何解决写者饥饿？

A:

- 修改逻辑：一旦有写者到达，阻止新的读者进入，直到写者完成。
    
- 使用 active_writers 来保证同一时间只有一个 writer 真正写。
    

---

## **🔹POSIX 读写锁 (pthread_rwlock)**

  

Q: POSIX 提供的读写锁 API 有哪些？

A:

- 初始化与销毁：
```
 pthread_rwlock_init, pthread_rwlock_destroy
```
- 加锁与解锁：
```
pthread_rwlock_rdlock, pthread_rwlock_wrlock, pthread_rwlock_unlock
```
- 非阻塞与定时：
```
pthread_rwlock_tryrdlock, pthread_rwlock_trywrlock
pthread_rwlock_timedrdlock, pthread_rwlock_timedwrlock
```
## **🔹Barriers (屏障)**

  

Q: 什么是 barrier？

A: 屏障是一种同步机制，要求多个线程都到达某个点后，才能一起继续执行。

  

Q: POSIX barrier API？

A:
```
pthread_barrier_init, pthread_barrier_destroy, pthread_barrier_wait
```
Q: Barrier 的应用例子？

A: Fork-join 模型：多个线程并行执行，等到所有线程都完成，主线程才能继续。

---
### **线程安全 (Thread Safety)**

- **Q: 什么是线程安全？**
    
    - **A:** 线程安全指的是让那些在设计时未考虑线程问题的库函数，能够在多线程环境下安全地运行 。由于 Unix 是在线程普及之前开发的，其许多库函数本身并非线程安全的 。
        
- **Q: 旧版 Unix API 存在哪些线程不安全的问题？**
    
    - **A:**
        
        - **全局变量**: 例如 `errno`，在多线程中同时调用并都失败时，无法保证线程获取到正确的错误码 。
            
        - **共享数据**: 例如 `printf()`，多线程同时调用可能会导致输出交错混乱 。
            
- **Q: `errno` 的线程不安全问题有什么具体的代码示例吗？**
    
    - **A:** 在下面的代码中，如果两个线程同时调用 `IOfunc` 并且 `write` 都失败了，后一个线程设置的 `errno` 可能会覆盖前一个线程的 `errno`，导致前一个线程读取到错误的错误码 

```
int IOfunc(int fd) {
    extern int errno;
    if (write(fd, buffer, size) == -1) {
        if (errno == EIO)
            ...
    }
    ...
}
```
- **Q: 多线程使用 `printf()` 会出现什么问题？**
    
    - **A:** 两个线程的输出可能会混合在一起，导致内容错乱 。例如：
        
        - 线程 1 执行:
            
            `printf("goto statement reached");`
            
        - 线程 2 执行:
            
            `printf("Hello World\n");`
            
        - 最终可能得到的输出是:
            
            `goto Hello Wostatement reachedrld`
            

### **线程安全的实现方法**

- **Q: 如何解决 `errno` 的线程安全问题？**
    
    - **A:** POSIX 通过提供线程私有数据存储机制来解决 。通过将
        
        `errno` 定义为指向每个线程独立的存储位置，确保每个线程访问自己的 `errno` 副本 。这样无需修改应用程序代码，只需重新编译即可 。
        
- **Q: 对于返回全局数据的函数，如何使其线程安全？**
    
    - **A:** 添加一个“可重入”(`_r`)版本 。例如，非线程安全的
        
        `gethostbyname()` 返回一个指向全局变量的指针 ，而 POSIX 提供了
        
        `gethostbyname_r()` 版本，该版本要求调用者自行提供用于存储返回数据的缓冲区，从而解决了共享全局数据的问题 。
        
    - **非线程安全版本:**
        
        C
        
        ```
        struct hostent *gethostbyname(const char *name)
        ```
        
    - **线程安全的可重入版本:**
        
        C
        
        ```
        int gethostbyname_r(const char *name,
                            struct hostent *ret,
                            char *buf,
                            size_t buflen,
                            struct hostent **result,
                            int *h_errnop)
        ```
        
- **Q: 如何解决 `printf()` 这类函数的线程安全问题？**
    
    - **A:** 可以使用同步机制来包装库函数 。例如，对于 C 标准 I/O 库中的
        
        `FILE*` 对象，可以使用以下函数来加锁和解锁，以保护 `printf()` 这样的调用 ：
        
        C
        
        ```
        void flockfile(FILE *filehandle)
        int ftrylockfile(FILE *filehandle)
        void funlockfile(FILE *filehandle)
        ```
        

### **线程的定时与等待**

- **Q: 线程如何进行延时或等待？**
    
    - **A:**
        
        - **`nanosleep()`**: 可以让线程挂起一段指定的时间 ，但这是一种“尽力而为”的机制 。
            
            C
            
            ```
            struct timespec timeout, remaining_time;
            timeout.tv_sec = 3;         // seconds
            timeout.tv_nsec = 1000;     // nanoseconds
            nanosleep(&timeout, &remaining_time);
            ```
            
        - **`pthread_cond_timedwait()`**: 是一种更精确的等待方式，它允许线程等待一个条件变量，直到被唤醒或超过一个指定的**绝对时间点** (`abstime`) 。
            
            C
            
            ```
            // abstime 需要通过当前时间加上相对的超时时间来计算
            struct timespec absolute_timeout;
            ... // 计算 absolute_timeout
            pthread_mutex_lock(&m);
            while (!may_continue)
                pthread_cond_timedwait(&cv, &m, &absolute_timeout);
            pthread_mutex_unlock(&m);
            ```
            

### **线程的异常与信号 (Signal)**

- **Q: 什么是信号 (Signal)？**
    
    - **A:** 信号是一种操作系统提供的“回调机制” 或“软件中断” 。它用于响应异常（如算术错误 ）、外部事件（如定时器到期、键盘输入 ）或用户自定义事件 。例如，除以零的操作会产生一个信号 。
        
        C
        
        ```
        int x, y;
        x = 0;
        ...
        y = 16 / x; // 这会产生一个信号
        ```
        
- **Q: 如何向进程或线程发送信号？**
    
    - **A:**
        
        - **`kill(pid_t pid, int sig)`**: 发送一个指定的信号给某个进程 。
            
        - **`pthread_kill(pthread_t thr, int sig)`**: 在同一进程内，发送信号给指定的线程 。不过，若要终止线程，推荐使用 POSIX 的取消机制 (cancellation mechanism) 。


---
好的 👍，我来帮你梳理 **lecture7-slides2.pdf** 的新内容，并结合我们之前的风格解释细致一些：

---

### **🔹Thread Safety**

- **背景**
    
    Unix 最早设计时没有线程的概念，所有的库函数都是默认单线程环境。后来多线程流行起来后，这些库函数在多线程下就变得不安全了。
    
- **问题**
    
    比如全局变量 errno，或者 printf 中共享的 buffer。如果多个线程同时访问，就可能出现数据混乱。
    
- **解决**
    
    - 改写库函数 → 支持线程安全
        
    - 用同步机制（mutex 等）包装老函数
        
    

---

### **🔹Thread Safety vs. Reentrancy**

- **Thread-safe**: 多个线程可以同时调用函数，而不会互相影响。
    
- **Reentrant**: 即使函数在执行中被打断（例如信号或中断），再次进入也能安全执行。
    
- 二者不同，但很多时候 **线程安全**和**可重入**最终都要求类似的措施。
    

---

### **🔹Global Variables 问题**

- errno 是全局变量，如果两个线程几乎同时调用失败的系统调用，一个线程可能会覆盖另一个的 errno。
    
- **结果**: 某个线程读到的错误码可能不是它自己的。
    

---

### **🔹解决方案**

1. **线程私有数据 (Thread-specific data, TSD)**
    
    - POSIX 提供了机制，让每个线程有自己独立的 errno。
        
    - 类似 #define errno __errno(thread_ID)，每个线程访问到的是属于自己的那份。
        
    - Windows 里叫 **thread-local storage (TLS)**。
        
    
2. **Reentrant 版本 API**
    
    - 例如 gethostbyname() 原本返回一个指向全局变量的指针（不安全）。
        
    - POSIX 提供了 gethostbyname_r()，让调用者自己提供 buffer 存放返回结果 → 避免共享数据。
        
    

---

### **🔹Shared Data 问题**

- 多线程同时 printf，可能导致输出交错：
    

```
goto Hello Wostatement reachedrld
```

-   
    
- **解决**: 用 flockfile(FILE*) 等函数对 FILE* 上锁，确保同一时间只有一个线程在写。
    

---

### **🔹Sleeping 和 Timeout**

- **nanosleep()**: 可以指定休眠时间。
    
- **pthread_cond_timedwait()**:
    
    - 可以在等待条件变量时设置超时时间。
        
    - 超时后返回错误码（不是无限等）。
        
    

---

### **🔹Deviations: 信号和线程取消**

- **Unix signal**：最初设计用来优雅终止进程，例如 <Ctrl+C>。
    
- **用途**:
    
    - 异常（除零、非法内存访问等）
        
    - 外部事件（定时器、键盘输入）
        
    - 杀死或暂停进程
        
    
- **信号和中断类似**：可以阻塞、解除阻塞、挂起、恢复。
    
- **线程**：有 pthread_kill() 发送信号给线程，但推荐用 **pthread cancellation** 代替。
    

---

### **🔹信号的状态**

- **pending**: 信号已经产生，但被阻塞，还没被传递。
    
- **delivered**: 一旦解除阻塞，立即传递。
    
- 类似 CPU 硬件中断的 “disable/enable”。
    
---
