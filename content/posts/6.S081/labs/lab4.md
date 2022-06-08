+++
author = "Hugo Authors"
date = "2022-06-05"
title = "Lab4 Traps"
tags = [
    "C",
    "Course",
    "MIT"
    
]
categories = [
    "Operator System"
]
+++
# [Traps](https://pdos.csail.mit.edu/6.828/2020/labs/traps.html) 
## 实验目标
___

系统调用，异常和中断是应用程序需要响应程序或者外部设备必须做出的动作，操作系统采用了trap的方式处理上述三种情况。trap的方式提供了一种安全，可靠的方式对外部输入或者程序系统调用进行响应处理。trap的核心就是利用一段称为trampoline的代码作为"跳板"和利用trapframe保存应用程序运行状态。

## 实验详解
___
### Backtrace (moderate)
程序的栈帧标示着程序的函数调用栈，实现一个函数来打印当前函数的调用栈帧，核心就是利用r_fp函数读取函数的帧指针，一个典型的栈帧是先压入函数返回地址，然后压入(prev)栈帧。如图所示为一个典型的栈帧。    

![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab4-1.png) 

因此当前的fp-8保存着函数返回地址，fp-16保存着上一个栈帧fp地址。因此backtrace函数如下所示   

```c
// kernel/printf.c
// backtrace for debug
void backtrace(){
    printf("backtrace:\n");
    uint64 fp = r_fp();
    // xv6默认用户栈最大为4096KB
    while(PGROUNDUP(fp) - PGROUNDDOWN(fp) == PGSIZE){
        uint64 ra = *(uint64*)((char*)fp - 8);
        printf("%p\n", ra);
        fp = *(uint64*)((char*)fp - 16);
    }
}
```  
运行效果如图      
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab4-2.png) 

###  Alarm (hard)
alarm是要你提供一个系统调用实现，每当进程调用该系统调用时，提供两个参数n和fn, 每经过n个系统cpu时间片，执行一次回调函数fn。   

核心思路是理解trap机制，当有一个系统调用通过ecall进入trampoline代码的uservec部分，该段代码保存寄存器状态到p->trapframe,然后进入kernel/trap.c中的usertrap函数处理，该函数根据进入trap的不同原因调用不同的函数处理。执行完毕后，调用kernel/trap.c中的usertrapret函数，该函数会设置页表和一些p->trapframe的默认项以供下次trap使用，以及调用trampoline.S中的userret函数，该函数用p->trapframe的值重新恢复寄存器状态。

因此，一个核心的思路是将alarm系统调用的参数保存在struct proc中，在每次trap过程中，如果是定时器中断进入trap则需要将对应的ticks加1，一旦达到要求，需要在trap处理函数中将返回值即p->trapframe->epc值设置为该fn函数。然后系统会调用该fn函数，但如何返回到一开始的函数调用处呢？答案是利用sigreturn函数，在trap处理函数中保存当前的trapframe到一个新的trapframe中，在sigreturn系统调用中，将trapframe保存为原先的值，因此，sigalarm和sigreturn需要配对使用。

```c
// kernel/sysproc.c
uint64 sys_sigalarm(void){
    int alarm_interval;
    uint64 fn;
    if(argint(0, &alarm_interval) < 0)
        return -1;
    if(argaddr(1, &fn) < 0)
        return -1;
    struct proc* p = myproc();
    if(alarm_interval < 0) alarm_interval = 0;
    // 保存参数
    p->alarm_interval = alarm_interval; 
    p->handler = fn;
    p->ticks = 0;
    return 0;
}

uint64 sys_sigreturn(void){
    struct proc* p = myproc();
    *p->trapframe = *p->savedtrapframe;
    p->ticks = 0;
    return 0;
}

// kernel/trap.c
void
usertrap(void)
{
  // ...
  
  if(r_scause() == 8){
    // system call

    // ...
  } else if((which_dev = devintr()) != 0){
      if(which_dev == 2){
          if(p->alarm_interval){
              if(p->ticks == p->alarm_interval){
                  *p->savedtrapframe = *p->trapframe; // 保存原先trapframe
                  p->trapframe->epc = p->handler; // 设置跳转函数，trap返回指向该函数
              }
              // 运行时间片+1
              p->ticks++;
          }
      }
    // ok
  } 
  // ...
}
```  

运行效果图    
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab4-3.png)

## 实验总结
___
该实验的backtrace并不难，主要利用xv6的栈帧读取函数返回地址，alarm实验很有趣，类似于向系统提供了一个定时任务，主要是利用了trap机制，更改了函数的返回地址，在跳转到该回调函数之前保存了原先的trapframe，配合sigreturn使用恢复到一开始的函数调用处，很巧妙。最后给出总体测试结果图。   

![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab4-4.png)
