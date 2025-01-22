# OS实验1:应用程序与系统调用部分 报告

## 吕佳鸿 10235501436

### Sleep

这个实验的目的实现xv6的UNIX程序**`sleep`**：**`sleep`**应该暂停到用户指定的计时数。一个滴答(tick)是由xv6内核定义的时间概念，即来自定时器芯片的两个中断之间的时间。

实现如下： 

1. 用argc检测输入的参数是否为2个，若不是便报错退出
2. 若参数正确，系统调用sleep函数 并将以字符串形式输入的数字转换为整数最后退出

```c
#include "kernel/types.h"
#include "user/user.h"

int main(int argc ,char *argv[]){
    if (argc != 2){
        printf("error");
        exit(1);
    }
    sleep(atoi(argv[1]));
    exit(0);
}
```

### Pingpong

这个实验的目的编写一个使用UNIX系统调用的程序来在两个进程之间“ping-pong”一个字节，请使用两个管道，每个方向一个。父进程应该向子进程发送一个字节;子进程应该打印“`<pid>: received ping`”，其中`<pid>`是进程ID，并在管道中写入字节发送给父进程，然后退出;父级应该从读取从子进程而来的字节，打印“`<pid>: received pong`”，然后退出。

#### pipe

最开始没有太理解管道的概念上网搜了一下，大概是这样：管道是两个进程之间的连接，使得一个进程的标准输出成为另一个进程的标准输入。在 UNIX 操作系统中，管道可用于相关进程之间的通信（进程间通信）。

- 管道是单向通信，即我们可以使用管道，一个进程写入管道，另一个进程从管道读取。它打开一个管道，它是主内存的一个区域，被视为***“虚拟文件”\***。
- 创建进程及其所有子进程均可使用管道进行读写。一个进程可以向此“虚拟文件”或管道写入数据，而另一个相关进程可以从中读取数据。
- 如果某个进程在将内容写入管道之前尝试进行读取，则该进程将被暂停，直到写入内容为止。
- pipe 系统调用在进程的打开文件表中找到前两个可用位置，并将它们分配给管道的读写端。

并且，管道的行为符合**FIFO**（先进先出）原则，管道的行为类似于队列

![Lightbox](https://media.geeksforgeeks.org/wp-content/uploads/Process.jpg)

#### 代码

这版代码的思路就是使用两个管道 ，一个父进程到子进程，一个从子进程再到父进程，注意要及时关闭不必要的端，防止堵塞

```c
#include "kernel/types.h"
#include "user/user.h"

#define RD 0 //定义pipe的read端
#define WR 1 //定义pipe的write端

int main(int argc)
{
    int child_fd[2]; // 父到子
    int parent_fd[2]; // 子到父
    char buf[1];
    pipe(child_fd);
    pipe(parent_fd);
    int pid = fork();
    if (pid == 0) {
        close(child_fd[RD]);
        close(parent_fd[WR]); // 关闭写端 防止堵塞
        read(parent_fd[RD], buf, sizeof(buf)); // 读取父进程传来的字符（这个用一个char类型的buf来传递）
        close(parent_fd[RD]);
        fprintf(2,"%d: received ping\n", getpid());
        write(child_fd[WR], buf, sizeof(char)); // 向父进程写（也用buf）
        close(child_fd[WR]);
    } else {
        close(parent_fd[RD]);
        close(child_fd[WR]);
        write(parent_fd[WR], buf, sizeof(char)); // 向子进程写
        close(parent_fd[WR]);
        read(child_fd[RD], buf, sizeof(buf)); // 从子进程读取
        close(child_fd[RD]);
        fprintf(2,"%d: received pong\n", getpid());
    }
    exit(0);
}

```



在网上查的过程还看到另一张图如下：

![img](https://media.geeksforgeeks.org/wp-content/uploads/sharing-pipe.jpg)

及父进程和子进程可以共用一个管道，但要注意wait防止堵塞，按照这个思路实现如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define NULL ((void*)0)  // 手动定义 NULL
#define MSGSIZE 16

int main(void) {
    int fd[2];
    char buf[MSGSIZE];
    pipe(fd);
    int pid = fork();
    if (pid > 0){
        write(fd[1],"ping",MSGSIZE);
        wait(NULL);
        read(fd[0], buf, MSGSIZE);
        fprintf(2,"%d:received %s\n",getpid(), buf);
    } else {
        read(fd[0], buf, MSGSIZE);
        printf("%d:received %s\n",getpid(), buf);
        write(fd[1],"pong", MSGSIZE);
    }
    exit(0);
}
```

就是需要定义一下NULL，否则wait函数会报错。。。和上一个两个管道能简洁一点

两种都通过了测试，第二个方法还能快点

### Prime

这个实验的目的是使用`pipe`和`fork`来设置管道。第一个进程将数字2到35输入管道。对于每个素数，您将安排创建一个进程，该进程通过一个管道从其左邻居读取数据，并通过另一个管道向其右邻居写入数据。

通过查了一下参考资料，使用Eratosthenes的筛选法 以下是图片说明的过程

![img](https://xv6.dgs.zone/labs/requirements/images/p1.png)

代码如下：

```c
#include "kernel/types.h"
#include "user/user.h"

#define RD 0 // pipe读端
#define WR 1 // pipe写端

void transmit_data(int lpipe[2], int rpipe[2], int first)
{
  int data;
  // 从左管道读取数据
  while (read(lpipe[RD], &data, sizeof(int)) == sizeof(int)) {
    // 如果无法被当前质数整除，则将数据传递入右管道
    if (data % first != 0) {
      write(rpipe[WR], &data, sizeof(int));
    }
  }
  close(lpipe[RD]);
  close(rpipe[WR]);
}

int main(int argc, char *argv[])
{
  int p[2];
  pipe(p);  // 创建初始管道

  // 将 2 到 35 的整数写入管道
  for (int i = 2; i <= 35; i++) {
    write(p[WR], &i, sizeof(int));
  }

  close(p[WR]);  // 初始管道写端关闭

  while (1) {
    int first; // 读取管道中的第一个数
    if (read(p[RD], &first, sizeof(int)) != sizeof(int)) { // 如果没有数据可读，退出循环
      break;
    }

    // 打印该质数
    printf("prime %d\n", first);

    int new_pipe[2];
    pipe(new_pipe);  // 创建新管道

    if (fork() == 0) {
      // 子进程传递数据，筛除当前质数的倍数
      close(new_pipe[RD]);
      transmit_data(p, new_pipe, first);
      exit(0);  // 子进程完成后退出
    } else {
      // 父进程关闭旧管道并等待子进程完成
      close(new_pipe[WR]);
      close(p[RD]);
      wait(0);  // 等待子进程结束
      p[RD] = new_pipe[RD];  // 更新父进程的管道读取端
    }
  }

  close(p[RD]);  // 最后关闭管道
  exit(0);
}
```

最开始我是用的是递归的方法，反复调用prime函数，但编译时会警告，而xv6把所有的warning都视为错误，所以用迭代的方法完成。

### find

这个的实验目的是写一个简化版本的UNIX的`find`程序：查找目录树中具有特定名称的所有文件

思路主要是这三点

- 查看***user/ls.c\***文件学习如何读取目录
- 使用递归允许`find`下降到子目录中
- 不要在“`.`”和“`..`”目录中递归

整体的代码还是在ls.c的基础上进行修改的

```c
#include "kernel/types.h"
#include "kernel/fs.h"
#include "kernel/stat.h"
#include "user/user.h"


void find(char *path, const char *filename)
{
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  if ((fd = open(path, 0)) < 0) {
    fprintf(2, "find: cannot open %s\n", path);
    return;
  }

  if (fstat(fd, &st) < 0) {
    fprintf(2, "find: cannot fstat %s\n", path);
    close(fd);
    return;
  }

  //参数错误，find的第一个参数必须是目录
  if (st.type != T_DIR) {
    fprintf(2, "usage: find <DIRECTORY> <filename>\n");
    return;
  }

  if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf) {
    fprintf(2, "find: path too long\n");
    return;
  }
  strcpy(buf, path);
  p = buf + strlen(buf);
  *p++ = '/'; //p指向最后一个'/'之后
  while (read(fd, &de, sizeof de) == sizeof de) {
    if (de.inum == 0)
      continue;
    memmove(p, de.name, DIRSIZ); //添加路径名称
    p[DIRSIZ] = 0;               //字符串结束标志
    if (stat(buf, &st) < 0) {
      fprintf(2, "find: cannot stat %s\n", buf);
      continue;
    }
    //不要在“.”和“..”目录中递归
    if (st.type == T_DIR && strcmp(p, ".") != 0 && strcmp(p, "..") != 0) {
      find(buf, filename);
    } else if (strcmp(filename, p) == 0)
      printf("%s\n", buf);
  }

  close(fd);
}

int main(int argc, char *argv[])
{
  if (argc != 3) {
    fprintf(2, "usage: find <directory> <filename>\n");
    exit(1);
  }
  find(argv[1], argv[2]);
  exit(0);
}

```



### xargs

这个实验的目的是编写一个简化版UNIX的`xargs`程序：它从标准输入中按行读取，并且为每一行执行一个命令，将行作为参数提供给命令。

主要的思路就是就是每次从 stdin 中读取一行，然后将其与 xargs 后面的字符串拼起来形成一个命令，fork 出一个子进程并交给 exec 来执行。程序通过 read_line 函数从标准输入读取数据。read_line 会逐字节读取输入内容，跳过空格和制表符，找到非空白字符开始一个参数。遇到空格或制表符时，它认为这个参数已经结束，将其存储在 args 数组中。每当遇到换行符时，表示该行输入结束，read_line 返回 1，表明读取成功。如果读取到文件末尾，函数返回 0。

读取到完整的一行参数后，程序使用 fork() 创建子进程，然后在子进程中使用 exec() 来执行命令。exec() 使用 args[] 数组作为命令的参数列表，包含了预定义的命令和从标准输入读取到的参数。父进程等待子进程结束后，继续从输入中读取下一行数据，重复这个过程，直到标准输入结束或没有更多的参数。

代码如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "kernel/param.h"
#include "user/user.h"

static char buf[1024];
static char *args[MAXARG];
static int init_argc = 0;
static int argc_now = 0;

int read_line(int fd) {
    char c[2];
    argc_now = init_argc;
    int offset = 0;

    while (1) {
        // 跳过空白字符，找到第一个非空字符
        while (1) {
            if (read(fd, c, 1) != 1) {
                return 0; // 文件读取结束
            }
            if (c[0] == '\n') {
                return 1; // 读取到换行符
            }
            if (c[0] != ' ' && c[0] != '\t') {
                args[argc_now++] = buf + offset;
                buf[offset++] = c[0];
                break;
            }
        }

        // 继续读取一个完整的参数
        while (1) {
            if (read(fd, buf + offset, 1) != 1) {
                buf[offset] = '\0'; // 文件读取结束
                return 0;
            }
            if (buf[offset] == '\n') {
                buf[offset] = '\0'; // 行结束
                return 1;
            }
            if (buf[offset] == ' ' || buf[offset] == '\t') {
                buf[offset++] = '\0'; // 参数结束
                break;
            }
            ++offset;
        }
    }
}

int main(int argc, char *argv[]) {
    int pid, status;

    if (argc < 2) {
        fprintf(2, "Usage: xargs command ...\n");
        exit(1);
    }

    char *cmd = argv[1];
    init_argc = argc - 1;
    args[0] = cmd;

    for (int i = 1; i < argc; i++) {
        args[i] = argv[i + 1];
    }

    int more_input = 1;
    while (more_input) {
        more_input = read_line(0);  // 从标准输入读取
        args[argc_now] = 0; // 设置参数结尾

        if (more_input == 0 && argc_now == init_argc) {
            exit(0);
        }

        pid = fork();
        if (pid == 0) {
            exec(cmd, args); // 子进程执行命令
            printf("exec failed!\n");
            exit(1);
        } else {
            wait(&status); // 父进程等待子进程结束
        }
    }

    exit(0);
}
```

### Trace

本实验主要是实现一个追踪系统调用的函数

根据提示，首先再`proc`结构体中添加一个数据字段，用于保存`trace`的参数。并在`sys_trace()`的实现中实现参数的保存

由于`struct proc`中增加了一个新的变量,当`fork`的时候我们也需要将这个变量传递到子进程中

接下来应当考虑如何进行系统调用追踪了，根据提示，这将在`syscall()`函数中实现。下面是实现代码，

```c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();//myproc()：宏，当前的进程
  // trap 代码将用户寄存器保存到当前进程的 trapframe 中
  num = p->trapframe->a7;//num是系统调用号
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();//a0保存返回值
    if((p->mask>>num)&1)
      printf("%d: syscall %s -> %d\n", p->pid, syscall_name[num], p->trapframe->a0);//进程id、系统调用的名称、返回值
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}

```

### Sysinfo

这个实验的目的是添加一个系统调用`sysinfo`，它收集有关正在运行的系统的信息。

这个实验有两个主要内容 ：

1. 在*kernel/kalloc.c*中添加一个函数用于获取空闲内存量
2. 实现`sys_sysinfo`，将数据写入结构体并传递到用户空间

第一个通过阅读kalloc.c,内存是使用链表进行管理的，因此遍历`kmem`中的空闲链表就能够获取所有的空闲内存，用一个free_memory来记录空的内存

第二个sys_info收集系统的基本信息，并将这些信息复制到提供的内存地址。用户进程通过调用 sys_sysinfo 来获取这些信息。在执行过程中，系统首先从参数中获取目标地址，接着填充 sysinfo 结构体，最后将数据通过 copyout 函数传输到用户进程的地址空间。

```c
uint64
sys_sysinfo(void){
  uint64 addr;
  struct proc *p = myproc();
  if (argaddr(0,&addr) < 0){
    return -1;
  }
  struct sysinfo info;
  info.freemem = free_mem();
  info.nproc = nproc();

  if (copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0) {
    return -1;
  }
  return 0;
} 
```

## 问题探索

在做第二个syscall实验的时候，感觉对其中几个常出现的东西不太了解，在网上搜索学习了一下，在这里做一个记录

#### trapframe

trapframe (陷阱帧) 是用于保存进程上下文的结构，特别是当一个进程发生中断、异常或者系统调用等情况，内核需要处理用户进程时。

它位于用户页表中，紧挨着 trampoline 页，用于保存用户进程的寄存器状态和某些内核状态。

当 usertrap() 被调用时，用户态的寄存器会被保存到 trapframe 中。处理完成后，usertrapret() 会从 trapframe 中恢复寄存器并返回到用户空间。

trapframe 中包含了所有的寄存器，包括临时寄存器、保存寄存器以及用于传递参数和返回值的寄存器。

#### argraw

```c
static uint64
argraw(int n)
{
  struct proc *p = myproc();
  switch (n) {
  case 0:
    return p->trapframe->a0;
  case 1:
    return p->trapframe->a1;
  case 2:
    return p->trapframe->a2;
  case 3:
    return p->trapframe->a3;
  case 4:
    return p->trapframe->a4;
  case 5:
    return p->trapframe->a5;
  }
  panic("argraw");
  return -1;
}
```

通常，系统调用的参数会存放在 a0 到 a5 这几个寄存器中（在 RISC-V 架构中，a0 到 a7 是用于传递参数和返回值的寄存器，其中 a0 到 a5 用于最多 6 个参数）。函数通过 switch 语句来根据 n 的值（0 到 5）返回对应寄存器的值。

#### Syscall.c

```c
void syscall(void)
{
  int num;
  struct proc *p = myproc();  // 获取当前进程
  // 从 trapframe 中获取系统调用号，存储在 a7 寄存器中
  num = p->trapframe->a7;
  
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // 根据系统调用号 num，执行相应的系统调用处理函数
    p->trapframe->a0 = syscalls[num]();
  } else {
    // 如果系统调用号无效，返回 -1
    p->trapframe->a0 = -1;
  }
}
```

​	

1. **获取系统调用号**从 trapframe->a7 中获取系统调用号。a7 是用户进程在发起系统调用时使用的寄存器，它会在陷阱发生时被保存到 trapframe 结构中。
2. **调用相应的系统调用处理函数**：系统调用号存储在 num 中，syscall() 函数通过查找一个系统调用处理函数数组 syscalls[]，根据系统调用号 num 执行相应的函数。
3. **返回结果**：系统调用处理函数的返回值会被存储在 trapframe->a0 中。这是因为 a0 寄存器通常用于保存返回值。当系统调用处理完成后，trapframe->a0 会包含系统调用的返回值，返回到用户态时，用户进程可以从 a0 寄存器中读取返回值。

而系统调用号，便是我们定义在syscall.h中的

```c
#define SYS_fork    1
#define SYS_exit    2
...
```

syscall将返回的值存在a0寄存器内

#### proc

在proc.c和proc.h中，进程控制块存储在一个全局数组 proc 中，该数组定义在 proc.c 文件中。每个元素都是一个 struct proc 类型的结构体，表示一个进程。NPROC 是一个宏，表示系统中最大进程数。

```c
struct proc proc[NPROC];
```

proc的结构体是这样定义的：

```c
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID
  int mask;

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)

};

```

并且，我们也可以看到，在procstate中，定义了进程的五种状态`enum procstate { UNUSED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };`即：

​	•	UNUSED**（未使用）**：进程控制块目前未被使用，可以分配给新的进程。

​	•	SLEEPING**（睡眠状态）**：进程正在等待某些条件（例如 I/O 事件）的发生，不能运行，直到该条件满足。

​	•	RUNNABLE**（可运行状态）**：进程已经准备好运行，可以由调度程序选择并分配 CPU。

​	•	RUNNING**（运行中）**：进程正在运行，即当前被 CPU 执行。

​	•	ZOMBIE**（僵尸进程）**：进程已经终止，但其父进程还没有通过调用 wait() 获取其终止状态，因此它仍然保留在系统中。

#### kalloc

在 kernel/kalloc.c 中，空闲内存页通过 struct run 结构体来表示。所有空闲的内存页形成一个链表，链表中的每个节点都是一个 struct run 结构体。

```c
struct run {
  struct run *next;
};
```

所有空闲内存页通过 freelist 指针来管理。freelist 是一个 struct run * 类型的变量，它指向空闲页链表的头部。当系统需要分配一个新的物理内存页时，它会从 kmem.freelist 中取出第一个空闲页，并将 freelist 指向下一个空闲页。分配后的内存页将不再位于空闲链表中。当某个物理内存页不再需要时，它会被重新加入到 freelist 链表中。释放时，系统会将该页插入到空闲页链表的头部。

#### copyout

```c
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

1. pagetable_t pagetable 这是用户进程的页表，类型是 pagetable_t（页表的指针）。它用来管理进程的虚拟地址到物理地址的映射。页表将虚拟地址翻译为物理地址，用于访问进程内存。

2. uint64 dstva 这是用户空间的目标虚拟地址，表示数据要复制到哪里。dstva 是用户进程中的虚拟地址，表示数据应该被放置到用户地址空间中的哪个位置。

3. char *src 这是一个指向内核缓冲区的指针，数据将从这个缓冲区中读取。它指向内核空间的源数据，意味着这个缓冲区中包含了要复制给用户进程的数据。

4. uint64 len 这是要复制的字节数，表示从 src 缓冲区复制到用户空间的字节数。函数会复制 len 字节的数据到用户进程的虚拟地址空间中。

### 总结

![image-20240928164450862](/Users/kerwinlv/OS learning/OSlabutilgrade.jpg)

![image-20240928164514436](/Users/kerwinlv/OS learning/OSlabutilgrade.jpg)
