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