## 操作系统拓展实验第16周报告

匡正非 2015011273 计52



### 工作概述

​	在利用Syzkaller对uCore进行检测，并找到该系统中的数个bug之后，下一步的工作重心主要放在了对uCore的SMP多核系统版本进行bug检测上。

​	相比与单核的操作系统而言，多核的操作系统有着更多难以发现并调试的bug——并发性bug。这类bug不仅依赖于代码的构成，同时也依赖于不同核之间指令的执行顺序(instruction interleaving)关系，因此，它们往往具有难以确认位置、确认起因、甚至难以复现（尤其在真机上运行时）的特性。在单核情况当中，操作系统的bug往往通过VMM模拟来解决，而目前的主流VMM（如QEMU）也已经能够提供多核SMP的支持，但尽管如此，在面对并发性bug的时候，由于VMM模拟指令的执行顺序仍然具有不可控性，且指令顺序的信息对用户并不可见，目前已有VMM的支持显得相当疲软。

​	为了能够更加高效、合理地解决uCore中已经存在，或仍未找到但可能存在的bug，我们对一篇发表自OSDI2014的论文 “SKI: Exposing Kernel Concurrency Bugs through Systematic Schedule Exploration”进行了研究，并探索了将其运用与uCore上的可行性。在短短的三周之内，我们通过该论文对应的开源代码，我们深入了解了该算法的运行方式，并已经成功让该测试系统的Target OS部分在uCore上成功运行，证明了移植是可行的，但由于其代码年度过于久远，已经无法提供对当前情况的支持，因此我们只能放弃完成对uCore检测的最初计划，仅在此提供对于这篇论文及代码的分析报告。



### SKI

####核心思想

​	SKI这一算法的提出，旨在于解决之前提到的调试并行性bug的两大问题：如何控制指令执行顺序、以及如何找出能引发出错的指令执行顺序。传统的并行性测试主要通过压力测试(stress testing)的方式进行（例如我们在之前完成的fuzzing测试），即不断地要求系统运行测例寻找可能存在的bug，但由于并发性bug往往只可能存在于特定的执行顺序中，而压力测试采用的是随机查找的方式，因此其作用是十分有限的。为此，SKI采用了另一种新的测试手段——系统性测试(systematic testing)来解决这一问题，即对于同一个测试案例，通过调整不同的指令执行顺序并多次运行，从而找到可能发生的错误。

​	显然，对于一个多核操作系统而言，只有VMM能够知道并控制指令的执行顺序，所以SKI的主体部分是通过修改VMM来实现的。而对于测试程序来说，为了秉承尽量不修改测试对象的原则，SKI设计了一套供用户使用的测试框架，其中包含了一套利用hypercall与VMM进行交互的系统，使用起来灵活、方便。

​	另外，虽然SKI的算法核心在于制造不同的指令顺序，看起来十分简单，但事实上，论文的很大一部分内容其实是为了提高性能而加入的各种各样的优化，其中既有涉及到算法的，也有涉及到代码实现的。巧妙的优化也同样是这篇文章的重点之一。下面将更加具体地说明算法的设计以及这些优化。



#### 控制执行顺序

​	对于VMM而言，直接操纵不同的核的执行顺序是比较简单的，只需要决定合适开关各个核即可。但对于一个操作系统来说，直接影响其工作结果的实质上是进程之间的执行顺序，而由于进程随时有可能从一个核转移到另一个核上，所以对核的控制并不能完全达到控制操作系统的要求。幸运的是，主流的操作系统中普遍支持CPU亲和力（affinity）的设置，即可以将进程绑定到特定的一个核上运行，这样就解决了核与进程之间的映射关系。

​	不过这一做法还有两个问题需要得到解决：除了一般的用户进程以外，内核进程的操控往往是不受控制的，所以SKI在此无法对内核进程进行控制；另一方面，在实际系统运行的过程中随时可能产生大量的系统中断，这些中断可能出现在任意一个核上，剥夺该核运行用户程序的权利，转而运行中断处理程序。考虑到中断的执行时间也是决定并行性bug一个可能来源，SKI同样加入了对于中断执行的控制。具体来说，SKI将中断和进程统一了起来（文中称为context），当系统发生中断时会相应地生成一个context。而对于每一个CPU而言，它既有可能处在用户程序中，也有可能处在中断处理中，因此在实际的VMM中对于CPU的控制除了将其开启关闭以外，还需要加入两个状态之间的变化。

​	至此，SKI已经实现了对用户进程和中断处理的控制。下面需要考虑的是如何枚举各个进程之间的执行顺序。

​	

#### 生成执行顺序（schedule）

​	SKI采用了一种名为PCT的算法进行执行顺序的生成，这一算法从原理上来说也比较简单。它给每一个进程赋予了一个优先级(priority)，在程序开始的时候随机初始化，然后选择优先度最高的一个开始运行。在进程运行的过程中，可能会出现一系列的转换点（reschedule point），这些点出现的时候该进程的优先度会随机性的下降，当低于另外某个进程的时候，便进行进程切换。问题的关键在于转换点的位置设置。

​	文中最初采用的方案是：假设进程需要运行的指令期望个数是$k$的话，则在$[0 - k]$中任意选择$p$个位置作为转换点。但这种方式实际上带来的可能的执行顺序数目是惊人的：即使$p=1$，其可能性也和指令的个数成正比。事实上，在多核的处理机制上，由执行顺序带来的问题几乎都是由数据竞争而引起的，所以只有不同核访问同一个内存地址的时候才有可能出现麻烦。SKI正是利用了这点性质，规定只有关于共享数据访问的指令才有可能成为转换点，从而大大减小了测试集合的大小。关于如何找到共享数据访问的指令，SKI采用了自适应的方式：在不断重复运行测试的情况下，动态统计所有访问到了共享数据的指令，然后将这些指令累加到一个集合中，作为下一次测试中可能的转换点集合。

​	最后，还有一些点虽然不属于共享访问的区域，但他们仍然需要被制定为转换点：进程陷入等待的时候。SKI是基于VMM的，因此无法从内核得知何时出现了等待，因此只能通过一些程序运行时出现的特征（heuristic）进行判断。具体而言分为以下四种：

* Halt：程序中出现了Halt指令
* Pause：程序中出现了Pause指令
* Loop：程序中在一段时间内反复访问同一个内存地址，用于判断是否在等待某个锁
* Starvation：程序在一段时间内反复执行同一段代码，作用和Loop一样

​	这四种方式都只是通过对运行指令的判断来实现的，所以与内核完全无关。至此，关于SKI的算法介绍已经基本完成，下面介绍一下SKI在实现上的一些技巧。



#### 虚拟机快照（Snapshot）

​	之前已经提到过，SKI在测试时会产生很多不同的执行顺序，而测试的程序与环境是始终保持不变的。如果每次测试时，都需要重新搭建Target OS的测试环境的话，这是十分浪费的。所以，SKI在修改QEMU的同时加入了保存VM快照的功能，能够将Target OS运行的任意时刻的所有状态（寄存器、内存、文件系统）保存下来。有了这一功能，SKI在正式测试开始前会先进行一次预测试，将系统加载完毕，测试即将开始时的虚拟机状态保存下来，存到Host OS的tmpfs中去，这样便能在之后的测试中很快的做到复原了。

​	更加巧妙的是，由于这一快照存放在tmpfs中（内存中），因此可以通过fork+copy on write的方式快速复制，这又为测试节省了大量的时间。



#### Bug测试器

​	为了全方面探索系统运行时是否出现了bug，SKI使用了三种的方法来检测bug：第一，如果出现的bug直接导致了系统崩溃，那么通过Assertion可以在VM的输出中可以直接探测到；第二，如果系统没有崩溃但结果出错，可以在程序结束后在用户空间中进行检测；第三，针对数据竞争问题，SKI实现了一个检测所有内存访问情况的探测器。这种做法虽然可能会导致一些false positive的出现（一些数据竞争也未必会对程序造成影响），但是却是能够为找到并行性bug提供更多的帮助。



### SKI代码分析、移植可行性分析

####SKI源码结构

​	SKI的代码在2015年时已在github上开源，整体对于QEMU的改动有13000+行，除此之外还有用于运行的各类脚本、测试框架等数千行。

​	在QEMU当中，SKI的改动又可以分为两个部分：一部分集中于target-i386文件夹中，主要是SKI需要涉及到i386架构的模块，包括VMSTATE的更新，解码二进制指令（disas_insn）时要加入的新的处理函数（helper），以及对于interrupt处理中插入的处理等。而第二部分在根目录下，其中既对原有的qemu代码做了改动，也添加了很多新的代码，其中新添的文件文件名普遍以ski开头，它们构成了SKI的主要功能部分。由于工程量实在过于庞大，且作者没有提供任何文档，我们还基本没有理解这一部分代码的实际结构，目前只对一些文件有了部分了解：如ski-hyper.c用于读取从target-os发来的hypercall，ski-priority.c用于维护PCT算法中各个进程的优先级，ski-race-detector.c用于检测data-race的bug等。对于QEMU本身文件的改动则更加复杂，直至目前我们基本还没有进行研究。

​	而对于测试代码，官方给出的测试是在fsstress测试系统上进行改动而成的，因此代码量上也并不小；但由于作者在这方面给出了较为详细的文档，使得我们在阅读代码上变得轻松很多。虽然在论文中作者提到对操作系统的支持非常小，但事实上由于测试的对象是linux、freeBSD等较为成熟的系统，因此在实现上还是使用了一些复杂的功能，如ssh、bash等。我们目前已经完全掌握了具体测试的流程，下面就详细介绍一下。



#### 测试部分

​	在官方给出的测试系统中，既有Host需要运行的代码（主要由脚本构成），也有Target需要支持的代码（主要由脚本、C构成）。在Host代码中主要有两个脚本：create-snapshot（用于生成快照）和run-ski（用于开始测试），而Target代码中较为关键的部分可以分为四块：用于支持target和VMM之间hypercall通信的库（下称hyper库）、用于进行测试的文件（下称主文件）、以及一个用于搭建环境、开始测试的脚本（下称run脚本）。整个测试的流程是这样的：

1. 用户在host中运行create-snapshot脚本
2. create-snapshot脚本编译Target测试代码中C的部分
3. create-snapshot脚本利用改装后的QEMU运行Target
4. Target在加载完毕后，向Host发送"Guest finished booting"信息
5. create-snapeshot监听QEMU输入，当发现该消息后，将编译好的测试文件、脚本通过ssh发送至Target中，并命令其运行run脚本，然后继续监听
6. run脚本启动多个测试进程运行C语言编写的测试程序，每个进程对应一个核
7. 测试程序运行后，首先设置与自己对应的CPU的Affinity，然后开始第一次测试通过hypercall向QEMU发送测试即将开始指令，然后等待其他测试进程完成初始化
8. 所有进程都完成初始化后，一起开始测试
9. 进程测试完成后，通过hypercall向qemu发送测试完毕信号，然后等待其他进程完成
10. 所有进程测试完毕后，一同退出。然后run脚本向Host发送“Guest finished snapshot”指令
11. create-snapshot在收到此指令后，根据之前hypercall的情况建立snapshot并保存起来
12. create-snapshot结束，开始运行第二个脚本run-ski。该脚本除了设定环境以外，只有一条以之前snapshot作为输入启动QEMU的指令。之后的测试便都是在QEMU中进行，通过不断创建执行顺序、复原snapshot、然后重复上述的8-10过程来寻找bug。

​	可以说，上述的过程都仅仅利用了linux最基本的一些功能。但是，由于这些功能对于uCore来说仍然过多，且整个流程也较为繁琐，因此我们接下来还对其做了不少改动。



####移植到uCore

​	在经过将近一周的努力之后，目前我们已经将测试程序跑在uCore之中。首先，我们去掉了ssh部分，将测试集直接编译进了uCore的image中，这和我们在syzkaller中做的改动是类似的；其次，我们删掉了run脚本，并将其功能转移到了c语言编写成的测试例子中去，利用fork函数替换掉了bash中直接创建进程的命令；最后，我们对C代码进行了通篇的移植，删掉或修改了不少不存在的库，使得其能够运行在uCore的环境之下。在C代码中，hyper库的修改是尤其关键的，而这其中有一个很重要的结构困扰了我们一段时间——barrier。这是一个类似于管程的结构，其作用是为了在测试中让各个进程能够达到之前所说的两次同步。barrier运用到了条件变量，这在uCore的用户库中并没有直接实现（uCore的用户库只有信号量，而经过和其他组的协商，得知目前拥有smp的uCore只提供了spinlock），所以需要在用户程序中进行手动实现。

​	在uCore上的测试程序运行起来之后，我们分析出了uCore运行SKI目前需要拥有的功能：

* 能够让进程自我设定和CPU的亲和度，以此绑定到固定的核上
* 能够让进程执行mlock操作，避免其占用的内存被交换出去。这一点目前可以通过关闭uCore的swap机制解决
* 能够实现一个spinlock

​	这三点功能中，后两点已经有了解决方案，但第一点要求需要修改kernel，这样就打破了SKI不动kernel的宗旨。不过好在目前其他组的实现当中，有保证进程在被唤醒的时候不切换核的功能，这样实际上也相当于解决了问题。不过总而言之，无论是否最终要修改内核代码，要在uCore上运行起SKI的功能看来是完全可行的。



#### 目前的主要困难与展望

​	虽然SKI对于Target要求较少，但其对于QEMU的改动可以说相当庞大。十分不幸的是，目前SKI是基于QEMU的1.0.1版本完成的，这是一个7年前的版本，仅支持i386（甚至不支持amd64），且因为兼容性的问题无法在目前的ubuntu上运行。另一方面，目前uCore已有的smp还是在rv64架构上完成的，无论是将其移植至i386，还是将SKI移植至rv64，都是一项巨大的工程。另外，要对QEMU进行跨平台移植，首先就需要将其移植到一个更新的版本上来，为此我咨询过该论文的第一作者，得到的回复是：这是一个相当困难的过程，因为许多论文中提到的优化在实现中都利用到了QEMU制定版本的特性。所以，想要让SKI在uCore上运行，虽然从理论上来看着实可行，但却依然有着很高的阻碍。

​	关于改动QEMU的另一大困境，是这一工程并没有提供任何文档（除了一个运行指南以外），代码中的注释也相当之少，导致项目难以理解。而更为致命的是，这份代码本身就有一部分的缺失，没有提供target运行时的image，所以想要直接运行起来复现结果也是无法做到的。在这种情况下，想要开始一点点对QEMU作出改动，就已经需要做大量的前期工作了。

​	不过，我们仍然可以展望在SKI的情况下，能够为uCore的发展提供什么样的更好的帮助。在其他组实现uCore的smp系统时，出现了很多各种各样的并行性bug，这些bug最终都是通过肉眼调试完成的。但是，如果可以获得一个确定的执行顺序完整复现的话，可以让这些bug的调试变得简单很多。但SKI的功能远不于此，它的主要功能实在足够复杂的系统当中找出全新的bug，这些bug在平常的模拟运行中可能并不会出现，但当系统移动到真机上、或是增加了核心个数，或是单单因为增加了一些毫无关系的代码（例如print）时可能会突然出现，使得开发者猝不及防。为此，我们需要一个相对更加复杂的测试程序，它可以是一个驱动、一个新的文件系统，也可能就是一个普通的大型用户程序。

​	另一方面，我们畅想是否能将SKI与我们之前完成的Syzkaller结合起来，变成一个更加强大的工具。一方面，syzkaller中的executor本身即是一个多进程的大型程序，我们可以尝试将其移植在多核情况并进行相应的测试；另一方面，syzkaller中实际还提供了对于多线程的处理，虽然我们在目前的版本中删去了这一功能，但在多核的情况下，利用其进行多核的测试也是十分有必要的。如果能够将syzkaller的多线程测试和SKI结合起来，那么将是一个非常强大的测试多核系统的工具。至于可行性，因为SKI只对QEMU进行了改动，而syzkaller中对QEMU的依赖只有对kAFL的使用，将其替换成pipe即可免除这一部分，所以两者改动的代码并无交集。唯一的问题在于两个算法之间的协调：SKI使用的是不断重启QEMU，运行同一个测试的方式，而syzkaller采用的则是不断输入测试，一直运行同一个target的方式，在这两个方式中我们可能需要做出一些折衷，例如让syzkaller生成一个测例之后，由SKI完成一定次数的循环，然后再交由syzkaller进行下一个测例的方式。

​	无论如何，尽管我们最终没能达到我们想要的目标：将SKI在uCore中成功运行，但是我们对于这一算法已经有了相当深入的了解，也充分意识到了它在uCore上、在和syzkaller结合上拥有着的十足潜力。在此衷心希望未来能够看到这一项目能够继续发展下去，从而作出更加强大、和操作系统解耦度更高的bug测试工具。

​	

