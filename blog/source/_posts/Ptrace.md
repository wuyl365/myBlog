---
title: Ptrace实现反调试
date: 2022-04-26 14:56:51
tags:
---

# Ptrace实现反调试

### ptrace介绍

ptrace系统调用提供了一种方法，这个方法可以让一个进程监视、控制另一个进程的执行，并且可以查看和更改被追踪进程的内存和寄存器。通常用来下断点和调试。

其基本原理是: 当使用了ptrace跟踪后，所有发送给被跟踪的子进程的信号(除了SIGKILL)，都会被转发给父进程，而子进程则会被阻塞，这时子进程的状态就会被系统标注为TASK_TRACED。而父进程收到信号后，就可以对停止下来的子进程进行检查和修改，然后让子进程继续运行。

### ptrace函数

```
#include <sys/ptrace.h>
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void  *data);
```

ptrace_request request：指示了ptrace要执行的命令。

1. enum __ptrace_request request：指示了ptrace要执行的命令。
2. pid_t pid: 指示ptrace要跟踪的进程。
3. void *addr: 指示要监控的内存地址。
4. void *data: 存放读取出的或者要写入的数据。

ptrace是如此的强大，以至于有很多大家所常用的工具都基于ptrace来实现，如GDB

### GDB

GDB是GNU发布的一个强大的程序调试工具，用以调试C/C++程序。可以使程序员在程序运行的时候观察程序在内存/寄存器中的使用情况。它的实现也是基于ptrace系统调用来完成的。
其原理是利用ptrace系统调用，在被调试程序和gdb之间建立跟踪关系。然后所有发送给被调试程序的信号(除SIGKILL)都会被gdb截获，gdb根据截获的信号，查看被调试程序相应的内存地址，并控制被调试的程序继续运行。GDB常用的使用方法有断点设置和单步跟踪，接下来我们来分析一下他们是如何实现的。

### 反调试

在类 Unix 系统中，提供了一个系统调用 `ptrace` 用于实现断点调试和对进程进行跟踪和控制，而 `PT_DENY_ATTACH` 是苹果增加的一个 `ptrace` 选项，用于阻止 GDB 等调试器依附到某进程

```
ptrace(PT_DENY_ATTACH, 0, 0, 0);
void anti_gdb_debug() {

    void *handle = dlopen(0, RTLD_GLOBAL | RTLD_NOW);

    ptrace_ptr_t ptrace_ptr = dlsym(handle, "ptrace");

    ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0);

    dlclose(handle);

}
// 阻止 gdb/lldb 调试
// 调用 ptrace 设置参数 PT_DENY_ATTACH，如果有调试器依附，则会产生错误并退出
#import <dlfcn.h>
#import <sys/types.h>

typedef int (*ptrace_ptr_t)(int _request, pid_t _pid, caddr_t _addr, int _data);
#if !defined(PT_DENY_ATTACH)
#define PT_DENY_ATTACH 31
#endif

void anti_gdb_debug() {
    void *handle = dlopen(0, RTLD_GLOBAL | RTLD_NOW);
    ptrace_ptr_t ptrace_ptr = dlsym(handle, "ptrace");
    ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0);
    dlclose(handle);
}

int main(int argc, char * argv[]) {
#ifndef DEBUG
    // 非 DEBUG 模式下禁止调试
    anti_gdb_debug();
#endif
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

此时，如果尝试对 App 进行 GDB 依附，则会得到一个 Segmentation fault 错误。

ptrace被广泛用于反调试,因为一个进程只能被ptrace一次,如果事先调用了ptrace方法,那就可以防止别人调试我们的程序.

### 反调试方法2

如果别人的的app进行了ptrace防护，那么怎么让他的ptrace不起作用呢。

由于ptrace是系统函数，那么我们可以用fishhook来hook住ptrace函数，然后让他的app调用我们自己的ptrace函数，即写动态库，如果多个动态库hook 了ptrace ,我们可以调整 Link Binary Libraries的顺序加载，假设人家应用自己写的hook ptrace动态库肯定会在自己前面，最后的方式我们可以通过修改macho的二进制让他的ptrace失效【不去执行ptrace】，然后进行调试.

比如我们使用MonkeyDev，默认打开了

```
rebind_symbols((struct rebinding[1]){{"ptrace", my_ptrace, (void*)&orig_ptrace}},1);
int my_ptrace(int _request, pid_t _pid, caddr_t _addr, int _data){
    if(_request != 31){
        return orig_ptrace(_request,_pid,_addr,_data);
    }
    
    NSLog(@"[AntiAntiDebug] - ptrace request is PT_DENY_ATTACH");
    
    return 0;
}
```

就可以直接绕过

所以我们不想暴露ptrace系统方法，不希望被断点断住

系统提供了一个系统调用函数syscall ，所有的系统调用都可以通过syscall 去实现。 syscall (26,31,0,0) 来调用系统函数ptrace，ptrace的系统调用函数号是26，syscall是通过软中断来实现从用户态到内核态。当然这种方式也可以通过hook syscall来绕开。

那么我们可以通过汇编svc调用来实现。可以使用内联汇编去实现反调试的代码

```
// 使用inline方式将函数在调用处强制展开，防止被hook和追踪符号
static __attribute__((always_inline)) void anti_debug()
{
    // 判断是否是ARM64处理器指令集
#ifdef __arm64__
    // volatile修饰符能够防止汇编指令被编译器忽略
    __asm__ __volatile__
    (
     "mov x0, #26\n"
     "mov X1, #31\n"
     "mov X2, #0\n"
     "mov X3, #0\n"
     "mov X4, #0\n"
     "mov w16, #0\n"
     "svc #0x80"
    );
#endif
}
```

可以看出上面原理就是调用了26号系统调用，26是ptrace 0x80触发中断取找w16执行.
