---
title: "Linux系统调用"
date: 2020-06-07T20:18:30+08:00
keywords: ["系统调用"]
categories : ["系统调用"]
banner : "img/blogimg/xitongdiaoyong.png"
summary : "本期重点和大家讨论系统调用机制。其中涉及到了一些及系统调用的性能、上下文深层问题，同时也穿插着讲述了一些内核调试方法。并 且最后试验部分我们利用系统调用与相关内核服务完成了一个搜集系统调用序列的特定任务，该试验具有较强的实用和教学价值。"
---

#### 系统调用

##### 1. 什么是系统调用

​		系统调用，顾名思义，说的是操作系统提供给用户程序调用的一组“特殊”接口。用户程序可以通过这组“特殊”接口来获得操作系统内核提供的服务，比如用户可以通过文件系统相关的调用请求系统打开文件、关闭文件或读写文件，可以通过时钟相关的系统调用获得系统时间或设置定时器等。

​		从逻辑上来说，系统调用可被看成是一个内核与用户空间程序交互的接口——它好比一个中间人，把用户进程的请求传达给内核，待内核把请求处理完毕后再将处理结果送回给用户空间。

​		系统服务之所以需要通过系统调用来提供给用户空间的根本原因是为了对系统进行“保护”，因为我们知道Linux的运行空间分为内核空间与用户空间，它们各自运行在不同的级别中，逻辑上相互隔离。所以用户进程在通常情况下不允许访问内核数据，也无法使用内核函数，它们只能在用户空间操作用户数据，调用用户空间函数。比如我们熟悉的“hello world”程序（执行时）就是标准的用户空间进程，它使用的打印函数`printf`就属于用户空间函数，打印的字符“hello word”字符串也属于用户空间数据。

​		但是很多情况下，用户进程需要获得系统服务（调用系统程序），这时就必须利用系统提供给用户的“特殊接口”——系统调用了，它的特殊性主要在于规定了用户进程进入内核的具体位置；换句话说，用户访问内核的路径是事先规定好的，只能从规定位置进入内核，而不准许肆意跳入内核。有了这样的陷入内核的统一访问路径限制才能保证内核安全无虞。我们可以形象地描述这种机制：作为一个游客，你可以买票要求进入野生动物园，但你必须老老实实地坐在观光车上，按照规定的路线观光游览。当然，不准下车，因为那样太危险，不是让你丢掉小命，就是让你吓坏了野生动物。



##### 2. Linux的系统调用

​		对于现代操作系统，系统调用是一种内核与用户空间通讯的普遍手段，Linux系统也不例外。但是Linux系统的系统调用相比很多Unix和windows等系统具有一些独特之处，无处不体现出Linux的设计精髓——简洁和高效。

 		Linux系统调用很多地方继承了Unix的系统调用（但不是全部），但Linux相比传统Unix的系统调用做了很多扬弃，它省去了许多Unix系统冗余的系统调用，仅仅保留了最基本和最有用的系统调用，所以Linux全部系统调用只有250个左右（而有些操作系统系统调用多达1000个以上）。

​		这些系统调用按照功能逻辑大致可分为“进程控制”、“文件系统控制”、“系统控制”、“存储管理”、“网络管理”、“socket控制”、“用户管理”、“进程间通信”等几类，详细情况可参阅文章[系统调用列表](http://www-900.ibm.com/developerworks/cn/linux/kernel/syscall/part1/appendix.shtml)

​		如果你想详细看看系统调用的说明，可以使用`man 2 syscalls` 命令查看，或干脆到 `<内核源码目录>/include/asm-i386/unistd.h`源文件中找到它们的源本。

​		熟练了解和掌握上面这些系统调用是对系统程序员的必备要求，但对于一个开发内核的人员或内核开发者来[[1\]](http://wwww.kerneltravel.net/journal/iv/syscall.htm#_ftn1)说，死记硬背下这些调用还远远不够。如果你仅仅知道存在的调用而不知道为什么它们会存在，或只知道如何使用调用而不知道这些调用在系统中的主要用途，那么你离驾驭系统还有不小距离。

​		要弥补这个鸿沟，第一，你必须明白系统调用在内核里的主要用途。虽然上面给出了数种分类，不过，总的概括来讲，系统调用在系统中的主要用途无非以下几类：

- 控制硬件——系统调用往往作为硬件资源和用户空间的抽象接口，比如读写文件时用到的`write/read`调用。
- 设置系统状态或读取内核数据——因为系统调用是用户空间和内核的唯一通讯手段[[2\]](http://wwww.kerneltravel.net/journal/iv/syscall.htm#_ftn2)，所以用户设置系统状态，比如开/关某项内核服务（设置某个内核变量），或读取内核数据都必须通过系统调用。比如`getpgid、getpriority、setpriority、sethostname`
- 进程管理——一系统调用接口是用来保证系统中进程能以多任务在虚拟内存环境下得以运行。比如 `fork、clone、execve、exit`等

第二，什么服务应该存在于内核；或者说什么功能应该实现在内核而不是在用户空间。这个问题并没有明确的答案，有些服务你可以选择在内核完成，也可以在用户空间完成。选择在内核完成通常基于以下考虑：

- 服务必须获得内核数据，比如一些服务必须获得中断或系统时间等内核数据。
- 从安全角度考虑，在内核中提供的服务相比用户空间提供的毫无疑问更安全，很难被非法访问到。
- 从效率考虑，在内核实现服务避免了和用户空间来回传递数据以及保护现场等步骤，因此效率往往要比在用户空间实现高许多。比如`httpd`等服务。
- 如果内核和用户空间都需要使用该服务，那么最好实现在内核空间，比如随机数产生。

 理解上述道理对掌握系统调用的本质意义很大，希望网友们能从使用中多总结，多思考。

##### 3. 系统调用、用户编程接口（**API**）、系统命令和内核函数的关系

​		系统调用并非直接和程序员或系统管理员打交道，它仅仅是一个通过软中断机制（我们后面讲述）向内核提交请求，获取内核服务的接口。而在实际使用中程序员调用的多是用户编程接口——`API`，而管理员使用的则多是系统命令。

​		用户编程接口其实是一个函数定义，说明了如何获得一个给定的服务，比如`read()、malloc()、free()、abs()`等。它有可能和系统调用形式上一致，比如`read()`接口就和`read`系统调用对应，但这种对应并非一一对应，往往会出现几种不同的API内部用到同一个系统调用，比如`malloc()、free()`内部利用`brk()`系统调用来扩大或缩小进程的堆；或一个API利用了好几个系统调用组合完成服务。更有些`API`甚至不需要任何系统调用——因为它并不是必需要使用内核服务，如计算整数绝对值的`abs()`接口。

​		另外要补充的是Linux的用户编程接口遵循了在Unix世界中最流行的应用编程界面标准——`POSIX`标准，这套标准定义了一系列`API`。在Linux中（Unix也如此），这些`API`主要是通过C库（`libc`）实现的，它除了定义的一些标准的C函数外，一个很重要的任务就是提供了一套封装例程（`wrapper routine`）将系统调用在用户空间包装后供用户编程使用。

​		不过封装并非必须的，如果你愿意直接调用，Linux内核也提供了一个`syscall()`函数来实现调用，我们看个例子来对比一下通过C库调用和直接调用的区别。

```c
#include <syscall.h>
#include <unistd.h>
#include <stdio.h>
#include <sys/types.h>

int main(void) {
	long ID1, ID2;

	/*-----------------------------*/
	/* 直接系统调用*/
	/* SYS_getpid (func no. is 20) */
	/*-----------------------------*/

	ID1 = syscall(SYS_getpid);

	printf ("syscall(SYS_getpid)=%ld\n", ID1);

	/*-----------------------------*/
	/* 使用"libc"封装的系统调用 */
	/* SYS_getpid (Func No. is 20) */
	/*-----------------------------*/

	ID2 = getpid();

	printf ("getpid()=%ld\n", ID2);

	return(0);
}
```

​		系统命令相对编程接口更高了一层，它是内部引用API的可执行程序，比如我们常用的系统命令`ls、hostname`等。Linux的系统命令格式遵循系统V的传统，多数放在`/bin`和`/sbin`下（相关内容可看看shell等章节）。

​		有兴趣的话，可以通过`strace ls`或`strace hostname` 命令查看一下它们用到的系统调用，你会发现诸如`open、brk、fstat、ioctl` 等系统调用被用在系统命令中。

​		下一个需要解释一下的问题是内核函数和系统调用的关系。大家不要把内核函数想像的过于复杂，其实它们和普通函数很像，只不过在内核实现，因此要满足一些内核编程的要求[[3\]](http://wwww.kerneltravel.net/journal/iv/syscall.htm#_ftn3)。系统调用是一层用户进入内核的接口，它本身并非内核函数，进入内核后，不同的系统调用会找到对应到各自的内核函数——换个专业说法就叫：系统调用服务例程。实际上针对请求提供服务的是内核函数而非调用接口。

​		比如系统调用 `getpid`实际上就是调用内核函数`sys_getpid`。

```c
asmlinkage long sys_getpid(void)
{
    return current->tpid;
}
```

​		Linux系统中存在许多内核函数，有些是内核文件中自己使用的，有些则是可以`export`出来供内核其他部分共同使用的，具体情况自己决定。

​		内核公开的内核函数——export出来的——可以使用命令`ksyms` 或 `cat /proc/ksyms`来查看。另外，网上还有一本归纳分类内核函数的书叫作`《The Linux Kernel API Book》`，有兴趣的读者可以去看看。

​		总而言之，从用户角度向内核看，依次是系统命令、编程接口、系统调用和内核函数。在讲述了系统调用实现后，我们会回过头来看看整个执行路径。

##### 4. 系统调用**的**实现

​		Linux中实现系统调用利用了`X86`体系结构中的软件中断[[4\]](http://wwww.kerneltravel.net/journal/iv/syscall.htm#_ftn4)。软件中断和我们常说的中断(硬件中断)不同之处在于——它是通过软件指令触发而并非外设引发的中断，也就是说，又是编程人员开发出的一种异常，具体的讲就是调用`int $0x80`汇编指令，这条汇编指令将产生向量为`128`的编程异常。

​		之所以系统调用需要借助异常来实现，是因为当用户态的进程调用一个系统调用时，CPU便被切换到内核态执行内核函数[[5\]](http://wwww.kerneltravel.net/journal/iv/syscall.htm#_ftn5)，而我们在i386体系结构部分已经讲述过了进入内核——进入高特权级别——必须经过系统的门机制，这里的异常实际上就是通过系统门陷入内核（除了`int 0x80`外用户空间还可以通过`int3`——向量3、`into`——向量4 、`bound`——向量5等异常指令进入内核，而其他异常无法被用户空间程序利用，都是由系统使用的）。

​		我们更详细地解释一下这个过程。`int $0x80`指令的目的是产生一个编号为128的编程异常，这个编程异常对应的是中断描述符表**IDT**中的第128项——也就是对应的系统门描述符。门描述符中含有一个预设的内核空间地址，它指向了系统调用处理程序：`system_call()`（别和系统调用服务程序混淆,这个程序在`entry.S`文件中用汇编语言编写）。

​		很显然，所有的系统调用都会统一地转到这个地址，但Linux一共有2、3百个系统调用都从这里进入内核后又该如何派发到它们到各自的服务程序去呢？别发昏，解决这个问题的方法非常简单：首先Linux为每个系统调用都进行了编号（`0—NR_syscall`），同时在内核中保存了一张系统调用表，该表中保存了系统调用编号和其对应的服务例程，因此在系统调入通过系统门陷入内核前，需要把系统调用号一并传入内核，在**X86**上，这个传递动作是通过在执行`int0x80`前把调用号装入`eax`寄存器实现的。这样系统调用处理程序一旦运行，就可以从`eax`中得到数据，然后再去系统调用表中寻找相应服务例程了。

​		除了需要传递系统调用号以外，许多系统调用还需要传递一些参数到内核，比如`sys_write(unsigned int fd, const char * buf, size_t count)`调用就需要传递文件描述符`fd`、要写入的内容`buf`、以及写入字节数`count`等几个内容到内核。碰到这种情况，Linux会有6个寄存器可被用来传递这些参数：`eax` (存放系统调用号)、 `ebx、ecx、edx、esi`及`edi`来存放这些额外的参数（以字母递增的顺序）。具体做法是在`system_call( )`中使用`SAVE_ALL`宏把这些寄存器的值保存在内核态堆栈中。

 		有始便有终，当服务例程结束时，`system_call( )` 从`eax`获得系统调用的返回值，并把这个返回值存放在曾保存用户态 `eax`寄存器栈单元的那个位置上。然后跳转到`ret_from_sys_call( )`，终止系统调用处理程序的执行。

​		当进程恢复它在用户态的执行前，`RESTORE_ALL`宏会恢复用户进入内核前被保留到堆栈中的寄存器值。其中`eax`返回时会带回系统调用的返回码。（负数说明调用错误，0或正数说明正常完成）

 		我们可以通过分析一下`getpid`系统调用的真是过程来将上述概念具体化，分析`getpid`系统调用的一个办法是查看`entry.s`中的代码细节，逐步跟踪源码来分析运行过程，另外就是可借助一些内核调试工具，动态跟踪运行路径。

​		假设我们的程序源文件名为`getpid.c`，内容是：

```c
#include <syscall.h>
#include <unistd.h>
#include <stdio.h>
#include <sys/types.h>

int main(void) {
	long ID;
	
	ID = getpid();

	printf ("getpid()=%ld\n", ID);

	return(0);
}
```

将其编译成名为`getpid`的执行文件“`gcc –o getpid <路径>/getpid.c`”, 我们使用**KDB**来产看它进入内核后的执行路径。

- 激活`KDB` (按下`pause`键，当然你必须已经给内核打了`KDB`补丁);设置内核断点 “`bp sys_getpid`” ;退出`kdb “go”`;然后执行`./getpid` 。瞬间，进入内核调试状态,执行路径停止在断点`sys_getpid`处。
- 在`KDB>`提示符下，执行bt命令观察堆栈，发现调用的嵌套路径，可以看到在`sys_getpid`是在内核函数`system_call`中被嵌套调用的。
- 在`KDB>`提示符下，执行`rd`命令查看寄存器中的数值，可以看到`eax`中存放的`getpid`调用号——`0x00000014(=20)`.
- 在`KDB>`提示符下，执行`ssb`（或`ss`）命令跟踪内核代码执行路径,可以发现`sys_getpid`执行后，会返回`system_call`函数，然后接者转入`ret_from_sys_call`例程。（再往后还有些和调度有关其他例程，我们这里不说了它们了）

结合用户空间的执行路径，该程序大致可归结为以下几个步骤：

1. 该程序调用`libc`库的封装函数`getpid`。该封装函数将系统调用号`_NR_getpid`（第20个）压入`EAX`寄存器
2. 调用软中断 `int 0x80` 进入内核。

**（以下进入内核态）**

3. 在内核中首先执行`system_call`，接着执行根据系统调用号在调用表中查找到的对应的系统调用服务例程`sys_getpid`。
4. 执行`sys_getpid`服务例程。
5. 执行完毕后，转入`ret_from_sys_call`例程，系统调用中返回。

内核调试是一个很有趣的话题，方法多种多样，我个人认为比较好用的是`UML`（`user mode linux+gdb`）和 `KDB` 这两个工具。尤其`KDB`对于调试小规模内核模块或查看内核运行路径很有效，对于它的使用方法可以看看[Linux内核调试内幕](http://www-900.ibm.com/developerWorks/cn/linux/l-kdbug/index.shtml)这篇文章。

##### 5. 系统调用的思考

​		系统调用的内在过程并不复杂，我们不再多说了，下面这节我们主要就系统调用所涉及的一些重要问题作一些讨论和分析，希望这样能更有助于了解系统调用的精髓。

###### 5.1 调用上下文分析

​		系统调用虽说是要进入内核执行，但它并非一个纯粹意义上的内核例程。首先它是代表用户进程的，这点决定了虽然它会陷入内核执行，但是上下文仍然是处于进程上下文中，因此可以访问进程的许多信息（比如`current`结构——当前进程的控制结构），而且可以被其他进程抢占（在从系统调用返回时，由`system_call`函数判断是否该再调度），可以休眠，还可接收信号[[6\]](http://wwww.kerneltravel.net/journal/iv/syscall.htm#_ftn6)等等。

​		所有这些特点都涉及到了进程调度的问题，我们这里不做深究，只要大家明白系统调用完成后，再回到或者说把控制权交回到发起调用的用户进程前，内核会有一次调度。如果发现有优先级别更高的进程或当前进程的时间片用完，那么就会选择高优先级的进程或重新选择进程运行。除了再调度需要考虑外，再就是内核需要检查是否有挂起的信号，如果发现当前进程有挂起的信号，那么还需要先返回用户空间处理信号处理例程（处于用户空间），然后再回到内核，重新返回用户空间，有些麻烦但这个反复过程是必须的。

###### 5.2 调用性能问题

​		系统调用需要从用户空间陷入内核空间，处理完后，又需要返回用户空间。其中除了系统调用服务例程的实际耗时外，陷入/返回过程和系统调用处理程序（查系统调用表、存储/恢复用户现场）也需要花费一些时间，这些时间加起来就是一个系统调用的响应速度。系统调用不比别的用户程序，它对性能要求很苛刻，因为它需要陷入内核执行，所以和其他内核程序一样要求代码简洁、执行迅速。幸好Linux具有令人难以置信的上下文切换速度，使得其进出内核都被优化得简洁高效；同时所有Linux系统调用处理程序和每个系统调用本身也都非常简洁。

​		绝大多数情况下，Linux系统调用的性能是可以接受的，但是对于一些对性能要求非常高的应用来说，它们虽然希望利用系统调用的服务，但却希望加快响应速度，避免陷入/返回和系统调用处理程序带来的花销，因此采用由内核直接调用系统调用服务例程，最好的例子就`HTTPD`——它为了避免上述开销，从内核调用`socket`等系统调用服务例程。

###### 5.3 什么时候添加系统调用

​		系统调用是用户空间和内核空间交互的唯一手段，但是这并不是说要完成交互功能就非要添加新系统调用不可。添加系统调用需要修改内核源代码、重新编译内核，因此如果想灵活地和内核交互信息，最好使用以下几种方法：

- 编写字符驱动程序

  利用字符驱动程序可以完成和内核交互数据的功能。它最大的好处在于可以模块式加载，这样一来就避免了编译内核等手续，而且调用接口固定，容易操作。

- 使用`proc` 文件系统

  利用`proc`文件系统修订系统状态是一种很常见的手段，比如通过修改`proc`文件系统下的系统参数配置文件（`/proc/sys`），我们可以直接在运行时动态更改内核参数；再如，通过下面这条指令：`echo 1 > /proc/sys/net/ip_v4/ip_forward`开启内核中控制IP转发的开关。类似的，还有许多内核选项可以直接通过`proc`文件系统进行查询和调整。

- 使用虚拟文件系统

  有些内核开发者认为利用`ioctl（）`系统调用（字符设备驱动接口）往往会使得系统调用意义不明确，而且难以控制。而将信息放入到`proc`文件系统中会使信息组织混乱，因此也不赞成过多使用。他们建议实现一种孤立的虚拟文件系统来代替`ioctl()`和`/proc`，因为文件系统接口清楚，而且便于用户空间访问，同时，利用虚拟文件系统使得利用脚本执行系统管理任务更加方便、有效。

##### 6. 实验部分

###### 6.1 试验概述

​		本章的试验，我们将编写一个搜集内核中系统调用发生序列信息的内核服务，并利用一个新的系统调用为用户程序取回这些信息。

​		这个试验不但能教会大家如何为内核添加新系统调用，而且能教大家学会用户程序如何使用系统调用获取内核服务。更进一步，大家可以通过观察系统调用序列和发生频率更深入地理解系统调用和系统运行的关系。当然，这里除了系统调用的知识外，还会涉及到一些有关内核编程，比如等待队列、内核模块等知识，这些部分请大家查看相关资料。

​		我们的新系统调用为`audit`（在2.4.18内核中系统调用号是223，置于调用表最末尾），该调用对应的具体实现是内核函数`sys_audit`，它将把事先记录的系统调用序列信息（比如系统调用的进程号，命令等相关信息）返回给用户空间。而记录这些系统调用序列则是依靠另一个内核服务函数`syscall_audit`，该函数负责搜集系统调用数据并填充到一个自建的内核缓冲区中，等待系统调用audit将搜集到的内核数据取回到用户空间。

​		它的搜集方式很有意思。首先要修改系统调用处理程序`system_call`，在其中需要监控的每个调用（在我们例子钟222个系统调用都监控了，当然你也可以根据自己需求有选择的监控）执行完毕后都插入一个指令，该指令会转去调用内核服务函数`syscall_audit`来记录该次调用的信息。因为任何一个系统调用都要经过`system_call`统一处理，所以任何一次系统调用的信息都可被`syscall_audit`记录下来。

​		`Syscall_audit`内核服务函数要做的事情就是记录系统调用的信息，具体做法是建立一个内核缓冲区来存放被记录的函数。当搜集的数据量到达一定阀值时（比如设定为到达缓冲区总大小的80％，这样做可以避免再丢失新调用），唤醒系统调用进程取回数据。否则继续搜集，这时，系统调用程序会堵塞在一个等待队列上，直到被唤醒，也就是说，如果缓冲区还没接近满时，系统调用会等待它被填充。

​		`Sys_audit`系统调用服务函数所做的事情很简单，就是从缓冲区中取数据返回用户空间。如果缓冲还没满则挂起等待。

​		道理就这么多了。不过为了方便调试，我们采用模块化方式来加载`sys_audit`和`syscall_audit`。于是在内核内只需要提供`sys_audit`和`syscall_audit`两个函数的钩子，而在模块中利用`my_audit`（对应`syscall_audit`）和`my_sysaudit`(对应sys_audit)实现它们的具体功能。

​		除了内核函数以外，我们还需要一个用户空间的`daemon`程序（`auditd`），来不断地调用audit系统调用，以便搜集系统中发生的调用序列信息。（长时间的调用序列对于分析入侵或系统行为等才有价值）

![image003](E:\Linux内核之旅开源社区\website\content\blog\2020\Electronic_Magazine_Issue_4_hds_no2\img\image003.gif)

###### 6.2 Step by step

下面具体讲述一下如何添加这个调用。通过添加该调用可以学习内核编程、模块编程等许多有趣的技巧。

1. 修改`entry.S` ——在其中添加`audit`调用，并且在`system_call`中加入搜集例程。（该函数位于`<内核源代码>/arch/i386/kernel/`下）
2. 添加`audit.c`文件到`<内核源代码>/arch/i386/kernel/`下——该文件中定义了`sys_audit`和`syscall_audit` 两个函数需要的钩子函数（`my_audit`和`my_sysaudit`），它们会在`entry.S`中被使用。
3. 修改`<内核源代码>/arch/i386/kernel/i386-kysms.c`文件，在其中导出`my_audit`与`my_sysaudit`两个钩子函数。因为只有在内核符号表里导出，才可被其他内核函数使用，也就是说才能在模块中被挂上。
4. 修改`<内核源代码>/arch/i386/kernel/Makefile`文件，将`audit.c`编译入内核。

到这里就可以重新编译内核了，新内核已经加入了检测点。下一步就是编写模块来实现系统调用与内核搜集服务例程的功能。

1. 编写名为`audit`的模块，其中除了加载、卸载模块函数以外，主要实现了`mod_sys_audit`与`mod_syscall_audit`两个函数。它们会分别挂载到`my_sysaudit`和`my_audit`两个钩子上。
2. 编译后将模块加载 `insmod audit.o`。（你可通过`dmesg`查看加载信息）
3. 修改`/usr/include/asm/unistd.h` ——在其中加入audit的系统调用号。这样用户空间才可以找到audit系统调用。
4. 最后，我们写一个用户`deamon`程序，来循环调用audit系统调用，并把搜集到的信息打印到屏幕上。

完了。系统调用还有许多细节，请大家查看有关书籍吧。不啰嗦了。再见。

 

相关代码请下载  [auditexample.tar](img/auditexample.tar.gz)（实现于2.4.18内核）。

感谢SAL的开发者，例子程序基本框架来自于他们的灵感。

------

[[1]](http://wwww.kerneltravel.net/journal/iv/syscall.htm#_ftnref1)我们说的开发内核人员指开发系统内核，比如开发驱动模块机制、开发系统调用机制；而内核开发者则是指在内核基础之上进行的开发，比如驱动开发、系统调用开发、文件系统开发、网络通讯协议开发等。我们杂志所关注的问题主要在内核开发层次，即利用内核提供的机制进行开发。

[[2]](http://wwww.kerneltravel.net/journal/iv/syscall.htm#_ftnref2)对Linux而言，系统调用是用户程序访问内核的唯一手段，无论是`/proc`方式或设备文件方式归根到底都是利用系统调用完成的。

[[3]](http://wwww.kerneltravel.net/journal/iv/syscall.htm#_ftnref3)内核编程相比用户程序编程有一些特点，简单地讲，内核程序一般不能引用C库函数（除非你自己实现了，比如内核实现了不少C库种的String操作函数）；缺少内存保护措施；堆栈有限（因此调用嵌套不能过多）；而且由于调度关系，必须考虑内核执行路径的连续性，不能有长睡眠等行为。

[[4]](http://wwww.kerneltravel.net/journal/iv/syscall.htm#_ftnref4)软件中断虽然叫中断，但实际上属于异常（更准确说是陷阱）——CPU发出的中断——而且是由编程者触发的一种特殊异常。

[[5]](http://wwww.kerneltravel.net/journal/iv/syscall.htm#_ftnref5)系统调用过程可被理解成——由内核在核心态代表应用程序执行任务。

[[6]](http://wwww.kerneltravel.net/journal/iv/syscall.htm#_ftnref6)除了进程上下文外，Linux系统中还有另一种上下文——它被成为中断上下文。中断上下文不同于进程上下文，它代表中断执行，所以和进程是异步进行而且可以说毫不相干的。这种上下文中的程序，要避免睡眠，因为它们无法被抢占。





