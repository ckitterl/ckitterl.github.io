---
layout: post
title:  "x86_64和arm64的inline-assembly的hello world"
categories: assembly
---

这这篇文章里提供了两种ABI下的hello world如何在inline assembly下实现。从这些代码里我们可以学到几个技术点

- inline assembly的格式
- 不同ABI的寄存器使用(尤其是arm64)
- 不同ABI的系统调用

这个篇的基础上，我们可以自行扩展测试其他的汇编特性。比如汇编和C的互相调用, label的使用


## 基础

inline assmble的基础语法

```
asm asm-qualifiers ( AssemblerTemplate
                 : OutputOperands
                 [ : InputOperands
                 [ : Clobbers ] ])

asm asm-qualifiers ( AssemblerTemplate
                      : OutputOperands
                      : InputOperands
                      : Clobbers
                      : GotoLabels)
```


asm 关键字表示这是一个inline assembly

这里不详述了，[官方文档](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Volatile)写的还是很详细的



## x86_64

```c
/* hello_x86_64.c */
#include <unistd.h>
#include <asm/unistd.h>
ssize_t my_write(int fd, const void *buf, size_t size);
void my_exit();
ssize_t my_write(int fd, const void *buf, size_t size)
{
    ssize_t ret;
    asm volatile
    (
        "syscall"
        : "=a" (ret)
        // RAX             RDI      RSI       RDX
        : "A"(__NR_write), "D"(fd), "S"(buf), "d"(size)
        : "rcx", "r11", "memory"
    );
    return ret;
}

void my_exit() {
   /* __asm__ ("movq $60, %rax\n\t" // the exit syscall number on Linux*/
             /*"movq $2,  %rdi\n\t" // this program returns 2*/
             /*"syscall");*/
    asm ("syscall" :: "A"(__NR_exit), "D"(100));
}

int main() {
    my_write(1, "hello world!\n",  13);
    my_exit();
	// 这里没有return也是可以的，因为在my_exit函数里就已经退出程序了; 可以用echo $?查看最终的exit code，
	// 不是main函数的retrun值，而是上面my_exit的退出参数100
	return 0;
}
```

## arm64

arm64的line assembly不支持在input/output里指定特定的寄存器，但是系统调用需要将[特定的参数放到特定寄存器中](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md)，所以就需要用register关键字将变量和寄存器进行[绑定](https://gcc.gnu.org/onlinedocs/gcc/Local-Register-Variables.html#Local-Reg-Vars)。

> 经过测试，通用寄存器都是从0开始依次使用，所以，fd_local, buf, size_local这些都不是必须的，因为本来他们就是放到x0,x1,x2，刚好符合顺序，这里为了统一并不对读者产生阅读干扰，都是指定了寄存器。


```c
/* hello_arm64.c */
#include <unistd.h>
#include <asm/unistd.h>

int my_write(int fd, const void *buf, size_t size) {
    register long ret asm("x0");
    register long fd_local asm("x0") = fd;
    register const void *buf_local asm("x1") = buf;
    register size_t size_local asm("x2") = size;
    register long call  asm("r8") = __NR_write;
    asm volatile(
            "svc #0" // 系统调用，用arm-develop的说法就是陷入E1层,和x86的原理不太一样
            : "=r" (ret)
            : "r"(call), "r"(fd_local), "r"(buf), "r"(size_local)
            : "memory"
    );
    return ret;
}

int main() {
    my_write(1, "hello world!\n", 13);
    return 0;
}
```

这段代码要用arm64(aarch64)的toolchain进行编译；可以选用android ndk提供的或者单独安装编译链工具集; 这里采用android ndk

```bash
 $ $ANDROID_SDK/ndk/21.3.6528147/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android29-clang hello_arm64.c -o hello_arm64
```

如果有root的android手机，可以push到手机上运行；或者安装模拟器，linux系统可以安装[qumu](https://qemu-project.gitlab.io/qemu/user/main.html)来运行

```bash
qemu-aarch64  hello_arm64
```

> 还有一种方法就是用docker安装arm64-v8a的ubuntu，这样就可以在对应的container里构建运行了，比较方便



