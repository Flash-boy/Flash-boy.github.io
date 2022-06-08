+++
author = "Hugo Authors"
date = "2022-06-05"
title = "Lab3 Page tables"
tags = [
    "C",
    "Course",
    "MIT"
    
]
categories = [
    "Operator System"
]
+++
# [Page tables](https://pdos.csail.mit.edu/6.828/2020/labs/pgtbl.html) 
## 实验目标
___
本实验主要了解xv6虚拟内存的底层实现，xv6能够运行的机制是内核采用了恒等映射，即把内核的虚拟地址和物理地址在内核页表建立的是恒等映射，同时xv6采用的是三级页表。这个实验主要是弄懂xv6的内核布局以及页表机制，主要的源代码是kernel/kalloc.c, kernel/vm.c, kernel/memlayout.h。

## 实验详解
___
### Print a page table (easy)
主要是实现一个pagetable的层级打印，这个比较简单，只要弄得xv6的pagetable三级页表结构，配合kernel/riscv.h里提供的页表宏很容易得到答案

```c
// kernel/vm.c

// helper funtion for vmprint()
static void vmprinthelper(pagetable_t pagetable, int level){
    static char* prefix[] = {"..", ".. ..", ".. .. .."};
    if(level > 2) return;

    for(int i = 0; i < 512; i++){
        pte_t pte = pagetable[i];
        if(pte & PTE_V){
            // this PTE points to a lower-level page table.
            uint64 child = PTE2PA(pte);
            printf("%s%d: pte %p pa %p\n", prefix[level], i, pte, child);
            vmprinthelper((pagetable_t)child, level + 1);
        }
    }

}
// printf the valid pte of pagetable 
// pagetable: the physical pagetable address need to print
void vmprint(pagetable_t pagetable){
    printf("page table %p\n", pagetable);
    vmprinthelper(pagetable, 0);
}
```
效果如图     
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab3-1.png)

### A kernel page table per process (hard)
原先版本的xv6有一个全局的内核页表，采用恒等映射，同时每个进程在用户空间有一个页表。因此，在进程由用户态和内核态切换的过程中，需要切换页表，这会导致cpu的L1Cache失效，降低效率。同时也会造成如果想要将用户态的数据复制到内核态，首先需要主动遍历用户页表得到数据的物理地址，然后因为在内核物理地址和虚拟地址是恒等映射，在利用数据的物理地址进行复制。如果将内核页表和用户页表
合为一个页表，但不同页表项具有不同的权限，这样不仅能解决L1Cache失效的问题，同时也能将用户态数据复制到内核态不需要通过一层转换间接复制来完成数据的复制。    

本实验是为每个进程构造一个内核页表的副本，核心就是参考内核页表的生成，构造一个类似的页表保存在struct proc中。   

首先在struct proc中新增一个字段记录内核页表的副本  
```c
// kernel/proc.h
struct proc{
    // ...
    pagetable_t kernel_pagetable;
};
```

kvminit()函数是对内核页表的初始化，仿照该方法定义一个kvmcreat()函数来建立一个内核页表的映射
```c
// create pagetable of kernel
static void create_kvmmap(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm){
    if(mappages(pagetable, va, sz, pa, perm) != 0)
        panic("create_kvmmap");
}
// 方便后续初始化struct proc中的kernel_pagetable 
pagetable_t kvmcreat(){
  pagetable_t pagetable = (pagetable_t) kalloc();
  memset(pagetable, 0, PGSIZE);
  
  // uart registers
  create_kvmmap(pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  create_kvmmap(pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  create_kvmmap(pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  create_kvmmap(pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  create_kvmmap(pagetable, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  create_kvmmap(pagetable, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);


  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  create_kvmmap(pagetable, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);

  return pagetable;
}
/*
 * create a direct-map page table for the kernel.
 */
void 
kvminit()
{
  kernel_pagetable = kvmcreat();
  //create_kvmmap(kernel_pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);
}
```
全局内核页表在procinit函数中还为每个进程映射了内核栈，这里因为采用的是struct proc中的内核页表，因此需要将这部分工作
转移到allocproc函数中。
```c
// kernel/proc.c

void
procinit(void)
{
  struct proc *p;
  
  initlock(&pid_lock, "nextpid");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");

      // Allocate a page for the process's kernel stack.
      // Map it high in memory, followed by an invalid
      // guard page.
      // 注释掉这部分代码，在allocproc中实现
      /* do this in allocproc()
      char *pa = kalloc();
      if(pa == 0)
        panic("kalloc");
      uint64 va = KSTACK((int) (p - proc));
      kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
      p->kstack = va;
      */
  }
  kvminithart();
}

static struct proc*
allocproc(void)
{
 // ...
found:
  // ...

  // Set struct proc field -> kernel_pagetable;
  // 创建内核页表
  p->kernel_pagetable = kvmcreat();
  // 映射进程的内核栈
  char *pa = kalloc();
  if(pa == 0)
      panic("kalloc");
  uint64 va = TRAMPOLINE - 2 * PGSIZE;
  if(mappages(p->kernel_pagetable, va, PGSIZE, (uint64)pa, PTE_R | PTE_W) != 0)
      panic("allocproc");
  p->kstack = va;

  // ...
  return p;
}
```

完成以上部分就在struct proc中创建了一个和原先版本一样的内核页表副本。接着就是在scheduler函数中切换页表时使用
struct proc中的内核页表
```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();
    
    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;

        // 在运行进程过程中使用进程中的内核页表
        w_satp(MAKE_SATP(p->kernel_pagetable));
        sfence_vma();

        swtch(&c->context, &p->context);
        kvminithart(); // 切换回内核态使用全局内核页表
        
        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        found = 1;
      }
      release(&p->lock);
    }
#if !defined (LAB_FS)
    if(found == 0) {
      intr_on();
      asm volatile("wfi");
    }
#else
    ;
#endif
  }
}
```
最后释放进程时不要忘记释放对应struct proc中的内核页表
```c
// kernel/proc.c

static void
freeproc(struct proc *p)
{
  // ...
  if(p->kernel_pagetable){
    uint64 va = TRAMPOLINE - 2 * PGSIZE; 
    uvmunmap(p->kernel_pagetable, va, 1, 1);
    proc_freewalk(p->kernel_pagetable);
    p->kernel_pagetable = 0;
  }
  // ...
}

// kernel/vm.c
// 释放struct proc中的kernel_pagetable
void proc_freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    pagetable[i] = 0;
    if(pte & PTE_V){
      if((pte & (PTE_W | PTE_R | PTE_X)) == 0){
        uint64 child = PTE2PA(pte);
        proc_freewalk((pagetable_t)child);
      }
    }
  }
  kfree((void*)pagetable);
}
```
实验效果如图     
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab3-2.png)

### Simplify coypin/copyinstr (hard) 
本实验就是将用户进程映射到struct proc中的kernel_pagetable这，核心代码就是在任何时候用户空间映射改变的时候
修改kernel_pagetable    

首先userinit是初始化进程，后续所有用户进程都是基于该进程的fork。需要在该进程的kernel_pagetable这创建映射
```c
// kernel/proc.c
// Set up first user process.
void
userinit(void)
{
  struct proc *p;
  pte_t* pte, *kernelpte;

  p = allocproc();
  initproc = p;
  
  // allocate one user page and copy init's instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  // 映射用户进程代码到kernel_pagetable
  pte = walk(p->pagetable, 0, 0);
  kernelpte = walk(p->kernel_pagetable, 0, 1);
  *kernelpte = (*pte) & ~PTE_U;

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  release(&p->lock);
}
``` 
接着对于修改了用户进程映射的函数都需要做类似的修改包括，fork, exec, sbrk。
```c
// kernel/proc.c
int
fork(void)
{
  // ...

  // map user addr to proc kernel_pagetable; 
  for(int j = 0; j < p->sz; j += PGSIZE){
      pte = walk(np->pagetable, j, 0);
      kernelpte = walk(np->kernel_pagetable, j, 1);
      *kernelpte = (*pte) & ~PTE_U;
  }
  // ...
}

// kernel/exec.c
int
exec(char *path, char **argv)
{
  // ...
  // Load program into memory.
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    uint64 sz1;
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    // 用户进程空间查过PLIC直接按出错处理
    if(sz1 >= PLIC)
      goto bad;

    sz = sz1;
    if(ph.vaddr % PGSIZE != 0)
      goto bad;
    if(loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }

  // 取消旧映射，创建新映射
  uvmunmap(p->kernel_pagetable, 0, PGROUNDUP(oldsz) / PGSIZE, 0);
  for(int j = 0; j < sz; ++j){
      pte = walk(pagetable, j, 0);
      kernelpte = walk(p->kernel_pagetable, j, 1);
      *kernelpte = (*pte) & ~PTE_U;
  }
  // ...
}

// kernel/sysproc.c
uint64
sys_sbrk(void)
{
  int addr;
  int n;
  struct proc* p = myproc();
  pte_t* pte, *kernelpte;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  if(growproc(n) < 0)
    return -1;
  // 添加映射到kernel_pagetable
  if(n > 0){
    for(int j = addr; j < addr + n; j += PGSIZE){
        pte = walk(p->pagetable, j, 0);
        kernelpte = walk(p->kernel_pagetable, j, 1);
        *kernelpte = (*pte) & ~PTE_U;
    }
  }else{
    for(int j = addr - PGSIZE; j >= addr + n; j -= PGSIZE){
        uvmunmap(p->kernel_pagetable, j, 1, 0);
    }
  }
  return addr;
}
``` 
效果如图    
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab3-3.png)

## 实验总结
___
该实验是比较难的一个实验，需要完全明白xv6的页表是如何实现的，同时要对用户进程分配释放等比较清楚，需要看到大部分kernel/proc.c的代码，以及虚拟内存相关的代码，花了很长时间去完成该lab。总之搞懂每个函数的作用，按照hint去做，也能够完成

