# 2 介绍

如果你曾经选修过操作系统的课程，你应该已经知道当一个计算机程序运行时它做了什么。如果不是这样，这本书对你来说可能有些困难，所以你应该停止阅读这本书或者先去了解必要的背景知识（Patt/Patel的[PP03]和Bryant/O’Hallaron的[BOH10]都是很好的书籍）。

所以当一个程序运行时会发生什么？

其实一个正在运行的程序做的是一件非常简单的事情：执行指令。数以百万次每秒（今天，甚至数十亿次），处理器从内存中**读取**一条指令，对它进行**解码**（即计算出这是一个什么指令），并执行（即完成指令要求做的事情，如将两个数字相加、读取存储器、检验一个条件、跳转至一个函数等等）。这个指令完成之后，处理器移至下一指令，以此类推，直到程序最终完成[1]。

我们刚刚描述的是冯.诺依曼模型的基础知识。听起来很简单，不是吗？但是在这本书中，我们将学习，当一个程序运行时，许多其他各种各样的东西都将同时运行完成使系统易于使用的首要目标。

   关键问题：如何虚拟化资源
   我们将要在书中回答一个简单的主要问题：操作系统如何虚拟化资源？这是我们的核心问题。为什么操作系统要这样做不是关键的问题，因为答案应该很明显：它使得系统方便使用。因此，我们更注重方式，即操作系统通过什么机制和策略实现虚拟化，操作系统如何使其有效运行以及需要什么硬件的支持。
   我们会在阴影框里使用“关键问题”，就像这个一样，用来点出我们在建立一个操作系统时需要解决的问题。因此，在特定的主题下，你会发现一个或多个用来突出问题的关键问题（是的，这确实是复数）。当然，章节中的具体内容会给出解决方案，或者至少提出解决问题的基本要素。
   
     确实存在一个软件体负责使程序的运行变得简单（甚至允许你同时运行很多程序），允许程序间共享内存，与设备交互以及其他类似的有趣的事情。这个软件体就叫做操作系统，它负责确保系统以一种简单的方式正确有效地运行。
   
     操作系统采用的主要方式是通过一种通用技术即虚拟化。也就是，操作系统将物理资源（如处理器，存储器或磁盘）转变为它的通用的，有效的并且易于使用的虚拟形式。因此，我们有时也将操作系统叫做虚拟机。
     
     当然，为了让用户告诉操作系统做什么，使用虚拟机的特性（如运行程序，分配内存或者访问文件），操作系统也提供了一些接口（API）让用户调用。一个典型的操作系统提供了几百个系统调用供应用程序使用。因为操作系系统使用这些调用完成运行程序，访问内存和设备和其他相关的行为，我们有时也说操作系统给给应用程序提供了一个标准库。
   
       最后，因为虚拟化允许许多程序同时运行（即共享CPU），获取他们自己的指令和数据（即共享存储器）和访问设备（即共享磁盘等等），操作系统有时也被叫做资源管理者。每个CPU，存储器和磁盘都是系统的资源，操作系统的职责就是有效管理这些资源，甚至处理一些可能的目标。为了更进一步了解操作系统，让我们看一些例子。
   
     
   

