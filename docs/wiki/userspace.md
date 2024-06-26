# 用户空间

## 用户态与内核态

从执行原理来说，不同进程的指令均运行在同一个处理器上，基于同一个处理器访问系统资源。但从资源管理、安全管理、崩溃预防的角度来看，操作系统需隔离敏感资源，限制不同进程的可执行的指令以及可访问的地址空间。因此，现代操作系统通常会将进程分为两种状态：

-   用户态（user mode）：是指进程运行在低特权级别，只能执行受限指令，访问受限资源；
-   内核态（kernel mode）：是指进程运行在高特权级别，可以执行特权指令，并且可以直接访问所有系统资源。

处理器通常借助某个控制寄存器中的一个模式位（mode bit）来提供区分用户态、内核态。该寄存器描述了进程当前享有的特权。一个运行在内核态的进程可以执行指令集中的任何指令，并访问系统中的任何内存位置。

而用户态中的进程不允许执行特权指令（privileged instruction），比如停止处理器、改变模式位、发起一个 I/O 操作等。用户进程也不能直接访问高权限级的地址空间，**任何这样的尝试都会导致致命的保护异常，触发中断**。

!!! note "关于一般保护异常"

    一般保护异常（General Protection Fault，GPF）是操作系统在发现进程试图执行非法、未授权的或特权级别不足的操作时产生的一种异常。

用户程序必须通过内核态的介入，才能**间接地**访问敏感资源。进程可以通过一些特殊指令（如系统调用、中断）请求操作系统的服务，操作系统会在内核态中执行这些服务，然后返回结果给用户态。

相关数据的传递有一定的约束，这些被称为调用约定（calling convention）。调用约定规定了用户态和内核态之间的数据传递方式，包括参数传递、数据结构类型、返回值传递、异常处理等。

## Rings

在 x86_64 体系结构中，操作系统和应用程序运行在不同的特权级别（privilege levels）中，这些级别被称为 Rings。一般来说，较*低*的数字代表更*高*的特权级别：

-   Ring 0：内核态，最高特权级别，可以执行所有指令，访问所有内存。
-   Ring 1/2：被设计用于设备驱动程序和其他系统服务，但在现代操作系统中不再使用。
-   Ring 3：用户态，最低特权级别，只能执行受限指令，需要访问受限资源时，必须通过系统调用等方式。

!!! question "为什么现代操作系统不使用 Ring 1/2？"

    1. **内存保护的粒度问题：** 虽然 x86 CPU 提供了四个特权级别，但对应特权只能在内存段粒度生效，而非现代操作系统所期待的更细粒度的内存保护。因此，现代大多数 x86 操作系统忽略段机制，而是依赖基于页表项（PTEs）的保护机制。目前， x86 体系结构页表项的保护机制只有两个特权级别，即用户态和内核态。

    2. **可移植性问题：** 操作系统的设计者考虑需到跨平台的可移植性，不仅仅是限于 x86 架构。为了保持操作系统的可移植性，不依赖于特定处理器架构上的特权级别数量，而是统一使用两个特权级别，使得操作系统更易于在不同的处理器架构上移植和运行。

    3. **历史遗留问题：** x86 架构的设计早于现代操作系统的实现，许多 x86 的系统编程特性在设计时并不清楚操作系统将如何使用。随着现代操作系统对 x86 架构的深入了解和不断发展，许多早期设计的特性被现代操作系统所忽略，包括段机制和任务状态段等。对于 x64 架构，一些被废弃的特性被省略，使得操作系统可以更加简洁地设计和实现。

## 用户态库

### 从 hello world 开始

初学 C 语言时，教材展示的第一个程序往往如下：

```c
#include <stdio.h>

int main() {
    printf("Hello, World!\n");
    return 0;
}
```

`#include <stdio.h>` 与 `printf` 函数的组合太过于基础，以至于我们很少去思考：为什么要这么写？是谁实现的 `printf` 函数？这些函数是如何被调用、链接、执行的？为什么调用这些函数就能在屏幕上输出文字？这些问题的答案就在用户态库中。

### 用户态库概述

以 `printf` 为代表的常见函数并不是 C 语言关键字，而是 C 语言标准库的一部分。

C 语言标准库提供了一组函数和数据结构，供应用程序使用以实现各种功能。C 语言标准库中函数和数据结构的实现通常基于操作系统提供的用户态库。**用户态库集成了常见系统服务，屏蔽了底层硬件的细节，为应用程序提供了统一的编程接口。**

在 Linux 系统中，常见的用户态库包括 glibc、musl 等，而在 Windows 系统中对应的用户态库则是 msvcrt 等。这些库通过暴露 C 语言接口，为开发者提供了一种统一的编程 API，使得开发者可以在不同的操作系统环境下编写跨平台的应用程序。

1. **glibc（GNU C Library）**：是 Linux 系统中最常用的 C 语言库之一，提供了许多系统调用的封装，包括文件操作、内存管理、进程控制等。它是 GNU 项目的一部分，具有广泛的支持和社区维护。
2. **musl**：与 glibc 类似，也是一个 C 标准库，但相比于 glibc，musl 更注重轻量级和性能。它专注于遵循 POSIX 标准，并且更加注重代码的简洁和可维护性。
3. **msvcrt**：是 Windows 系统下的 C 运行时库，提供了类似于 glibc 的功能。

### 用户态库的设计

这些用户态库的设计与其提供的服务紧密相关：

-   **跨平台兼容性**：用户态库的设计要求之一即屏蔽底层硬件的差异，向上提供统一的编程接口。换言之，用户态库的一致可以保持用户态程序跨平台兼容性，在更换平台时不需要修改应用程序的代码。
-   **性能与效率**：用户态库的设计需要考虑到性能和效率，尤其是对于系统调用等底层操作的封装，需要尽可能地减少额外的开销。[这篇文章](http://arkanis.de/weblog/2017-01-05-measurements-of-system-call-performance-and-overhead)测量了不同系统调用的性能和开销，并讨论了系统调用设计方法。
-   **功能完备性**：库中需要包含常用的系统调用封装和其他功能，以便开发者可以方便地进行应用程序开发，而不需要重复造轮子。

### 用户态库的服务

以 glibc 为例，以下是常见的一些用户态库 C 语言接口：

1. **系统调用接口**：glibc 提供了访问操作系统底层服务的接口，如 `open()`、`read()`、`write()`等，这些接口允许程序与操作系统进行通信，执行诸如文件操作、进程管理等任务。
2. **内存管理接口**：glibc 提供了内存分配和释放的接口，如 `malloc()`、`free()`、`calloc()`等，这些接口允许程序在运行时动态地分配和释放内存。
3. **文件操作接口**：除了系统调用之外，glibc 还提供了更高层次的文件操作接口，如 `fopen()`、`fclose()`、`fread()`、`fwrite()` 等，这些接口提供了更便利的文件访问方法。
4. **网络和套接字接口**：glibc 提供了一些网络编程的接口，如 `socket()`、`bind()`、`connect()`等，用于创建和管理网络套接字。
5. **其他工具接口**：glibc 提供了一系列工具函数，如字符串处理的接口、数学函数接口等。

这些接口为 C 语言程序提供了丰富的功能和便利的操作方式，使得开发者能够更加轻松地编写各种类型的应用程序。

### Rust 中的用户态库

Rust 语言中的标准用户态库是 `std`，它提供了一系列的系统调用封装、内存管理、文件操作、网络编程等接口。与 C 语言的用户态库类似，Rust 的用户态库也提供了一系列的接口，使得开发者能够更加方便地进行应用程序开发。

不过，与 C 语言的用户态库的二进制 lib 分发不同，Rust 使用源码分发。这意味着开发者可以直接跳转并查看 `std` 的源码，而在 C 语言中定位相关函数时，一般仅能查看其头文件。

## 参考资料

-   [Getting to Ring 3 - OSDev](https://wiki.osdev.org/Getting_to_Ring_3)
-   [Security - OSDev](https://wiki.osdev.org/Security)
-   [Paging - OSDev](https://wiki.osdev.org/Paging)
-   [Protection ring - Wikipedia](https://en.wikipedia.org/wiki/Protection_ring)
-   [Computer Systems: A Programmer's Perspective - CSAPP](http://www.csapp.cs.cmu.edu/3e/home.html)
-   [Why do x86 CPUs only use 2 out of 4 rings?](https://superuser.com/questions/1063420/why-do-x86-cpus-only-use-2-out-of-4-rings)
-   [printf 是怎么输出到控制台的呢？](https://www.zhihu.com/question/456916638/answer/3099313413)
