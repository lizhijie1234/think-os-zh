# 第八章 多任务

> 作者：[Allen B. Downey](http://greenteapress.com/wp/)

> 原文：[Chapter 8  Multitasking](http://greenteapress.com/thinkos/html/thinkos009.html)

> 译者：[飞龙](https://github.com/)

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

在当前的许多系统上，CPU包含多个核心，也就是说它可以同时运行多个进程。而且，每个核心都具有“多任务”的能力，也就是说它可以从一个进程快速切换到另一个进程，创造出同时运行许多进程的幻象。

操作系统中，实现多任务的这部分叫做“内核”。在坚果或者种子中，内核是最内层的部分，由外壳所包围。在操作系统各种，内核是软件的最底层，由一些其它层包围，包括称为“Shell”的界面。计算机科学家喜欢引喻。

究其本质，内核的工作就是处理中断。“中断”是一个事件，它会停止通常的指令周期，并且使执行流跳到称为“中断处理器”的特殊代码区域内。

当一个设备向CPU发送信号时，会发生硬件中断。例如，网络设备可能在数据包到达时会产生中断，或者磁盘驱动器会在数据传送完成时产生中断。多数系统也带有以固定周期产生中断的计时器。

软件中断由运行中的程序所产生。例如，如果一条指令由于某种原因没有完成，可能就会触发中断，便于这种情况可被操作系统处理。一些浮点数的错误，例如除零错误，会由中断处理。

当程序需要访问硬件设备时，会进行“系统调用”，它就像函数调用，除了并非跳到函数的起始位置，而是执行一条特殊的指令来触发中断，使执行流跳到内核中。内核读取系统调用的参数，执行所请求的操作，之后使被中断进程恢复运行。

## 8.1 硬件状态

中断的处理需要硬件和软件的配合。当中断发生时，CPU上可能正在运行多条指令，寄存器中也储存着数据。

通常硬件负责将CPU设为一致状态。例如，每条指令应该执行完毕，或者完全没有执行，不应该出现执行到一半的指令。而且，硬件也负责保存程序计数器（PC），便于内核了解从哪里恢复执行。

之后，中断处理器通常负责保存寄存器的上下文。为了完成工作，内核需要执行指令，这会修改数据寄存器和位寄存器。所以这个“硬件状态”需要在修改之前保存，并在被中断的进程恢复运行后复原。

下面是这一系列事件的大致过程：

1.  当中断发生时，硬件将程序计数器保存到一个特殊的寄存器中，并且跳到合适的中断处理器。
2.  中断处理器将程序计数器和位寄存器，以及任何打算使用的数据寄存器的内容储存到内存中。
3.  中断处理器运行处理中断所需的代码。
4.  之后它复原所保存寄存器的内容。最后，复原被中断进程的程序计数器，这会跳回到被中断的进程。

如果这一机制正常工作，被中断进程通常没有办法知道这是一个中断，除非它检测到了指令间的小变化。

## 8.2 上下文切换

中断处理器非常快，因为它们不需要保存整个硬件状态。它们只需要保存打算使用的寄存器。

但是当中断发生时，内核并不总会恢复被中断的进程。它可以选择切换到其它进程，这种机制叫做“上下文切换”。

通常，内核并不知道一个进程会用到哪个寄存器，所以需要全部保存。而且，当它切换到新的进程时，它可能需要清除储存在内存管理单元（MMU）中的数据。以及在上下文切换之后，它可能要花费一些时间，为新的进程将数据加载到缓存中。 出于这些因素，上下文切换相对较慢，大约是几千个周期或几毫秒。

在多任务的系统中，每个进程都允许运行一小段时间，叫做“时间片”或“quantum”。在上下文切换的过程中，内核会设置一些硬件计数器，它们会在时间片的末尾产生中断。当中断发生时，内核可以切换到另一个进程，或者允许被中断的进程继续执行。操作系统中做决策的这一部分叫做“调度器”。

## 8.3 进程的生命周期

当进程被创建时，操作系统会为进程分配包含进程信息的数据结构，称为“进程控制块”（PCB）。在其它方面，PCB跟踪进程的状态，这包括：

+ 运行（Running），如果进程正在运行于某个核心上。
+ 就绪（Ready），如果进程可以但没有运行，通常由于就绪进程数量大于内核的数量。
+ 阻塞（Blocked），如果进程由于正在等待未来的事件，例如网络通信或磁盘读取，而不能运行。
+ 终止（Done）：如果进程运行完毕，但是带有没有读取的退出状态信息。

下面是一些可导致进程状态转换的事件：

+ 一个进程在运行中的程序执行类似于`fork`的系统调用时诞生。在系统调用的末尾，新的进程通常就绪。之后调度器可能恢复原有的进程（“父进程”），或者启动新的进程（“子进程”）。
+ 当一个进程由调度器启动或恢复时，它的状态从就绪变为运行。
+ 当一个进程被中断，并且调度器没有选择使它恢复，它的状态从运行变成就绪。
+ 如果一个进程执行不能立即完成的系统调用，例如磁盘请求，它会变为阻塞，并且调度器会选择另一个进程。
+ 当类似于磁盘请求的操作完成时，会产生中断。中断处理器弄清楚哪个进程正在等待请求，并将它的状态从阻塞变为就绪。
+ 当一个进程调用`exit`时，中断处理器在PCB中储存退出代码，并将进程的状态变为终止。

## 8.4 调度

就像我们在2.3节中看到的那样，一台计算机上可能运行着成百上千条进程，但是通常大多数进程都是阻塞的。大多数情况下，只有一小部分进程是就绪或者运行的。当中断发生时，调度器会决定那个进程应启动或恢复。

在工作站或笔记本上，调度器的首要目标就是最小化响应时间，也就是说，计算机应该快速响应用户的操作。响应时间在服务器上也很重要，但是调度器同时也可能尝试最大化吞吐量，它是单位时间内所完成的请求。

调度器通常不需要关于进程所做事情的大量信息，所以它基于一些启发来做决策：

+ 进程可能被不同的资源限制。执行大量计算的进程是计算密集的，也就是说它的运行时间取决于得到了多少CPU时间。从网络或磁盘读取数据的进程是IO密集的，也就是说如果数据输入和输出更快的话，它就会更快，但是在更多CPU时间下它不会运行得更快。最后，与用户交互的程序，在大多数时间里可能都是阻塞的，用于等待用户的动作。操作系统有时可以将进程基于它们过去的行为分类，并做出相应的调度。例如，当一个交互型进程不再被阻塞，应该马上运行，因为用户可能正在等待回应。另一方面，已经运行了很长时间的CPU密集的进程可能就不是时间敏感的。
+ 如果一个进程可能会运行较短的时间，之后发出了阻塞的请求，它可能应该立即运行，出于两个原因：（1）如果请求需要一些时间来完成，我们应该尽快启动它，（2）长时间运行的进程应该等待短时间的进程，而不是反过来。作为类比，假设你在做苹果馅饼。面包皮需要5分钟来准备，但是之后需要半个小时的冷却。而馅料需要20分钟来准备。如果你首先准备面包皮，你可以在其冷却时准备馅料，并且可以在35分钟之内做完。如果你先准备馅料，就会花费55分钟。

大多数调度器使用一些基于优先级的调度形式，其中每个进程都有可以调上或调下的优先级。当调度器运行时，它会选择最高优先级的就绪进程。

下面是决定进程优先级的一些因素：

+ 具有较高优先级的进程通常运行较快。
+ 如果一个进程在时间片结束之前发出请求并被阻塞，就可能是IO密集型程序或交互型程序，优先级应该升高。
+ 如果一个进程在整个时间片中都运行，就可能是长时间运行的计算密集型程序，优先级应该降低。
+ 如果一个任务长时间被阻塞，之后变为就绪，它应该提升为最高优先级，便于响应所等待的东西。
+ 如果进程A在等待进程B的过程中被阻塞，例如，如果它们由管道连接，进程B的优先级应升高。
+ 系统调用`nice`允许进程降低（但不能升高）自己的优先级，并允许程序员向调度器传递显式的信息。

对于运行普通工作负载的多数系统，调度算法对性能并没有显著的影响。简单的调度策略就足够好了。

## 8.5 实时调度

但是，对于与真实世界交互的程序，调度非常重要。例如，从传感器和控制马达读取数据的程序，可能需要以最小的频率完成重复的任务，并且以最大的响应时间对外界事件做出反应。这些需求通常表述为必须在“截止期限”之前完成的“任务”。

调度满足截止期限的任务叫做“实时调度”。对于一些应用，类似于Linux的通用操作系统可以被修改来处理实时调度。这些修改可能包括：

+ 为控制任务的优先级提供更丰富的API。
+ 修改调度器来确保最高优先级的进程在固定时间内运行。
+ 重新组织中断处理器来保证最大完成时间。
+ 修改锁和其它同步机制（下一章会讲到），允许高优先级的任务预先占用低优先级的任务。
+ 选择保证最大完成时间的动态内存分配实现。

对于更苛刻的应用，尤其是实时响应是生死攸关的领域，“实时操作系统”提供了专用能力，通常比通用操作系统拥有更简单的设计。
