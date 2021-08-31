---
layout: post
title:  "x86_64��arm64��inline-assembly��hello world"
categories: assembly
---

����ƪ�������ṩ������ABI�µ�hello world�����inline assembly��ʵ�֡�����Щ���������ǿ���ѧ������������

- inline assembly�ĸ�ʽ
- ��ͬABI�ļĴ���ʹ��(������arm64)
- ��ͬABI��ϵͳ����

���ƪ�Ļ����ϣ����ǿ���������չ���������Ļ�����ԡ��������C�Ļ������, label��ʹ��


## ����

inline assmble�Ļ����﷨

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


asm �ؼ��ֱ�ʾ����һ��inline assembly

���ﲻ�����ˣ�[�ٷ��ĵ�](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Volatile)д�Ļ��Ǻ���ϸ��



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
	// ����û��returnҲ�ǿ��Եģ���Ϊ��my_exit��������Ѿ��˳�������; ������echo $?�鿴���յ�exit code��
	// ����main������retrunֵ����������my_exit���˳�����100
	return 0;
}
```

## arm64

arm64��line assembly��֧����input/output��ָ���ض��ļĴ���������ϵͳ������Ҫ��[�ض��Ĳ����ŵ��ض��Ĵ�����](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md)�����Ծ���Ҫ��register�ؼ��ֽ������ͼĴ�������[��](https://gcc.gnu.org/onlinedocs/gcc/Local-Register-Variables.html#Local-Reg-Vars)��

> �������ԣ�ͨ�üĴ������Ǵ�0��ʼ����ʹ�ã����ԣ�fd_local, buf, size_local��Щ�����Ǳ���ģ���Ϊ�������Ǿ��Ƿŵ�x0,x1,x2���պ÷���˳������Ϊ��ͳһ�����Զ��߲����Ķ����ţ�����ָ���˼Ĵ�����


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
            "svc #0" // ϵͳ���ã���arm-develop��˵����������E1��,��x86��ԭ��̫һ��
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

��δ���Ҫ��arm64(aarch64)��toolchain���б��룻����ѡ��android ndk�ṩ�Ļ��ߵ�����װ���������߼�; �������android ndk

```bash
 $ $ANDROID_SDK/ndk/21.3.6528147/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android29-clang hello_arm64.c -o hello_arm64
```

�����root��android�ֻ�������push���ֻ������У����߰�װģ������linuxϵͳ���԰�װ[qumu](https://qemu-project.gitlab.io/qemu/user/main.html)������

```bash
qemu-aarch64  hello_arm64
```

> ����һ�ַ���������docker��װarm64-v8a��ubuntu�������Ϳ����ڶ�Ӧ��container�ﹹ�������ˣ��ȽϷ���



