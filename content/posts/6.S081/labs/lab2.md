+++
author = "Hugo Authors"
date = "2022-06-05"
title = "Lab2 System calls"
tags = [
    "C",
    "Course",
    "MIT"
    
]
categories = [
    "Operator System"
]
+++
# [System calls](https://pdos.csail.mit.edu/6.828/2020/labs/syscall.html)

## 实验目标
___

本实验主要是去了解系统调用是如何实现的，xv6提供了20多个系统调用，系统调用采用的是一种trap机制，并不是向传统的用户态函数调用，这里需要通过trap机制，进入内核，执行完实际的系统调用，然后切换回用户态程序。要完成本lab，需要仔细阅读以下几个文件的源代码。

- user/user.h 用户系统调用的函数原型，以及一些用户函数原型。  
- user/usys.pl 用户系统调用函数原型实现的汇编脚本。 
- kernel/syscall.h 系统调用编号
- kernel/syscall.c 内核系统调用接口实现
- kernel/sysproc.c 内核实际系统调用函数实现

总结一下，系统调用实际过程和步骤  
1. 用户调用user/user.h中声明的系统调用函数
2. user/usys.pl中的汇编代码表示一个系统调用只是将系统调用编号放进a7寄存器之，然后调用ecall指令进入内核的trap中
3. 内核在trap中处理，运行kernel/syscall.c中的syscall函数
4. syscall函数通过a7寄存器的系统调用编号找到对应的函数(实现在kernel/sysproc.c中)
5. 运行kernel/sysproc.c中对应的系统调用函数  

## 实验详解
___
### System call tracing (moderate) 
实验要求实现trace函数，通过trace函数传入一个mask,可以追踪程序在运行过程中调用过哪些对应系统调用，。根据hint的提示
一个很自然的想法是在运行kernel/syscall.c中的syacall中处理，当我们调用系统函数时，会走到该函数，
通过判断系统调用是否为我们需要追踪的系统调用，如果是，输出相关消息，如何保存我们需要追踪的系统调用mask，根据提示在struct proc中进行保存。解决方法如下。

在user/user.h中声明trace系统调用和原型，以及在user/usys.pl脚本中添加对应的实现
```c
// user/user.h
int trace(int);

// user/usys.pl
entry("trace");
```
添加系统调用编号到kernel/syscall.h中
```c
// kernel/syscall.h
#define SYS_trace 22
```
修改kernel/syscall.c中的函数指针数组，以及定义函数名字符串数组，以及在
```c
// kernel/syscall.c
extern uint64 sys_trace(void);

static uint64 (*syscalls[])(void) = {
// ...
[SYS_trace]   sys_trace, 
};

static char* syscallsStr[] = {
// ...
[SYS_trace]   "syscall trace",
};

void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    // 如果系统调用编号在掩码中打印追踪信息
    // a0寄存器保存着系统调用返回值
    if(p->trace_mask & (1 << num)){
        printf("%d: %s -> %d\n", p->pid, syscallsStr[num], p->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

在kernel/proc.h中添加字段trace_mask
```c
// kernel/proc.h
struct proc{
  // ...
  int trace_mask;  
};
```
在fork函数中需要复制该字段的值到子进程
```c
// kernel/proc.c
int fork(void){
// ...
np->trace_mask = p->trace_mask;
// ...
}
```

最后实现kernel/sysproc.c中实现sys_trace函数，初始化trace_mask字段的值
```c
// kernel/sysproc.c

// trace syscall save the mask to struct proc
uint64 sys_trace(void){
    int mask;
    // 获取掩码参数值
    if(argint(0, &mask) < 0)
        return -1;
    struct proc* p = myproc();
    p->trace_mask = mask;
    return 0;
}
```

效果如图所示     
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab2-1.png)

### Sysinfo (moderate)
这个系统调用主要的功能是收集当前系统的空闲内存和空闲进程数，有前面的基础实现起来很简单，这里只给出重要的代码实现

sys_sysinfo函数实现
```c
// kernel/sysproc.c

// sysinfo syscall 
uint64 sys_sysinfo(void){
    uint64 addr;
    if(argaddr(0, &addr) < 0)
        return -1;
    struct sysinfo sf;
    sf.freemem = freemem_size(); // 获取freemem大小
    sf.nproc = used_procnum();   // 获取
    struct proc* p = myproc();
    // 将数据复制到用户进程addr地址指向的strcut sysinfo中
    if(copyout(p->pagetable, addr, (char*)&sf, sizeof(sf)) < 0) 
        return -1;
    return 0;
}

在kalloc.c中实现freemem_size函数，
```c
// kernel/kalloc.c

struct {
  struct spinlock lock;
  struct run *freelist;
  int pagenum; // 添加字段记录空闲page数
} kmem;

void
kfree(void *pa)
{
  // ...
  acquire(&kmem.lock);
  // ...
  kmem.pagenum++; // kfree中增加空闲page数
  release(&kmem.lock);
}

void *
kalloc(void)
{
  // ...
  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r){
    kmem.freelist = r->next;
    kmem.pagenum--; //kalloc中减少空闲page数
  }
  release(&kmem.lock);
  // ...
}

// return the free mem size(bytes) 
uint64 freemem_size(void){
    uint64 n;
    acquire(&kmem.lock);
    n = kmem.pagenum * PGSIZE;     
    release(&kmem.lock);
    return n;
} 
```

在proc.c中实现used_procnum函数
```c
// kernel/proc.c
int nproc = 0; // 全局变量记录当前使用进程数
struct spinlock nproc_lock; // 锁保证nproc数据的正确性

// allocproc分配unused proc
static struct proc*
allocproc(void)
{
// ...
found:
  // ...
  acquire(&nproc_lock);
  nproc = nproc + 1; //增加进程数
  release(&nproc_lock);
  return p;
}

// freeproc释放进程
static void
freeproc(struct proc *p)
{
  // ...
  acquire(&nproc_lock);
  nproc = nproc - 1;
  release(&nproc_lock);
}

uint64 used_procnum(){
    uint64 n;
    acquire(&nproc_lock);
    n = nproc;
    release(&nproc_lock);
    return n;
}
```

运行效果图    
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab2-2.png) 

## 实验总结
___
总体来说该lab不难，主要是需要花时间去看内核源代码。整个xv6代码写的很规范，各种注释很全，最后运行 make grade可 以得到该lab的所有测试如图     
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab2-3.png)


