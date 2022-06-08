+++
author = "Hugo Authors"
date = "2022-06-05"
title = "Lab1 Utilities"
tags = [
    "C",
    "Course",
    "MIT"
    
]
categories = [
    "Operator System"
]
+++
# [Utilities](https://pdos.csail.mit.edu/6.828/2020/labs/util.html)

## 实验目标
___

本实验主要是来熟悉xv6和它的系统调用，系统调用是一类特殊的函数，是操作系统提供的一类对特殊资源的访问，像磁盘，IO等。本课程有一个核心就是理解系统调用是如何实现的，在后续课程内容会又详细介绍。在命令行中我们经常看到许多内置的命令例如 **cd**, **ls**等，这些命令就是利用系统调用封装的用户级程序。lab会要求实现几个类似的工具命令。总体来说本lab不难，主要是对xv6的熟悉。

每个实验都有相应的代码分支管理，所有在每个实验需要切换到对应分支。基本按照lab主页的提示(hint)就可以完成该实验，

## 实验详解
___
### Boot xv6 (easy)
启动xv6,运行内置的ls命令。  
很简单，只要照着做就能看到如下效果。

![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab1-1.png)   

xv6操作系统自己实验了一些常用的命令，像cat, ls, echo等。接下来的任务就是实现一些类似的命令。

### 1. sleep (easy)

sleep 要求在程序sleep一小段时间，解决方法是调用xv6系统提供的系统调用函数。这里说一下xv6源代码主要有两个文件夹 kernel/ 和 user/。kernel主要是内核代码的实现，user则是用户编写的代码，例如前面的cat, ls, echo以及我们实现的sleep等。因此按照要求编写代码 user/sleep.c代码如下

```c
// user/sleep.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char* argv[]){
    if(argc != 2){
        printf("Usage: sleep <seconds>\n");
        exit(1);
    }    
    sleep(atoi(argv[1]) * 10);
    exit(0);
}
```
user/user.h头文件包含了xv6提供的系统调用的声明，需要包含。 

程序的返回xv6中必须用exit函数。   

别忘了更新Makefile

编译运行，并输入sleep可以看到如下的 

![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab1-2.png)

### pingpong (easy)

pipe是进程通信的一种方式，pipe是一种单向的数据传输，从一端输入，从另一端输出。pingpong要求在父子进程中传输数据，并输出各自pid。核心就是利用pipe函数两次，利用fork函数创建子进程，在父子进程中注意关闭不需要的读端和写端，然后读写pipe。

```c
// user/pingpong.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char* argv[]){
    int p1[2], p2[2];
    if(pipe(p1) < 0 || pipe(p2) < 0){
        printf("Error: pipe()\n");
        exit(1);
    }
    int pid;
    if((pid = fork()) == 0){
        close(p1[1]); 
        close(p2[0]);
        char c;
        int p = getpid();

        read(p1[0], &c, 1);
        printf("%d: received ping\n", p);
        close(p1[0]);
        write(p2[1], &c, 1);
        close(p2[1]);
    }else{
        close(p1[0]);
        close(p2[1]);
        int p = getpid();
        char c;

        write(p1[1], "w", 1); 
        close(p1[1]);
        read(p2[0], &c, 1);
        printf("%d: received pong\n", p);
        close(p2[0]);
    }
    exit(0);
}
```
效果如图  
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab1-3.png)

### primes (moderate/hard)

primes程序要求并行的输出一连串自然数中是质数的数据，核心是利用pipe和fork函数，一个很巧妙的算法步骤是   

1. 从父进程读取数据，如果没有数据可读，返回；若有则读取到的第一个数一定是质数，记作n并输出。
2. 继续从父进程读取数据，判断该数是不是n的倍数，若是则忽略，若不是则写该数据到子进程。

**注意关闭不需要的读端或写端**。想象一下如果系统支持多核，是不是能够做到并行 :smile:

```c
// user/primes.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void prime(int p[2]){
    close(p[1]); //子进程关闭写端
    int n;
    // 如果没有数据可读关闭读端
    if(read(p[0], &n, sizeof(n)) == 0){
        close(p[0]);
        exit(0);
    }
    printf("prime %d\n", n);
    int num;
    int p1[2]; pipe(p1);
    int pid = fork();
    if(pid == 0){
        prime(p1);
    }else if(pid > 0){
       close(p1[0]); 
       // 从父进程读取数据，将当前不是质数倍数的传递给子进程
       while(read(p[0], &num, sizeof(num))){
           if(num % n != 0){
               write(p1[1], &num, sizeof(num));
           }
       }
       close(p); //关闭从父进程读取的读端
       close(p1[1]); //关闭向子进程写数据的写端
       wait(0);
       exit(0);
    }
    exit(1);
}
int main(int argc, char* argv[]){
    int p[2];
    pipe(p); 
    int pid = fork();
    if(pid == 0){
        // 子进程处理数据
        prime(p);
    }else if(pid > 0){
        // 父进程传输筛选的数据给子进程
        close(p[0]); // 父进程关闭读端
        for(int i = 2; i <= 35; ++i){
            write(p[1], &i, sizeof(i)); 
        }
        close(p[1]);// 关闭写端，可以让读端read返回0
        wait(0);// 等待子进程运行结束
        exit(0);
    }
    exit(1);
}
```
效果如图   
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab1-4.png)

### find (moderate)

find命令要求递归查找某个目录下是否具有某个文件，核心是利用fstate函数读取某个文件或者目录，如果是文件，判断文件名是否符合要求，否则，递归查找。

```c
// user/find.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

// 返回文件路径中最后一个/后的字符串
char* fmtname(char *path){
    char *p;

    // Find first character after last slash.
    for(p=path+strlen(path); p >= path && *p != '/'; p--)
        ;
    p++;
    return p;
}

void find(char* path, char* filename){
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;
    if((fd = open(path, 0)) < 0){
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }
    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch(st.type){
        case T_FILE: // 当前path是文件，判断是否符合要求
            if(strcmp(fmtname(filename), fmtname(path)) == 0){
                printf("%s\n", path);
            }
            break;
        case T_DIR:  // 如果当前path是目录，则读取目前中每一个条目
            if(strlen(path) + 1 + DIRSIZ + 1 > sizeof(buf)){
                printf("find: path too long\n");
                break;
            }
            strcpy(buf, path);
            p = buf + strlen(buf);
            *p++ = '/';
            while(read(fd, &de, sizeof(de)) == sizeof(de)){
                // . 和 .. 需要剔除，避免递归死循环
                if(de.inum == 0 || strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
                    continue;
                strcpy(p, de.name); // 更新路径
                find(buf, filename);// 递归查询
            }        
            break;
    }
    close(fd);
}
int main(int argc, char* argv[]){
    if(argc == 2){
        find(".", argv[1]);
    }else if(argc == 3){
        find(argv[1], argv[2]);
    }
    exit(0);
}
``` 
效果如图
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab1-5.png)

### xargs (moderate)

xargs命令是unix下一个十分有用的命令，它可以把输入的数据批量转换为要执行的命令的参数。核心就是按行批量处理输入数据，然后利用exec函数执行。
```c
// user/xargs.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "kernel/param.h"
#include "user/user.h"

int main(int argc, char* argv[]){
    char* args[MAXARG];
    char line[1024];
    if(argc == 1){
        args[0] = "echo";
    }else{
        args[0] = argv[1];
    }
    char* cmd = args[0];
    int x = 1;
    for(int i = 2; i < argc; ++i){
        args[x++] = argv[i];
    }
    // read a line one time
    int n;
    while((n = read(0, line, 1024)) > 0){
        if(fork() == 0){
            // 子进程解析行并调用exec执行命令
           int index = x;
           char* arg = line;
           // printf("line: [%s]\n", line);
           for(int i = 0; i < n; ++i){
               if(line[i] == ' ' || line[i] == '\n'){
                   line[i] = 0;
                   args[index++] = arg;
                   arg = line + i + 1;
               }
           } 

           args[index] = 0;
           exec(cmd, args);
        }else{
            // 父进程等待子进程执行完毕
            wait(0);
        }
    }
    exit(0);
}
```
效果如图
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab1-6.png)

## 实验总结
___
总体来说该lab不难，主要是熟悉一些基本的概念,主要熟悉系统调用，像pipe, fork, fstat, exec等。最有趣的是primes，通过利用fork和管道，实现了数据的流水线并行处理。     

最后运行 make grade可 以得到该lab的所有测试如图
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab1-7.png)
