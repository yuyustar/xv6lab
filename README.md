# xv6lab

本项目主要是基于本项目以基于RISC-V指令集架构的xv6操作系统为基础，对其系统调用、内存管理、进程管理、文件系统、中断等模块进行扩展与优化。

参考资料：

- 课程官网：[6.S081 Fall 2020](https://pdos.csail.mit.edu/6.828/2020/xv6.html)
- 课程视频：[6.S081--bilibili](https://www.bilibili.com/video/BV19k4y1C7kA?from=search&seid=5542820295808098475)
- 视频翻译：[6.S081课程翻译--gitbook](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/)
- 书籍翻译：[6.S081 XV6BOOK翻译](https://xv6.dgs.zone/)

以下ReadMe内容为学习制作该项目时的一些文字记录，包括个人思考、实现流程和相关问题等。

如果想要查看相关实现代码细节可以切换分支。

# lab1

## copy：读写

### 前置知识：

每个进程都有从0开始的文件描述符的私有空间

0读取标准输入，1将输出写入文件，错误消息2，控制台默认文件扫描符

#### read(),write()

系统调用以字节为单位读取或写入已打开的以文件描述符命名的文件

read(fd，buf，n)从文件描述符fd读取最多n字节，将它们复制到buf，并返回读取的字节数，引用文件的每个文件描述符都有一个与之关联的偏移量。read从当前文件偏移量开始读取数据，然后将该偏移量前进所读取的字节数：（也就是说）后续读取将返回第一次读取返回的字节之后的字节。当没有更多的字节可读时，read返回0来表示文件的结束。

系统调用write(fd，buf，n)将buf中的n字节写入文件描述符，并返回写入的字节数。只有发生错误时才会写入小于n字节的数据。与读一样，write在当前文件偏移量处写入数据，然后将该偏移量向前推进写入的字节数：每个write从上一个偏移量停止的地方开始写入。

cat就是基于以上两个基础函数实现，标准输入到标准输出。

### lab1.1copy

```c++
#include "kernel/types.h"
#include "user/user.h"

int main()
{
  char buf[64];

  while(1){
      //控制台读入数据，
    int n = read(0, buf, sizeof(buf));
    //无输入结束程序
    if(n <= 0)
      break;
    //输出到控制台，调用系统函数
    write(1, buf, n);
  }

  exit(0);
}
```

在Makefile 152行添加配置，并执行

![image-20240312134629466](C:\Users\yuyu\AppData\Roaming\Typora\typora-user-images\image-20240312134629466.png)![image-20240312134804246](C:\Users\yuyu\AppData\Roaming\Typora\typora-user-images\image-20240312134804246.png)



## ping pong：管道

### 前置知识

管道是作为一对文件描述符公开给进程的小型内核缓冲区，一个用于读取，一个用于写入。将数据写入管道的一端使得这些数据可以从管道的另一端读取。管道为进程提供了一种通信方式。

#### fork（）：

文件描述符和fork相互作用，使I/O重定向更容易实现。fork复制父进程的文件描述符表及其内存，以便子级以与父级在开始时拥有完全相同的打开文件。系统调用`exec`替换了调用进程的内存，但保留其文件表。此行为允许shell通过`fork`实现I/O重定向，在子进程中重新打开选定的文件描述符，然后调用`exec`来运行新程序。

##### fork和exec分离的用处：

在这两个调用之间，shell有机会对子进程进行I/O重定向，而不会干扰主shell的I/O设置。

#### pipe（）：

创建管道，可以传入数组，并在数组中记录读写文件描述符，fork后，父子进程都指向管道的文件描述符。

子进程调用close和dup使文件描述符0指向管道的读取端（前面说过优先分配最小的未使用的描述符），然后关闭p中所存的文件描述符，并调用exec运行wc。当wc从它的标准输入读取时，就是从管道读取。父进程关闭管道的读取端，写入管道，然后关闭写入端。

如果没有可用的数据，则管道上的read操作将会进入等待，直到有新数据写入或所有指向写入端的文件描述符都被关闭，在后一种情况下，read将返回0，就像到达数据文件的末尾一样。事实上，**read在新数据不可能到达前会一直阻塞，这是子进程在执行上面的wc之前关闭管道的写入端非常重要的一个原因**：如果wc的文件描述符之一指向管道的写入端，wc将永远看不到文件的结束。

使用两个管道进行父子进程通信，需要注意的是如果管道的写端没有close，那么管道中数据为空时对管道的读取将会阻塞。因此对于不需要的管道描述符，要尽可能早的关闭。

pipe(p[2])会将两个文件描述符保存到p[2]中，一个管道可被写入多次，内容不会被覆盖，可分多次读出。管道是全双工的

##### 管道和临时文件

用管道实现的命令，也可以不使用管道实现

在这种情况下，管道相比临时文件至少有四个优势

- 首先，管道会自动清理自己；在文件重定向时，shell使用完`/tmp/xyz`后必须小心删除
- 其次，管道可以任意传递长的数据流，而文件重定向需要磁盘上足够的空闲空间来存储所有的数据。
- 第三，管道允许并行执行管道阶段，而文件方法要求第一个程序在第二个程序启动之前完成。
- 第四，如果实现进程间通讯，管道的**阻塞**式读写比文件的非阻塞语义更高效。

#### code

```c++
int main(){
    //pipe1(p1)：写端父进程，读端子进程
    //pipe2(p2)；写端子进程，读端父进程
    int p1[2],p2[2];
    //来回传输的字符数组：一个字节
    char buffer[] = {'X'};
    //传输字符数组的长度
    long length = sizeof(buffer);
    //父进程写，子进程读的pipe
    pipe(p1);
    //子进程写，父进程读的pipe
    pipe(p2);
    //子进程
    if(fork() == 0){
        //关掉不用的p1[1]、p2[0]
        close(p1[1]);
        close(p2[0]);
        //子进程从pipe1的读端，读取字符数组
        if(read(p1[0], buffer, length) != length){
            printf("a--->b read error!");
            exit(1);
        }
        //打印读取到的字符数组
        printf("%d: received ping\n", getpid());
        //子进程向pipe2的写端，写入字符数组
        if(write(p2[1], buffer, length) != length){
            printf("a<---b write error!");
            exit(1);
        }
        exit(0);
    }
    //关掉不用的p1[0]、p2[1]
    close(p1[0]);
    close(p2[1]);
    //父进程向pipe1的写端，写入字符数组
    if(write(p1[1], buffer, length) != length){
        printf("a-->b write error!");
        exit(1);
    }
    //父进程从pipe2的读端，读取字符数组
    if(read(p2[0], buffer, length) != length){
        printf("a<--b read error!");
        exit(1);
    }
    //打印读取的字符数组
    printf("%d: received pong\n", getpid());
    //等待进程子退出
    wait(0);
    exit(0);
}
}
```

![image-20240312141741461](C:\Users\yuyu\AppData\Roaming\Typora\typora-user-images\image-20240312141741461.png)

![image-20240312142308051](C:\Users\yuyu\AppData\Roaming\Typora\typora-user-images\image-20240312142308051.png)

## sleep：

```c++
int main(int argn, char *argv[]){
    if(argn != 2){//异常判断
        fprintf(2, "must 1 argument for sleep\n");
        exit(1);
    }
    int sleepNum = atoi(argv[1]);//转化为asii码
    printf("(nothing happens for a little while)\n");
    sleep(sleepNum);
    exit(0);
}
```

## primes

详见primes.c

思路：类似递归，但是是多进程版本不断从左边的临近管道中筛选，最后传送给右邻居，每次第一个数据都是素数。

调用函数，包括上述的write，read，pipe，fork，close

## find

类似于ls.c 的代码思路

主要用到的函数

open，stat，memmove等

![image-20240312151143666](C:\Users\yuyu\AppData\Roaming\Typora\typora-user-images\image-20240312151143666.png)

- ## xargs

实现类似unix xargs类似功能，shell中的 ‘|‘ 是将标准输出转换为标准输入，linux中的xargs命令是将标准输入转换为shell命令行参数。

主要思路和函数

1. 一个字符一个字符的读取，遇到换行或空格则为一个word.
2. 命令行一行最长4095bytes
3. free(arg)会把内存空间的内容也删掉。



# lab2

## System call tracing

### 前置知识

- xv6的内存管理和系统调用流程。
- system call的用户空间代码：user/user.h和user/usys.pl
- system call的kernel空间代码：kernel/syscall.h和user/syscall.c
- 进程相关代码：kernel/proc.h和kernel/proc.c

#### 思路：

增加一个叫做trace的系统调用，让进程在调用mask的系统调用时，输出对应的进程号，调用名名和返回值。另外修改所有系统调用的入口函数，让它根据进程的mask输出对应的值。

#### system call调用链路

1.在user/user.h做函数声明；

2.Makefile调用usys.pl（perl脚本）生成usys.S，里面写了具体实现，通过ecall进入kernel，通过设置寄存器a7的值，表明调用哪个system call

3.ecall表示一种特殊的trap，转到kernel/syscall.c:syscall执行

4.syscall.c中有个函数指针数组，即一个数组中存放了所有指向system call实现函数的指针，通过寄存器a7的值定位到某个函数指针，通过函数指针调用函数

#### 具体实现方法：

1.创建新系统调用首先在内核中合适的位置（取决于要实现的功能属于什么模块，理论上随便放都可以，只是主要起绑定作用），实现的内核调用，由于系统调用进程操作，所以放入到`sysproc.c`。

```c++
uint64
sys_trace(void)
{
  int mask;

  if(argint(0, &mask) < 0) // 通过读取进程的 trapframe，获得 mask 参数
    return -1;
  
  myproc()->syscall_trace = mask; // 设置调用进程的 syscall_trace mask
  return 0;
}
```

2.在`syscall.h`中加入新系统调用的序号

```c++
 #define SYS_trace  22 
```

3.在`kernel/syscall.c`中使用 extern 全局声明新的内核调用函数，并且在系统调用映射表中，加入从前面定义的编号到系统调用函数指针的映射

- 新增函数定义sys_trace
- 函数指针数组也需要新增sys_trace
- 新建一个数组存放system_call的名称
- 根据mask值打印system call

4.在 `usys.pl` 中，加入用户态到内核态的跳板函数，运行后会生成usys.S创建文件，里面定义了每个系统调用的用户态跳板函数，跳板函数 trace() 使用 CPU 提供的 ecall 指令，调用到内核态

5.在user.h用户态的头文件中添加定义，使得用户态程序可以找到该跳板入口函数。

调用流程的主要目的是实现用户态和内核态的良好隔离。

并且由于内核与用户进程的页表不同，注册也不能互通，所以参数无法直接通过 C 语言参数的形式传过来，而是需要使用 argaddr、argint、argstr 等系列函数，从进程的 trapframe 中读取用户进程注册中的参数。

同时由于页表不同，指针也不能直接互通访问（按钮内核不能直接对用户态传进来的指针进行解引用），而是需要利用copyin、copyout方法结合进程的页表，才能顺利找到用户态指针（逻辑地址）对应的物理内存地址。

6.在proc.h中修改proc结构的定义，添加syscall_trace字段，用mask的方式来记录trace的系统调用。

7.在 proc.c 中，创建新进程的时候，为新添加的 syscall_trace 附上默认值 0（否则初始状态下可能会有垃圾数据），修改fork函数，让子进程可以继承父进程的syscall_trace



## sysinfo

### 前置知识

要把结构体从内核内存拷贝到用户进程内存中，先写入虚拟地址，再进行拷贝

空闲页相关：

xv6中，空闲内存页的记录方式是，将空虚内存页**本身**直接用于链表节点，形成一个空闲内存页链表，每次需要分配，将链表根部的页分配出去。每次需要恢复，则将空闲内存页分配出去。页这个作为新的根节点，把原来的freelist链表接收后面。注意这里是**直接使用空闲页本身**作为链表节点，所以不需要使用额外的空间来存储空闲页链表，常见的记录闲置页的方法有：闲置表法、闲置链表法、位示图法（位图法）、成组链接法。这里xv6采用闲置链表法。

内存是使用链表进行管理的，因此遍历空闲链表就能够获取所有的空闲内存

#### 实现方法：

内存遍历：

1.在kernel/kalloc.c中添加一个函数用于获取空闲内存量

```c++
struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;
```

遍历该内存量中的多有空闲列表

```c++
void freebytes(uint64 *dst)
{
  *dst = 0;
  struct run *p = kmem.freelist; // 用于遍历

  acquire(&kmem.lock);
  while (p) {
    *dst += PGSIZE;
    p = p->next;
  }
  release(&kmem.lock);
}
```

2.在kernel/proc.c中添加一个函数获取进程数，遍历proc数组，统计处于活动状态的进程即可

3.实现sys_sysinfo，将数据写入结构体并传递到用户空间

4.在kernel/syscall.h中宏定义 #define SYS_sysinfo 23

5.修改user/usys.pl中新增一个entry

6.在user/user.h中新增sysinfo结构体、sysinfo函数声明

7.kernel/syscall.c中新增函数定义

8.函数指针数组新增一个"sysinfo"元素

9.kernel/proc.h中mask数组长度为24

10.kernel/defs.h做出函数声明

11.kernel/sysproc.c加入sysinfo.h的header头，新增sysinfo的实现

12.新写一个user/sysinfo.c的用户程序



# Lab3

## page tables

添加一个打印页表的内核函数，以如如下格式打印出传进的页表，用于后面两个实验调试用

### 前置知识：

RISC-V指令（用户和内核指令）使用的是虚拟地址，而机器的RAM或物理内存是由物理地址索引的。RISC-V页表硬件通过将每个虚拟地址映射到物理地址来为这两种地址建立联系。

RISC-V 的逻辑地址寻址是采用三级页表的形式，9 bit 一级索引找到二级页表，9 bit 二级索引找到三级页表，9 bit 三级索引找到内存页，最低 12 bit 为页内偏移（即一个页 4096 bytes），如果转换地址所需的三个页面索引中的任何一个不存在，页式硬件就会引发页面故障异常，并让内核来处理该异常。

为了减少从物理内存加载 PTE 的开销，RISC-V CPU 将页表条目缓存在 Translation Look-aside Buffer (TLB) 中。

与物理内存和虚拟地址不同，虚拟内存不是物理对象，而是指内核提供的管理物理内存和虚拟地址的抽象和机制的集合。

三级结构使用了一种更节省内存的方式来记 页面索引。在大范围的虚拟地址没有被映射的常见情况下，三级结构可以忽略整个页面目录。

Xv6为每个进程维护一个页表，用以描述每个进程的用户地址空间，外加一个单独描述内核地址空间的页表。内核配置其地址空间的布局，以允许自己以可预测的虚拟地址访问物理内存和各种硬件资源。

QEMU将设备接口作为内存映射控制寄存器暴露给软件，这些寄存器位于物理地址空间`0x80000000`以下。内核可以通过读取/写入这些特殊的物理地址与设备交互；这种读取和写入与设备硬件而不是RAM通信。

内核使用“直接映射”获取内存和内存映射设备寄存器；也就是说，将资源映射到等于物理地址的虚拟地址。

有几个内核虚拟地址不是直接映射：

- 蹦床页面(trampoline page)。它映射在虚拟地址空间的顶部；用户页表具有相同的映射。第4章讨论了蹦床页面的作用，但我们在这里看到了一个有趣的页表用例；一个物理页面（持有蹦床代码）在内核的虚拟地址空间中映射了两次：一次在虚拟地址空间的顶部，一次直接映射。
- 内核栈页面。每个进程都有自己的内核栈，它将映射到偏高一些的地址，这样xv6在它之下就可以留下一个未映射的保护页(guard page)。保护页的PTE是无效的（也就是说`PTE_V`没有设置），所以如果内核溢出内核栈就会引发一个异常，内核触发`panic`。如果没有保护页，栈溢出将会覆盖其他内核内存，引发错误操作。恐慌崩溃（panic crash）是更可取的方案。*（注：Guard page不会浪费物理内存，它只是占据了虚拟地址空间的一段靠后的地址，但并不映射到物理地址空间。）*

虽然内核通过高地址内存映射使用内核栈，是它们也可以通过直接映射的地址进入内核。另一种设计可能只有直接映射，并在直接映射的地址使用栈。然而，在这种安排中，提供保护页将涉及取消映射虚拟地址，否则虚拟地址将引用物理内存，这将很难使用。

内核必须在运行时为页表、用户内存、内核栈和管道缓冲区分配和释放物理内存。xv6使用内核末尾到`PHYSTOP`之间的物理内存进行运行时分配。它一次分配和释放整个4096字节的页面。它使用链表的数据结构将空闲页面记录下来。分配时需要从链表中删除页面；释放时需要将释放的页面添加到链表中。

模拟如上的 CPU 查询页表的过程，对三级页表进行遍历，然后按照一定格式输出。

### 实现方法

在kernel/vm.c中定义，复制xv6中常有的递归释放页表函数freewalk，修改部分代码为打印。

```c++
int pgtblprint(pagetable_t pagetable, int depth) {
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(pte & PTE_V) { // 如果页表项有效
      // 按格式打印页表项
      printf("..");
      for(int j=0;j<depth;j++) {
        printf(" ..");
      }
      printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));

      // 如果该节点不是叶节点，递归打印其子节点。
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        // this PTE points to a lower-level page table.
        uint64 child = PTE2PA(pte);
        pgtblprint((pagetable_t)child,depth+1);
      }
    }
  }
  return 0;
}

int vmprint(pagetable_t pagetable) {
  printf("page table %p\n", pagetable);
  return pgtblprint(pagetable, 0);
}
```

修改kernel/defs.h中增加vmprint()的声明：int vmprint(pagetable_t pagetable);

在kernel/exec.c中增加对vmprint()的调用：vmprint(p->pagetable);

## 内核页表

xv6 原本的设计是，用户进程在用户态使用各自的用户态页表，但是一旦进入内核态（例如使用了系统调用），则切换到内核页表（通过修改 satp 寄存器，trampoline.S）。然而这个内核页表是全局共享的，也就是全部进程进入内核态都共用同一个内核态页表：

本 Lab 目标是让每一个进程进入内核态后，都能有自己的独立内核页表，

### 前置知识：

xv6 设计中，所有处于内核态的进程都共享同一个页表，即意味着共享同一个地址空间。由于 xv6 支持多核/多进程调度，同一时间可能会有多个进程处于内核态，所以需要对所有处于内核态的进程创建其独立的内核态内的栈，也就是内核栈，供给其内核态代码执行过程。

xv6 在启动过程中，会在 procinit() 中为所有可能的 64 个进程位都预分配好内核栈 kstack，具体为在高地址空间里，每个进程使用一个页作为 kstack，并且两个不同 kstack 中间隔着一个无映射的 guard page 用于检测栈溢出错误。

在 xv6 原来的设计中，内核页表本来是只有一个的，所有进程共用，所以需要为不同进程创建多个内核栈，并 map 到不同位置（见 procinit() 和 KSTACK 宏）。

改变的设置中，每一个进程都会有自己独立的内核页表，并且每个进程也只需要访问自己的内核栈，而不需要能够访问所有 64 个进程的内核栈。所以可以将所有进程的内核栈 map 到其**各自内核页表内的固定位置**（不同页表内的同一逻辑地址，指向不同物理内存）。

#### exec函数实现执行过程

会涉及到文件操作和所得使用，有些内容要看到后面才能理解

栈是单独一个页面，显示的是由exec创建后的初始内容

exec是创建地址空间的用户部分的系统调用。它使用一个存储在文件系统中的文件初始化地址空间的用户部分，exec中的函数执行与下列内存分配息息相关。

![image-20240313120545800](C:\Users\yuyu\AppData\Roaming\Typora\typora-user-images\image-20240313120545800.png)

包含命令行参数的字符串以及指向它们的指针数组位于栈的最顶部。再往下是允许程序在main处开始启动的值（即main的地址、argc、argv），这些值产生的效果就像刚刚调用了main(argc, argv)一样。

首先，不同进程的页表将用户地址转换为物理内存的不同页面，这样每个进程都拥有私有内存。第二，每个进程看到的自己的内存空间都是以0地址起始的连续虚拟地址，而进程的物理内存可以是非连续的。第三，内核在用户地址空间的顶部映射一个带有蹦床（trampoline）代码的页面，这样在所有地址空间都可以看到一个单独的物理内存页面。

##### 流程：

先检查ELF文件，exec使用proc_pagetable分配一个没有用户映射的新页表，再找到分配内存的物理地址，写入ELF段的每一页，使用readi从文件中读取。

初始化用户栈， 在当前页之后再分配两页内存并使用第二页作为用户栈，在用户栈中压入参数字符串  

准备剩下的栈空间在ustack变量中

读取exec函数传递进来的参数列表，直至遇到结束符

argc作为返回值返回，存放在a0中，这个调用的返回值就是argc 

sp作为argv，即指向参数0的指针的指针，存放在a1中返回

ustack数组存放的是函数参数的地址，ustack中的前三个条目是伪返回程序计数器（fake return program counter）、argc和argv指针。

栈顶指针下移，给存放的参数留下足够空间 ，多下移一个字节是为了存放结束符

xv6执行许多检查来避免这些风险。例如，`if(ph.vaddr + ph.memsz < ph.vaddr)`检查总和是否溢出64位整数，危险在于用户可能会构造一个ELF二进制文件，其中的`ph.vaddr`指向用户选择的地址，而`ph.memsz`足够大，使总和溢出到0x1000，这看起来像是一个有效的值。

##### 疑问

*为什么要将sp设置到trapframe的a1寄存器中？sp指向是当前的栈顶位置，它本身的值也是程序参数argv的值。这个值在返回之后何时被放入到用户栈中，或者何时被调用*

解答：

trapframe是一个专门用来存放上下文状态的专用物理内存区域，每个进程在创建时都会分配一个物理页作为trapframe页，这个物理页的物理地址会保存在进程结构体的p->trapframe指针中。最后，在proc_pagetable函数中会将高地址TRAPFRAME映射到p->trapframe这个物理地址中。

看到一个trapframe中不仅保存了RISC-V的所有通用寄存器，还保存了包括内核栈指针、内核页表等等一系列关键信息。现在trapframe的地址已经保存在a0寄存器里了，是时候将现有的寄存器数据存放到trapframe帧里了，在阅读下面的代码时注意一件事情，我们这里操纵的所有地址都是虚拟地址，这些代码能够正确地执行，是因为我们目前还处于用户地址空间，一旦切换到内核地址空间，a0这个地址就不会正确对应到trapframe了。

将CPU的所有寄存器存放到trapframe中的对应位置

注意前32个字节用来存放切换到内核时需要使用的信息

s0-s11都是被调用者保存的寄存器，我们切换到新进程之后一定会用到这些**寄存器**，所以**context**保存它们是应该的。保存堆栈指针sp是为了可以在以后进程重新被调度时内核栈可以被成功恢复，保存返回地址ra则是为了让进程恢复调度时指令可以连续执行，因为**上下文切换会使得指令流切换到另一个进程中去**，原有的ra寄存器的内容会被篡改。

在Xv6中，context = callee-saved registers + ra + sp，

Xv6中是通过多次交换的方式来实现进程切换的，调用一次swtch函数完成一次交换动作，

①：在发生调度之前，旧进程P1的上下文正占据着CPU核心中的硬件寄存器堆，在执行着自己的逻辑。发生调度之后，swtch的第一个动作(对应于汇编的前半段)，就是将原本正在硬件寄存器堆中执行的上下文存回位于内存的P1上下文结构体中。a0寄存器
②：CPU将内核进程的上下文载入硬件寄存器堆，开始调度过程，这对应于swtch代码的后半段。a1寄存器
③：调度器找到了合适的可以调度的进程，内核进程再次被换出，硬件寄存器堆中的值被存回内核进程上下文，这对应于第二次调用swtch时前半段代码做的事情。
④：新进程上下文被载入硬件寄存器堆，这对应于第二次调用swtch时后半段代码，自此核心开始在新进程上运行



### 实现流程

在kernel/proc.h的struct proc中新加kernelPageTable

修改kernel/vm.c，新增一个vmmake()方法可以创建一个内核页表（不包含CLINT的映射）

在kernel/vm.c，修改vminit()方法，内部由vmmake()实现，此处为全局内核页表创建过程，另外加上CLINT的映射。

在kernel/proc.c，修改procinit()方法，不再于此方法中为每个进程分配内核栈

在kernel/proc.c，修改allocproc()，在此时创建内核页表，并在内核页表上分配一个内核栈

在kernel/proc.c，修改scheduler()，在swtch()切换进程前修改satp，保证进程执行期间用的是进程内核页表，切换完后再修改satp为全局内核页表

kernel/proc.c，修改freeproc()方法，添加对进程内核页表的资源

在kernel/proc.c，新增proc_free_kernel_pagetable()方法，用于释放进程内核页表指向的物理内存，以及进程内核页表本身

在kernel/vm.c，新增uvmfree2()方法，用于释放内核页表上的内核栈



## copyin函数修改

在上一个实验中，已经使得每一个进程都拥有独立的内核态页表了，这个实验的目标是，在进程的内核态页表中维护一个用户态页表映射的副本，这样使得内核态也可以对用户态传进来的指针（逻辑地址）进行解引用。这样做相比原来 copyin 的实现的优势是，原来的 copyin 是通过软件模拟访问页表的过程获取物理地址的，而在内核页表内维护映射副本的话，可以利用 CPU 的硬件寻址功能进行寻址，效率更高并且可以受快表加速。

要实现这样的效果，我们需要在每一处内核对用户页表进行修改的时候，将同样的修改也同步应用在进程的内核页表上，使得两个页表的程序段（0 到 PLIC 段）地址空间的映射同步。

#### 思路

与 uvmdealloc 功能类似，将程序内存从 oldsz 缩减到 newsz。但区别在于不释放实际内存，用于内核页表内程序内存映射与用户页表程序内存映射之间的同步

为映射程序内存做准备。实验中提示内核启动后，能够用于映射程序内存的地址范围是 [0,PLIC)，我们将把进程程序内存映射到其内核页表的这个范围内，首先要确保这个范围没有和其他映射冲突。

查阅 xv6 book 可以看到，在 PLIC 之前还有一个 CLINT（核心本地中断器）的映射，该映射会与我们要 map 的程序内存冲突。查阅 xv6 book 的以及 start.c 可以知道 CLINT 仅在内核启动的时候需要使用到，而用户进程在内核态中的操作并不需要使用到该映射。

![image-20240313134958583](C:\Users\yuyu\AppData\Roaming\Typora\typora-user-images\image-20240313134958583.png)

修改 kvm_map_pagetable()，去除 CLINT 的映射，这样进程内核页表就不会有 CLINT 与程序内存映射冲突的问题。但是由于全局内核页表也使用了 kvm_map_pagetable() 进行初始化，并且内核启动的时候需要 CLINT 映射存在，故在 kvminit() 中，另外单独给全局内核页表映射 CLINT，同时在 exec 中加入检查，防止程序内存超过 PLIC。

再进行同步映射，在每个修改到进程用户页表的位置，都将相应的修改同步到进程内核页表中。一共要修改：fork()、exec()、growproc()、userinit()。

#### 实现流程：

在kernel/proc.c中，修改userinit方法

在kernel/proc.c中，修改fork方法，进程页表的mapping复制到进程的内核页表

在kernel/exec.c中，修改exec()，在用户进程页表重新生成完后，取消进程内核页表之前的映射，在进程内核页表，建立新进程页表的映射，并添加用户空间地址不能大于PLIC的判断

在sysproc.c中，修改sys_sbrk()，在内存扩张、缩小时，相应更改进程内核页表

在kernel/proc.c中，修改freeproc和proc_free_kernel_pagetable方法，取消进程内核页表地址映射

在kernel/defs.h中，添加copyin_new()、copyinstr_new()的声明

在kernel/vm.c中，替换copyin()、copyinstr()为copyin_new()、copyinstr_new()



# Lab4

本实验探索如何使用陷阱实现系统调用。您将首先使用栈做一个热身练习，然后实现一个用户级陷阱处理的示例。

Xv6的语境中所谓陷阱的触发有以下三种情况：

- 系统调用
- 严重错误（比如除0错误)
- 设备中断

用户态陷阱包含上面的三种触发情况，而内核态陷阱只包含后两种情况。作为用户态陷阱中最具有代表性的类型，对系统调用的全流程进行分析会帮助我们梳理清楚从用户态陷阱的每一个步骤。

在RISC-V标准中，将异常(exception)定义为当前CPU运行时遇到的与指令有关的不寻常情况，而使用中断(interrupt)定义为因为外部异步信号而引起的让控制流脱离当前CPU的事件。而陷阱(trap)表示的则是，由异常或者中断引起的控制权转移到陷阱处理程序的过程。其实Xv6的定义和RISC-V的定义是相互兼容的，我们说在Xv6中有三种方式会触发陷阱：系统调用、中断和程序异常，其实系统调用中的ecall指令和程序执行中的异常都属于RISC-V标准中定义的异常情况，而设备中断则属于下面我们将要介绍的全局中断。

### 前置知识

#### 系统调用

1.用户态进程执行系统调用

2.系统在此时会执行对应的函数，系统调用的函数存储在一个汇编文件中，函数有三个指令

1. 在a7寄存器中放入系统调用号
2. 中断指令ecall，内核态的钥匙
3. ret返回到系统调用的下一条指令

ecall作用：关中断；从用户态切换到内核态；设置寄存器保存原pc、原模式并设置中断产生的原因；跳转到trampoline中的uservec函数实现。

为什么有以上内容的跳转？因为ecall指令为了简介，所以没有实现页表切换，内核栈也没有切换，因此无法执行内核中的代码。

所以trampoline usrvec函数的作用主要是切换到内核页表、切换到内核栈和保存寄存器，也就是上下文保存和用户内核态的切换。

由于用户表和内核表的映射位置都是固定且相同的，无论在哪，映射到了页表最顶部的一页，切换页表之后，寄存器指向的仍旧是trampoline的代码，返回时候也不会切换页表。

3.其中uservec函数最后跳转到usertrap函数执行，进入v内核，根据ecall中设置的终端产生原因来进行不同的操作，对于系统的调用，进一步调用syscall函数执行。

4.根据ecall中的指令来系统调用号来调用对应的函数sys_write;

5.按照函数规定的要求读取寄存器，对于write函数来说就是文件描述符，读缓冲区，数据缓冲区大小，再调用真正执行功能的函数。

6.执行完后，返回到syswrite，syscall函数，usertrap函数，执行完后会调用usertrapret函数，设置部分寄存器的值来控制返回到用户进程的状态。

简单总结一下**usertrapret**做的事情：

- 关中断(直到执行SRET之前，始终不响应中断)
- 设置stvec到uservec
- 设置trapframe中与内核有关的信息，为下一次陷阱处理做准备
- 正确地设置sepc，跳转回正确的地址继续执行

**调用userret函数**，准备恢复现场、切换页表、设置sscratcuserret恢复现场、切换到用户页表、设置sscratch。

在usertrapret的最后，它通过函数指针的方式调用了userret函数，这段函数定义在trampoline.S的最后作为系统调用整个流程的收尾

其实这里的逻辑几乎和uservec是一致的，只是流程上恰好反过来。值得注意的是位于最后的sret指令，它几乎和ecall指令是反过来的。 我们前面将ecall指令比作进入内核的钥匙，这里的sret就可以看作是一把锁。具体来说，sret指令完成了以下一些事情(由硬件完成的)：

将sepc中保存的值放入PC，sepc的值在usertrapret中已经被设置好了，对于系统调用来说，它返回的是系统调用的下一条指令
将权限模式设置为sstatus中的SPP位，这在usertrapret中也已经设置好了，0表示返回用户模式
将sstatus中SIE位的值设置为SPIE，在usertrapret中SPIE被设置为1表示开启中断
最后将sstatus中的SPP置为0，SPIE置为1

执行了ret指令了，控制流再次返回了getcmd函数

##  RISC-V assembly (easy)

回答的一些问题（将答案存储在***answers-traps.txt\***文件中）：

1. 哪些寄存器保存函数的参数？例如，在`main`对`printf`的调用中，哪个寄存器保存13？

   call.c执行`make fs.img`编译它，并在***user/call.asm\***中生成可读的汇编版本。

   ```c++
    printf("%d %d\n", f(8)+1, 13);
    24:	4635              li	a2,13		# 将13放置到a2寄存器
    26:	45b1              li	a1,12		# 将12放置到a1寄存器
    28:	00000517          auipc	a0,0x0		# a0 = pc = 0x28
    2c:	7a050513          addi	a0,a0,1952 	# a0 = a0 + 1952 = 0x28 + 0x7a0 = 0x7c8
    30:	00000097          auipc	ra,0x0		# ra = pc = 0x30
    34:	5f8080e7          jalr	1528(ra) 	# pc = ra + 1528 = 0x30 + 0x5f8 = 0x628
    											# ra = pc + 4 = 0x38
   ```

   13被放置到了a2寄存器中，而表示计算结果的f(8) + 1 = 8 + 3 + 1 = 12则被放置到了a1寄存器中，a0指向了一个地址7c8，可以想见这个地址应该是指向printf函数中第一个输出格式字符串的地址。上述的参数传递是符合RISC-V的calling convention的，事实上在传递整数参数时，如果参数个数少于8个，它们都会被放置在a0-a7中进行传递。

2. `main`的汇编代码中对函数`f`的调用在哪里？对`g`的调用在哪里(提示：编译器可能会将函数内联）

   编译器直接将对f函数的调用结果硬编码到了代码中，这样可以极大程度地减少函数调用过程的开销。

3. `printf`函数位于哪个地址？

   从上述代码中可以看出，printf函数的地址就是最后一行汇编执行完成之后的PC的值，为0x628

4. 在`main`中`printf`的`jalr`之后的寄存器`ra`中有什么值？

   ra寄存器的值应该是0x38，使用GDB调试一下call.c这个用户程序，看计算结果是否正确，调试的步骤如下：

- 打开一个命令行终端，输入：make CPUS=1 qemu-gdb，打开gdb-server等待调试器连接；

- 打开另外一个命令行终端，输入调试器命令：gdb-multiarch；

- 然后在调试器一端，将要调试的用户态文件加载进来file user/_call，并在jalr指令处(虚拟地址是0x34)打一个断点
- 使用continue指令继续内核的启动过程，注意因为调试的是用户态程序call，只有在进入操作系统之后执行这个程序才可以出发上述的断点。我输入continue，切换到GDB服务器端，可以看到内核已经打印出了一些信息，并且GDB显示已经触发了断点
- 触发的断点并不是call.c这个程序中的，而是在启动内核的某个瞬间PC恰好等于了0x34这个地址值，GDB就触发了这个断点。GDB只会检查当前PC值是否等于你设定的断点位置，但它并不管这是哪段程序中的，使用disas指令看看当前上下文的汇编代码
- 可以看到当前执行的汇编代码和call.c中的printf的反汇编完全不一致，所以现在触发的这个所谓断点并不是位于call.c中的，我们直接用continue指令跳过这个断点，于是内核这时完全启动成功了，这时我们在终端中执行一下call程序，可以看到它被断点阻塞住了
- 回到GDB一端，可以看到再次触发了我们的断点，进入到程序中，用disas查看一下反汇编，发现和call.asm中的代码完全一致
- 这次是真的阻塞在了我们期待的地方，于是我们输入stepi指令，让其向前执行一条汇编，此时控制流将转向printf函数，同时ra寄存器会完成设置
- 看当前ra寄存器的值即可，使用info registers ra指令，它的值是0x38

5.运行下列代码：

```c++
unsigned int i = 0x00646c72;
printf("H%x Wo%s", 57616, &i);
```

程序的输出是什么？这是将字节映射到字符的ASCII码表

输出取决于RISC-V小端存储的事实。如果RISC-V是大端存储，为了得到相同的输出，你会把`i`设置成什么？是否需要将`57616`更改为其他值？

写入call.c编译运行一下，输出了一个"He110 World"，其中e110是57616的16进制表示，而rld是无符号整数i中每个字节的对应字符。如果现在RISC-V存储改为大端，i的值就应该初始化为0x00726c46了，而57616无需修改，因为**大小端存放并不会改变它转化为16进制数之后的结果**。

 字节的排列方式有两个通用规则。例如，将一个多位数的低位放在较小的地址处，高位放在较大的地址处，则称**小端序**；反之则称**大端序**。

## Backtrace

实现一个backtrace函数，可以打印函数调用栈，从而方便进行debug。最重要的一个前备知识是了解并熟悉Xv6中的函数调用栈帧结构，知道了各个寄存器的相对存放位置。

### 前置知识

xv6中的函数调用栈结构：

在使用Frame Pointer时，这个寄存器其实指向的是上一级栈帧的最后一个位置，这是因为RISC-V中栈指针(stack pointer)是遵循满递减原则的。那么FP-8一定存放的是上一级函数的返回地址ra，FP-16这个地址一定存放的是上一级栈帧指针Previous FP，这是我们实现回溯的重要条件。

fp 指向当前栈帧的开始地址，sp 指向当前栈帧的结束地址。 （栈从高地址往低地址生长，所以 fp 虽然是帧开始地址，但是地址比 sp 高）栈帧中从高到低第一个 8 字节 fp-8 是 return address，也就是当前调用层应该返回到的地址。
栈帧中从高到低第二个 8 字节 fp-16是 previous address，指向上一层栈帧的 fp 开始地址。
剩下的为保存的寄存器、局部变量等。一个栈帧的大小不固定，但是至少 16 字节。

![image-20240314121844586](C:\Users\yuyu\AppData\Roaming\Typora\typora-user-images\image-20240314121844586.png)

知道了栈帧的基本结构之后，我们会发现这样的多层嵌套调用的函数的栈帧本质上构成了一个链表，链表的入口就是当前存放在FP寄存器中的值，链表的next指针就是Previous FP指针(FP-16)。最后就是回溯的终点，我们知道链表的终点一般是一个空指针(nullptr)，这里也是一样的，RISC-V abi中明确规定了FP链表的最后一个Previous FP位置的值为0(The end of the frame record chain is indicated by the address zero appearing as the next link in the chain)。但我们在实验中没有这么做，Xv6的内核实现中整个栈只有一页(4K)大小，所以我们完全可以通过地址值去检测是否回溯到了最后一个栈帧

### 实现流程

1. 在kernel/defs.h添加定义
2. 修改kernel/riscv.h中增加r_fp()的实现，用来读取寄存器s0
3. 在kernel/printf.c中增加backtrace()的实现，如果已经到达栈底，返回地址和fp寄存器
4. 在kernel/sysproc.c的sys_sleep()函数中调用backtrace()
5. 在xv6代码库中执行make qemu，xv6中执行bttest，bttest调用sleep(system call)。依次调用trap.c、syscall.c、sysproc.c。

## Alarm

### 前置知识

##### 内核陷阱：

在研究内核陷阱流程中一个重要的是定时器中断，首先它是一种由CLINT转发而来的本地中断，定时器中断**往往会导致CPU的调度**，进而将陷阱的处理流程变得错综复杂。

内核态陷阱是没有系统调用这种情况的，同时因为内核代码的相对可靠性，内核出现严重错误时一般会使用panic停止内核的工作，所以从内核陷阱在Xv6中几乎就是中断(定时器中断、外部中断等)的代名词，这部分逻辑将会在kerneltrap函数中看到。

usertrap函数将内核态陷阱处理程序kernelvec的地址放置到寄存器stvec中，这样在内核发生陷阱时PC就会从kernelvec中开始执行。和用户态进入陷阱不同，因为此时已经进入了内核态，所以有很多准备工作已经完成了，比如内核页表和内核堆栈指针等等(我们在uservec中使用汇编代码已经完成了这些寄存器的初始化)。

只需要保存寄存器，调用kerneltrap，回复寄存器并返回即可。
之前我们在研究系统调用的时候，usertrap函数负责在内核态中鉴别陷阱的原因并调用不同的处理函数来处理之。其实kerneltrap函数也是一样的，因为在内核态没有系统调用这一说，这里导致陷阱的原因只剩下中断和程序错误而引起的异常。

kerneltrap首先尝试调用devintr来尝试处理中断，如果devintr并不能识别陷阱的来源，那么说明这不是一个中断导致的陷阱，而是因为程序异常而导致的，那么就会触发一个panic并中止内核的运行。

在kerneltrap函数完成之后，就会返回到上述的kernelvec函数中恢复寄存器并返回到内核被中断的地方继续执行下去。

**时钟中断**是一种特殊类型的中断，在RISC-V架构中它是由CLINT(Core Local Interrupt Controller)来管理的。所谓时钟中断被硬连线为一个M-Mode下的中断，在处置此中断时本质上借用了S-Mode下的软中断来实现。

 CLINT中有两种寄存器与时钟中断有关：

 mtime：它记录了从输入时钟源RTCCLK输入的总时钟周期数

mtimecmp： 当mtime的时间超过mtimecmp寄存器中的值之后，会自动触发时钟中断(MIP.mtip)

所以这里将mtimecmp寄存器的值设置为mtime+interval

就是相当于设定了时钟终端间隔为interval

另外就是timer_scratch这个数组，它起到了保存寄存器和一些必要信息的作用，从作用上来说它和用户态陷阱里的trapframe很像，但是它属于内核代码的组成部分，这是因为时钟中断的地位不同寻常，它是**构成操作系统核心功能(CPU调度)的基础**。

**时钟中断的触发与响应：**第一阶段timervec
通过上述的初始化设置，现在CLINT已经可以周期性地产生时钟中断了。因为我们在上面设置了mtvec寄存器的内容，使它指向timervec，那么现在一旦一个时钟中断被触发，就会跳转到timervec这个地址，将mtimecmp中的值加上interval，相当于为下一次时钟中断重置计数值，引发一个S-Mode下的软中断

当发生时钟中断信号时，RISC-V CPU会将自身特权等级提升至M-Mode(在此之前它可能位于U-Mode或者S-Mode，即分别对应到用户态和内核态)，然后将mtvec寄存器中的值加载到PC中开始执行上述汇编代码，并最终在触发了一个S-Mode软中断之后返回，返回之后的程序会去紧接着响应这个S-Mode下的软中断，进而完成时钟中断的功能，

时钟中断的响应是两段式的

接下来，evintr识别与转发，如何响应S-Mode下的软中断，devintr函数相当于一个中转站，对各种设备中断(包括软中断)进行转发和处理，并**根据返回值告知usertrap和kerneltrap函数本次处理的中断类型**

devintr代码中对S-Mode软中断的识别和处理如下，当发现本次中断的类型是S-Mode软中断时，devintr将会清除调sip.SSIP位，并返回值2。

然后usertrap和kerneltrap在接收到返回值为2时，会接着调用yield()让出当前CPU的使用权。

#### 思路

通过test0，如何调用处理程序是主要的问题。程序计数器的过程是这样的：

1. ecall指令中将PC保存到SEPC
2. 在usertrap中将SEPC保存到p->trapframe->epc
3. p->trapframe->epc加4指向下一条指令
4. 执行系统调用
5. 在usertrapret中将SEPC改写为p->trapframe->epc中的值
6. 在sret中将PC设置为SEPC的值

可见执行系统调用后返回到用户空间继续执行的指令地址是由p->trapframe->epc决定的，因此在usertrap中主要就是完成它的设置工作。

一个进程在执行时，定时器可能会在其中某个不确定的时刻触发一个时钟中断。这时候将会进入系统调用的相应流程，首先经由trampoline进入usertrap函数，对这个时钟中断进行处理，并适时地进行alarm响应过程。

- 记录当前已经经过的ticks总数，计数器加一
- 如果到了应该触发handler的间隔时间
- 首先将trapframe完整保存在proc的alarmframe中，将trapframe中的所有信息保存
- 修改trapframe中的epc，使得陷阱将会返回到用户态下的handler函数中 p->trapframe->epc = p->Handler
- 设置标志位，表明当前进程正处于alarm的处理流程中，不再响应其他alarm

之后会执行一些用户定义的动作，并在最后执行sigreturn系统调用，于是操作系统再次陷入内核态，经过usertrap的转发之后进入内核的sys_sigreturn系统调用，sys_sigreturn的实现非常简单，只是简单的恢复现场并重置计数器

恢复了在执行用户态程序时的现场，并将计时器和标志全部初始化，这样进程就可以成果响应下一次alarm了，接下来经过usertrapret的恢复，现在进程又回到了最早的程序中，且现场没有受到任何影响。

#### 实现流程

1）修改Makefile

2）修改user/usys.pl

3）修改user/user.h

4）修改kernel/syscall.h

5）修改kernel/syscall.c

以上是为了添加系统调用 sigalarm 和 sigreturn

6）在kernel/proc.h

7）在kernel/sysproc.c，编写 sigalarm 和 sigreturn

8）在kernel/trap.c，使得陷阱将会返回到用户态下的handler函数中 p->trapframe->epc = p->Handler



# Lab5

### 前置知识

实现一个内存页懒分配机制，在调用 sbrk() 的时候，不立即分配内存，而是只作记录。在访问到这一部分内存的时候才进行实际的物理内存分配。

页面故障的三种：

cpu无法将虚拟地址转化为物理地址时，cpu会发生页面异常

- 加载页面错误 (当加载指令无法转换其虚拟地址时)
- 存储页面错误 (当存储指令无法转换其虚拟地址时) 
- 指令页面错误 (当指令的地址无法转换时)。

scause寄存器中的值指示页面错误的类型，stval寄存器包含无法翻译的地址。

#### 写时复制

让父子最初共享所有物理页面，但将它们映射为只读。因此，当子级或父级执行存储指令时，risc-v CPU引发页面错误异常。

为了响应此异常，内核复制了包含错误地址的页面。它在子级的地址空间中映射一个权限为读/写的副本，在父级的地址空间中映射另一个权限为读/写的副本。

更新页表后，内核会在导致故障的指令处恢复故障进程的执行。

由于内核已经更新了相关的PTE以允许写入，所以错误指令现在将正确执行。

子进程会在`fork`之后立即调用`exec`，用新的地址空间替换其地址空间。在这种常见情况下，子级只会触发很少的页面错误，内核可以避免拷贝父进程内存完整的副本。

#### 懒分配

首先，当应用程序调用`sbrk`时，内核增加地址空间，但在页表中将新地址标记为无效。

其次，对于包含于其中的地址的页面错误，内核分配物理内存并将其映射到页表中

由于应用程序通常要求比他们需要的更多的内存

内核仅在应用程序实际使用它时才分配内存

用户态程序通过`sbrk`系统调用来在堆上分配内存，而`sbrk`则会通过`kalloc`函数来申请内存页面，之后将页面映射到页表当中。

当申请小的空间时，上述过程是没有问题的。但是如果当进程一次申请很大的空间，如数GB的空间，再使用上述策略来一页页地申请映射的话就会非常的慢（1GB/4KB=262,144）。这时候就引入了lazy allocation技术，当调用`sbrk`时不进行页面的申请映射，而是仅仅增大堆的大小，当实际访问页面时，就会触发缺页异常，此时再申请一个页面并映射到页表中，这是再次执行触发缺页异常的代码就可以正常读写内存了。

通过lazy allocation技术，就可以将申请页面的开销平摊到读写内存当中去，在`sbrk`中进行大量内存页面申请的开销是不可以接受的，但是将代价平摊到读写操作当中去就可以接受了

#### 磁盘分页

如果应用程序需要比可用物理RAM更多的内存，内核可以换出一些页面: 将它们写入存储设备 (如磁盘)，并将它们的PTE标记为无效。如果应用程序读取或写入被换出的页面，则CPU将触发页面错误。

然后内核可以检查故障地址。如果该地址属于磁盘上的页面，则内核分配物理内存页面，将该页面从磁盘读取到该内存，将PTE更新为有效并引用该内存，然后恢复应用程序。

为了给页面腾出空间，内核可能需要换出另一个页面。此功能不需要对应用程序进行更改，并且如果应用程序具有引用的地址 (即，它们在任何给定时间仅使用其内存的子集)，则该功能可以很好地工作。

结合分页和页面错误异常的其他功能包括自动扩展栈空间和内存映射文件。

### 思路

修改sbrk函数，使其不调用growproc函数进行页面分配，关键就是p->sz += n将堆大小增大，然后注释掉growproc。

当系统发生缺页异常时，就会进入到usertrap函数中，此时scause寄存器保存的是异常原因（**13表示是因为load引起的page fault**；**15表示是因为store引起的page fault**；12表示是因为指令执行引起的page fault。），stval是引发缺页异常的地址。

在usertrap判断scause为13或15后，就可以读取stval获取引发异常的地址，之后调用lazy_alloc对该地址的页面进行分配即可。在这里不需要进行p->trapframe->epc += 4操作，因为我们要返回发生异常的那条指令并重新执行

在lazy_alloc函数中，首先判断地址是否合法，之后通过PGROUNDDOWN宏获取对应页面的起始地址，然后调用kalloc分配页面，memset将页面内容置0，最后调用mappages将页面映射到页表中去。

sys_sbrk函数中的if(n < 0)部分，当参数为负数时，调用uvmdealloc取消分配。

lazy_alloc函数中的addr >= p->sz部分，当访问的地址大于堆的大小时就说明访问了非法地址，注意这里是>=而不是>。

在fork函数中通过uvmcopy进行地址空间的拷贝，我们只要将其中panic的部分改为continue就行了，当页表项不存在时并不是说明出了问题，直接跳过就可以了

当进程通过read或write等系统调用访问未分配页面的地址时，并不会通过页表硬件来访问，也就是说不会发生缺页异常；在内核态时是通过walkaddr来访问用户页表的，因此在这里也要对缺页的情况进行处理。
当出现pte == 0 || (*pte & PTE_V) == 0时，就说明发生了缺页，这时只要调用lazy_alloc进行分配，之后再次使用walk就能正确得到页表项了。

通过addr < p->trapframe->sp判断，当地址小于栈顶地址时就说明发生了非法访问

### 实现流程

1.在kernel/sysproc.c中去除growproc()

2.修改kernel/trap.c的usertrap()，使得在发生fault page时，给虚拟地址分配物理页

修改kernel/vm.c的uvmunmap()，让进程销毁时，对于尚未分配实际物理页的虚拟地址，不做处理，出现pte == 0 || (*pte & PTE_V) == 0时

修改kernel/sysproc.c的sys_sbrk()

修改kernel/vm.c的uvmunmap()，让进程销毁时，对于尚未分配实际物理页的虚拟地址，不做处理，两个continue，调用walk函数

修改kernel/vm.c

修改kernel/vm.c中的copyout，write方法将会用到尚未分配物理页的虚拟地址，在copyout中分配物理页

修改kernel/vm.c中的copyout，write方法将会用到尚未分配物理页的虚拟地址，在copyout中分配物理页

修改kernel/trap.c中的usertrap()

#### 问题：

两个continue

wal'k函数执行时，level==0时索引为4的项却是不存在的，此时walk不再检查PTE_V标志等信息，而是直接返回，因此即使虚拟地址对应的PTE实际不存在，walk函数的返回值也可能不为0，level为0时，有效索引为0~3，因此索引为4时返回的是最后一个有效PTE后面的一个地址。因此我们不能仅靠PTE为0来判断虚拟地址无效，还需要再次检查返回的PTE中是否设置了PTE_V标志位

# Lab6

在进程 fork 后，不立刻复制内存页，而是将虚拟地址指向与父进程相同的物理地址。在父子任意一方尝试对内存页进行修改时，才对内存页进行复制。 物理内存页必须保证在所有引用都消失后才能被释放，这里需要有引用计数机制。

在原始的XV6中，fork函数是通过直接对进程的地址空间完整地复制一份来实现的。但是，拷贝整个地址空间是十分耗时的，并且在很多情况下，程序立即调用exec函数来替换掉地址空间，导致fork做了很多无用功。即使不调用exec函数，父进程和子进程的代码段等只读段也是可以共享的，从而达到节省内存空间的目的。同时COW也可以将地址空间拷贝的耗时进行延迟分散，提高操作系统的效率。



### 思路

首先就是要对fork函数进行修改，使其不对地址空间进行拷贝。

fork函数会调用uvmcopy进行拷贝，因此只需要修改uvmcopy函数就可以了：删去uvmcopy中的kalloc函数，将父子进程页面的页表项都设置为不可写，并设置COW标志位（在页表项中保留了2位给操作系统，这里用的是第8位#define PTE_COW (1L << 8)）

设置一个数组用于保存内存页面的引用计数，由于会涉及到并行的问题，因此也需要设置一个锁，同时定义了一些辅助函数。

修改kfree函数，使其只有在引用计数为1的时候释放页面，其他时候就只减少引用计数

修改kalloc函数，使其在分配页面时将引用计数也设置为1：这里注意要判断r是否为0，kalloc实现时没有当r==0时就返回

在usertrap中加入判断语句，这里只需要处理scause==15的情况，因为13是页面读错误，而COW是不会引起读错误的。

在cowcopy函数中先判断COW标志位，当该页面是COW页面时，就可以根据引用计数来进行处理。如果计数大于1，那么就需要通过kalloc申请一个新页面，然后拷贝内容，之后对该页面进行映射，映射的时候清除COW标志位，设置PTE_W标志位；而如果引用计数等于1，那么就不需要申请新页面，只需要对这个页面的标志位进行修改就可以了

#### 实现流程

修改kernel/vm.c，新增int refNum[32768];来记录关联物理页的页表数量。32768是根据(PHYSTOP-KERNBASE)/PGSIZE得出。

2）修改kernel/riscv.h，新增PTE_COW标志位，参考riscv对PTE标志位定义，第9-10位为预留标志位，所以增加的为1L<<8

3）修改kernel/vm.c的uvmcopy()，让进程fork时，不赋值物理页，而是child进程页表指向parent进程的物理页，但标记要将parent和child的PTE都清除PTE_W标志位，并添加COW标志位，表明两个PTE指向一个物理页。

4）修改kernel/vm.c的mappages()，在页表与物理页绑定时，增加refNum对应元素计数。

5）修改kernel/vm.c的uvmunmap()，在页表与物理页解绑时，减少refNum对应元素计数，当refNum==1即仅kernel pagetable持有时，释放内存。

6）将kernel/vm.c中walk()定义在defs.h中。

7）修改kernel/trap.c的usertrap()，引入refNum，在发生page fault时，若该虚拟地址关联的PTE，表明关联的物理页是一个COW页，则新申请一个物理页，让此虚拟地址指向新物理页，并修改refNum计数。

8）修改kernel/vm.c的copyout()，同kernel/trap.c的usertrap。

#### 问题

遇到的问题很多

主要是由于对锁的设置不太明确所导致的，在对引用计数进行读写时注意锁的设置。

在mappages函数中会触发一个remap的panic，这里只要注释掉就行了，因为COW就是要对页面进行重新映射的。

还有要注意对引用数组的计数范围：

不然会越界产生bug：reference_count[((uint64)r - KERNBASE)/ PGSIZE] = 1; 

# Lab7

### 前置知识

**多路复用**

Xv6通过在两种情况下将每个CPU从一个进程切换到另一个进程来实现多路复用。第一：当进程等待设备或管道I/O完成，或等待子进程退出，或在sleep系统调用中等待时，xv6使用睡眠（sleep）和唤醒（wakeup）机制切换。第二：xv6周期性地强制切换以处理长时间计算而不睡眠的进程。这种多路复用产生了每个进程都有自己的CPU的错觉，就像xv6使用内存分配器和硬件页表来产生每个进程都有自己内存的错觉一样。

**关键问题**

上下文切换

进程透明的方式实现

使用一个锁方案避免争用

进程退出时必须释放进程的内存以及其他资源，但它不能自己完成所有这一切，因为（例如）它不能在仍然使用自己内核栈的情况下释放它

多核机器的每个核心必须记住它正在执行哪个进程，以便系统调用正确影响对应进程的内核状态。

sleep允许一个进程放弃CPU，wakeup允许另一个进程唤醒第一个进程。需要小心避免导致唤醒通知丢失的竞争。

**锁**

Xv6将自旋锁表示为struct spinlock (kernel/spinlock.h:2)。结构体中的重要字段是locked。当锁可用时为零，当它被持有时为非零。

多核处理器通常提供实现第5行和第6行的原子版本的指令。在RISC-V上，这条指令是amoswap r, a。amoswap读取内存地址a处的值，将寄存器r的内容写入该地址，并将其读取的值放入r中。也就是说，它交换寄存器和指定内存地址的内容。它原子地执行这个指令序列，使用特殊的硬件来防止任何其他CPU在读取和写入之间使用内存地址。

使用锁的一个困难部分是决定要使用多少锁，以及每个锁应该保护哪些数据和不变量。有几个基本原则。首先，任何时候可以被一个CPU写入，同时又可以被另一个CPU读写的变量，都应该使用锁来防止两个操作重叠。其次，请记住锁保护不变量（invariants）：如果一个不变量涉及多个内存位置，通常所有这些位置都需要由一个锁来保护，以确保不变量不被改变。



实现一个用户态的线程库；尝试使用线程来为程序提速；并且尝试实现一个同步屏障

### 用户态线程

#### 前置知识

补全 uthread.c，完成用户态线程功能的实现。

这里的线程相比现代操作系统中的线程而言，更接近一些语言中的“协程”。原因是这里的“线程”是完全用户态实现的，多个线程也只能运行在一个 CPU 上，并且没有时钟中断来强制执行调度，需要线程函数本身在合适的时候主动 yield 释放 CPU。这样实现起来的线程并不对线程函数透明，所以比起操作系统的线程而言更接近协程。

内核调度器无论是通过时钟中断进入（usertrap），还是线程自己主动放弃 CPU（sleep、exit），最终都会调用到 yield 进一步调用 swtch。 由于上下文切换永远都发生在函数调用的边界（swtch 调用的边界），恢复执行相当于是 swtch 的返回过程，会从堆栈中恢复 caller-saved 的寄存器， 所以用于保存上下文的 context 结构体只需保存 callee-saved 寄存器，以及 返回地址 ra、栈指针 sp 即可。恢复后执行到哪里是通过 ra 寄存器来决定的（swtch 末尾的 ret 转跳到 ra）

而 trapframe 则不同，一个中断可能在任何地方发生，不仅仅是函数调用边界，也有可能在函数执行中途，所以恢复的时候需要靠 pc 寄存器来定位。 并且由于切换位置不一定是函数调用边界，所以几乎所有的寄存器都要保存（无论 caller-saved 还是 callee-saved），才能保证正确的恢复执行。 这也是内核代码中 struct trapframe 中保存的寄存器比 struct context 多得多的原因。

另外一个，无论是程序主动 sleep，还是时钟中断，都是通过 trampoline 跳转到内核态 usertrap（保存 trapframe），然后再到达 swtch 保存上下文的。 恢复上下文都是恢复到 swtch 返回前（依然是内核态），然后返回跳转回 usertrap，再继续运行直到 usertrapret 跳转到 trampoline 读取 trapframe，并返回用户态。 也就是上下文恢复并不是直接恢复到用户态，而是恢复到内核态 swtch 刚执行完的状态。负责恢复用户态执行流的其实是 trampoline 以及 trapframe。



#### 思路

这个实验其实相当于在用户态重新实现一遍 xv6 kernel 中的 scheduler() 和 swtch() 的功能，所以大多数代码都是可以借鉴的。

uthread_switch.S 中需要实现上下文切换的代码，这里借鉴 swtch.S

首先定义一个context结构体保存线程上下文，并加入到thread结构体中。在上下文中只需要保存被调用者保存的寄存器，即sp和s0-s11，ra用来保存线程的返回地址，类似于进程中的pc

之后在thread_create中加入初始化代码，使ra指向线程的入口函数，sp和fp指向栈底。注意栈底应该是t->stack[STACK_SIZE - 1]，因为栈是从高地址向低地址增长的。

最后实现thread_switch函数并在thread_schedule中通过thread_switch((uint64)&t->context, (uint64)&next_thread->context);调用即可。thread_switch需要对上下文进行保护和恢复，并通过设置ra寄存器和ret指令来恢复下一个线程的执行。

#### 实现流程

修改user/uthread.c，新增头文件引用

修改user/uthread.c，struct thread增加成员struct context，用于线程切换时保存/恢复寄存器信息。此处struct context即kernel/proc.h中定义的struct context。

修改user/uthread.c，修改thread_switch函数定义

修改user/uthread.c，修改thread_create函数，修改ra保证初次切换到线程时从何处执行，修改sp保证栈指针位于栈顶（地址最高）

修改user/uthread.c，修改thread_schedule，调用thread_switch调用，保存当前线程上下文，恢复新线程上下文

修改user/uthread_switch.S，实现保存/恢复寄存器

### 线程使用

分析并解决一个哈希表操作的例子内，由于 race-condition 导致的数据丢失的问题，通过对哈希表的并行操作来练习锁的使用。

#### 思路

设定了五个散列桶，根据键除以5的余数决定插入到哪一个散列桶中，插入方法是头插法。

假设现在有两个线程T1和T2，两个线程都走到put函数，且假设两个线程中key=NBUCKET相等，即要插入同一个散列桶中。两个线程同时调用insert(key, value, &table[i], table[i])，insert是通过头插法实现的。如果先insert的线程还未返回另一个线程就开始insert，那么前面的数据会被覆盖。

因此只需要对插入操作上锁即可。

直接在put和get上操作加锁保证安全

#### 优化思路

从粗粒度的锁变成细粒度的锁

加锁后多线程的性能变得比单线程还要低了，虽然不会出现数据丢失，但是失去了多线程并行计算的意义：提升性能

由于哈希表中，不同的 bucket 是互不影响的，一个 bucket 处于修改未完全的状态并不影响 put 和 get 对其他 bucket 的操作，所以实际上只需要确保两个线程不会同时操作同一个 bucket 即可，并不需要确保不会同时操作整个哈希表。

所以可以将加锁的粒度，从整个哈希表一个锁降低到每个 bucket 一个锁。

在主函数中使用循环来实现

#### 实现流程

修改notxv6/ph.c，定义互斥锁

修改notxv6/ph.c，定义互斥锁，每个bucket一个锁

修改notxv6/ph.c，初始化互斥锁

修改notxv6/ph.c，对put操作加锁，在put中对数组某个bucket加锁



### 同步屏障

利用 pthread 提供的条件变量方法，实现同步屏障机制

#### 前置知识

同步屏障机制

设置一个屏障点，所有线程到达这个屏障点之才能够继续执行。

#### 思路

线程进入同步屏障 barrier 时，将已进入屏障的线程数量增加 1，然后再判断是否已经达到总线程数。

如果未达到，则进入睡眠，等待其他线程。

如果已经达到，则唤醒所有在 barrier 中等待的线程，所有线程继续执行；屏障轮数 + 1；

「将已进入屏障的线程数量增加 1，然后再判断是否已经达到总线程数」这一步并不是原子操作，并且这一步和后面的两种情况中的操作「睡眠」和「唤醒」之间也不是原子的，如果在这里发生 race-condition，则会导致出现 「lost wake-up 问题」线程 1 即将睡眠前，线程 2 调用了唤醒，然后线程 1 才进入睡眠，导致线程 1 本该被唤醒而没被唤醒。

解决方法是，「屏障的线程数量增加 1；判断是否已经达到总线程数；进入睡眠」这三步必须原子。所以使用一个互斥锁 barrier_mutex 来保护这一部分代码。pthread_cond_wait 会在进入睡眠的时候原子性的释放 barrier_mutex，从而允许后续线程进入 barrier，防止死锁

#### 实现流程

修改notxv6/barrier.c，实现barrier()方法，

```c++
static void barrier()
{
  pthread_mutex_lock(&bstate.barrier_mutex);//加锁
    
  if(++bstate.nthread < nthread) {//判断barrier的线程数量
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);//等待
      //调用pthread_cond_wait时，mutex必须已经持有
  } else {//唤醒
    bstate.nthread = 0;
    bstate.round++;
    pthread_cond_broadcast(&bstate.barrier_cond);//线程执行
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);//释放锁
}
```



# Lab8

对XV6内部的锁进行优化，减少锁争用，提高系统并行性

### Memory allocator

通过拆分 kmem 中的空闲内存链表，降低 kalloc 实现中的 kmem 锁竞争

#### 前置知识

kalloc 原本的实现中，使用 freelist 链表，将空闲物理页本身直接用作链表项（这样可以不使用额外空间）连接成一个链表，在分配的时候，将物理页从链表中移除，回收时将物理页放回链表中，无论是分配物理页或释放物理页，都需要修改 freelist 链表。由于修改是多步操作，为了保持多线程一致性，必须加锁。但这样的设计也使得多线程无法并发申请内存，限制了并发效率。

##### 不要过早优化

这里学习到了一个思路，不要过早优化，要先经过profile，使用kalloctest来查看状态，会显示出五种竞争频繁的锁，第一个就是kmem

在优化时，要先实现功能，再确定问题，然后进一步进行优化



锁竞争优化一般有几个思路：

只在必须共享的时候共享（对应为将资源从 CPU 共享拆分为每个 CPU 独立）；
必须共享时，尽量减少在关键区中停留的时间（对应“大锁化小锁”，降低锁的粒度）。lab7.2



本次采用第一个思路：

将共享资源变为不共享资源，只在需要时共享。



#### 思路

该 lab 的实验目标，即是为每个 CPU 分配独立的 freelist，这样多个 CPU 并发分配物理页就不再会互相排斥了，提高了并行性。

但由于在一个 CPU freelist 中空闲页不足的情况下，仍需要从其他 CPU 的 freelist 中“偷”内存页，所以一个 CPU 的 freelist 并不是只会被其对应 CPU 访问，还可能在“偷”内存页的时候被其他 CPU 访问，故仍然需要使用单独的锁来保护每个 CPU 的 freelist。

但一个 CPU freelist 中空闲页不足的情况相对来说是比较稀有的，所以总体性能依然比单独 kmem 大锁要快。

在最佳情况下，也就是没有发生跨 CPU “偷”页的情况下，这些小锁不会发生任何锁竞争。

使用快慢指针来找到链表的中点，之后将一半的页面移动到当前核心的链表上。

在现实场景中，最好进行测量后选取合适的数值，尽量使得“偷”页频率低。

#### 实现流程

修改kernel/kalloc.c，修改原有结构体声明kmem，定义结构体数组kmemArray[NCPU]，每个cpu对应一个。

修改kernel/kalloc.c，修改kinit方法，对NCPU个kmem结构体初始化，以kmem开头，函数只会被一个CPU调用，freerange调用kfree将所有空闲内存挂在该CPU的空闲列表上

修改kfree，使用cpuid()和它返回的结果时必须关中断

修改kernel/kalloc.c，修改kalloc方法，当前CPU freelist为空时，从其他CPU freelist处获取



###  Buffer cache

多个进程同时使用文件系统的时候，bcache.lock 上会发生严重的锁竞争。bcache.lock 锁用于保护磁盘区块缓存，在原本的设计中，由于该锁的存在，多个进程不能同时操作（申请、释放）磁盘缓存。

将缓冲区的分配与回收并行化以提高效率。

因为不像 kalloc 中一个物理页分配后就只归单个进程所管，bcache 中的区块缓存是会被多个进程（进一步地，被多个 CPU）共享的（由于多个进程可以同时访问同一个区块）。所以 kmem 中为每个 CPU 预先分割一部分专属的页的方法在这里是行不通的。

这里采用第二种思路

属于必须共享的情况，尽量减少在关键区中的停留时间，降低锁的粒度，使用更加精细的锁

原版 xv6 的设计中，使用双向链表存储所有的区块缓存，每次尝试获取一个区块 blockno 的时候，会遍历链表，如果目标区块已经存在缓存中则直接返回，如果不存在则选取一个最近最久未使用的，且引用计数为 0 的 buf 块作为其区块缓存，并返回。

新的改进方案，可以建立一个从 blockno 到 buf 的哈希表，并为每个桶单独加锁。这样，仅有在两个进程同时访问的区块同时哈希到同一个桶的时候，才会发生锁竞争。当桶中的空闲 buf 不足的时候，从其他的桶中获取 buf。

初始的XV6磁盘缓冲区中是使用一个LRU链表来维护的，而这就导致了每次获取、释放缓冲区时就要对整个链表加锁，也就是说缓冲区的操作是完全串行进行的。

为了提高并行性能，用哈希表来代替链表，这样每次获取和释放的时候，都只需要对哈希表的一个桶进行加锁，桶之间的操作就可以并行进行。只有当需要对缓冲区进行驱逐替换时，才需要对整个哈希表加锁来查找要替换的块。

#### 思路

使用哈希表就不能使用链表来维护LRU信息，因此需要在buf结构体中添加timestamp域来记录释放的事件，同时prev域也不再需要。

在brelse函数中对timestamp域进行维护，同时将链表的锁替换为桶级锁

在binit函数中对哈希表进行初始化，将bcache.buf[NBUF]中的块平均分配给每个桶，记得设置b->blockno使块的hash与桶相对应，后面需要根据块来查找对应的桶。

之后就是重点bget函数，首先在对应的桶当中查找当前块是否被缓存，如果被缓存就直接返回；如果没有被缓存的话，就需要查找一个块并将其逐出替换。

先在当前桶当中查找，当前桶没有查找到再去全局数组中查找，这样的话如果当前桶中有空闲块就可以避免全局锁。

在全局数组中查找时，要先加上表级锁，当找到一个块之后，就可以根据块的信息查找到对应的桶，之后再对该桶加锁，将块从桶的链表上取下来，释放锁，最后再加到当前桶的链表上去。

这里有个小问题就是全局数组中找到一个块之后，到对该桶加上锁之间有一个窗口，可能就在这个窗口里面这个块就被那个桶对应的本地查找阶段用掉了。因此，需要在加上锁之后判断是否被用了，如果被用了就要重新查找。

最后将bpin和bunpin的锁替换为桶级锁就行了。

#### 实现流程

修改kernel/buf.h，新增tick，作为时间标记

修改kernel/bio.c，新增NBUC，作为hash表bucket数量

修改kernel/bio.c，修改bcache，去除head

修改kernel/bio.c，定义结构体bMem，新增结构体数组bMem作为哈希表

修改kernel/bio.c，在binit方法中新增哈希表中每个bucket lock初始化

修改kernel/bio.c，新增replaceBuffer方法，用于LRU buffer赋值

修改kernel/bio.c中的bget方法

修改kernel/bio.c中的brelse方法

修改kernel/bio.c中的bpin、bunpin方法



# Lab9

为 xv6 的文件系统添加大文件以及符号链接支持

### 前置知识

文件系统可以说是XV6中最复杂的部分，整个文件系统包括了七层：文件描述符，路径名，目录，inode，日志，缓冲区，磁盘。

文件描述符类似于Linux，将文件、管道、设备、套接字等都抽象为文件描述符，从而可以使用read和write系统调用对其进行读写。XV6的read和write是使用if-else来对描述符类型进行判断，选择对应的底层函数；而在Linux中，则是使用函数指针直接指向对应的底层函数，避免进行多次判断。

路径名则提供了根据路径名从目录系统中查找文件的功能。在路径查找过程中需要避免可能会出现的死锁，例如路径名中包含..。

目录层类似于文件，目录文件的内部会保存该目录的目录项struct dirent，其中包含了文件名和对应的inode号。在XV中目录查找是使用遍历目录项数组来依次比较，时间复杂度为O(n)；而在NTFS、ZFS等文件系统中，会使用磁盘平衡树来组织目录项，使目录查找的复杂度降低为O(lgn)。

索引节点inode层为文件在磁盘上的组织，在磁盘中会有一块区域用于保存inode信息，包括文件类型、大小、链接数以及文件每个块对应的磁盘块号。通过路径从目录系统中查找到对应的inode号，之后就可以从磁盘上读取对应的inode信息，之后就可以根据偏移量查找对应的磁盘块号，最后对其进行读写。

日志层提供了事务以及故障恢复的功能，当有多个磁盘操作必须原子完成时就要用到事务（如删除文件时要从目录中删除文件，删除文件对应的inode，对空闲块bitmap进行修改等）。日志先将操作写到磁盘的日志区上，写入完成后再写入commit，最后再将所有操作真正写到磁盘上去。当在写入commit之前发生故障，就不需要进行操作，因为事务没有被提交；当在写入commit之后发生故障，就将日志区的日志全部重写一遍，保证事务被正确提交。

缓冲区则提供了磁盘块缓存，同时保证一个磁盘块在缓冲区中只有一个，使得同一时间只能有一个线程对同一个块进行操作，避免读到的数据不一致。

### Large files

#### 思路

这个 part 是让我们让最大文件大小从 268 blocks（12+256）直接块表和一级块表，增加到 65803 blocks（ 11 + 256 + 256 * 256），其实也就是增加一个二层映射块，并减少一个直接映射块。所以我们得去改动 dinode， 这个在file.h 和 fs.h做些更改就行。改完之后 dinode 结构应该如下图所示（红色部分代表改动）

![image-20240315145533647](C:\Users\yuyu\AppData\Roaming\Typora\typora-user-images\image-20240315145533647.png)

首先修改对应的宏以及inode定义，将 NDIRECT 直接索引的盘块号减少 1，腾出 inode 中的空间来存储二级索引的索引表盘块号。

之后修改bmap函数，使其支持二级块表，其实就是重复一次块表的查询过程。

最后修改itrunc函数使其能够释放二级块表对应的块，主要就是注意一下brelse的调用就行了，仿照一级块表的处理就行了。

#### 实现流程

修改kernel/fs.h。

修改kernel/file.h

修改kernel/fs.c的bmap和 itrunc函数，使其支持二级块表，其实就是重复一次块表的查询过程，使其能够释放二级块表对应的块，主要就是注意一下brelse的调用就行了，仿照一级块表的处理就行了。

### Symbolic links

#### 思路

写一个符号链接系统调用：symlink(char *target, char *path)；可以参考 ln -s 的功能

符号链接就是在文件中保存指向文件的路径名，在打开文件的时候根据保存的路径名再去查找实际文件。与符号链接相反的就是硬链接，硬链接是将文件的`inode`号指向目标文件的`inode`，并将引用计数加一。

symlink的系统调用实现起来也很简单，就是创建一个inode，设置类型为T_SYMLINK，然后向这个inode中写入目标文件的路径就行了。

最后在sys_open中添加对符号链接的处理就行了，当模式不是O_NOFOLLOW的时候就对符号链接进行循环处理，直到找到真正的文件，如果循环超过了一定的次数（10），就说明可能发生了循环链接，就返回-1。这里主要就是要注意namei函数不会对ip上锁，需要使用ilock来上锁，而create则会上锁。

#### 实现流程

修改kernel/syscall.h

修改kernel/usys.pl

修改kernel/user.h

修改kernel/syscall.c

修改kernel/stat.h

修改kernel/fcntl.h

修改kernel/sysfile.c，实现sys_symlink函数

修改kernel/sysfile.c，修改sys_open函数