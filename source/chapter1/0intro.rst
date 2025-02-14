引言
=====================

本章导读
--------------------------

.. chyyuu
  这是注释：我觉得需要给出执行环境（EE），Task，...等的描述。
  并且有一个图，展示这些概念的关系。
  
本章展现了操作系统的一个基本功能：让应用与硬件隔离，简化了应用访问硬件的难度和复杂性。这也是远古操作系统雏形和现代的一些简单嵌入式操作系统的主要功能。具有这样功能的操作系统的形态就是一个函数库，可以被应用访问，并通过函数库的函数来访问硬件。

大多数程序员的第一行代码都从 ``Hello, world!`` 开始，当我们满怀着好奇心在编辑器内键入仅仅数个字节，再经过几行命令编译（靠的是编译器）、运行（靠的是操作系统），终于在黑洞洞的终端窗口中看到期望中的结果的时候，一扇通往编程世界的大门已经打开。在本章第一节 :doc:`1app-ee-platform` 中，可以看到用Rust语言编写的非常简单的“Hello, world”应用程序是如何被进一步拆解和分析的。

不过我们能够隐约意识到编程工作能够如此方便简洁并不是理所当然的，实际上有着多层硬件和软件工具和支撑环境隐藏在它背后，才让我们不必付出那么多努力就能够创造出功能强大的应用程序。生成应用程序二进制执行代码所依赖的是以 **编译器** 为主的开发环境；运行应用程序执行码所依赖的是以 **操作系统** 为主的执行环境。

本章主要是讲解如何设计和实现建立在裸机上的执行环境，从中对应用程序和它所依赖的执行环境有一个全面和深入的理解。

本章我们的目标仍然只是输出 ``Hello, world!`` ，但这一次，我们将离开舒适区，基于一个几乎空无一物的平台从零开始搭建我们自己的高楼大厦，而不是仅仅通过一行语句就完成任务。所以，在接下来的内容中，我们将描述如何让 ``Hello, world!`` 应用程序逐步脱离对编译器、运行时库和操作系统的现有复杂依赖，从脱离运行时库，到最终以最小的依赖需求能在裸机上运行。这时，我们也可把这个能在裸机上运行的 ``Hello, world!`` 应用程序称为一种支持输出字符串的非常初级的寒武纪“三叶虫”操作系统，它其实就是一个给应用提供各种服务（比如输出字符串）的库，方便了单一应用程序在裸机上的开发与运行。输出字符串功能好比是三叶虫的眼睛，有了它，我们就有了最基本的调试功能，即通过在代码中的不同位置插入特定内容的输出语句来实现对程序运行的调试。


.. chyyuu note
   
   在练习一节前面，是否有一个历史故事？
   在操作系统发展历史上，在1956年就诞生了有文字历史记录的操作系统GM-NAA I/O，并且被实际投入使用，它的一个主要任务就是"自动加载运行一个接一个的程序"，并能以库函数的形式给应用程序提供基本的硬件访问服务。

实践体验
---------------------------

本章设计实现了一个支持显示字符串应用的简单操作系统--“三叶虫”操作系统，它的形态就是一个函数库，给应用程序提供了显示字符串的函数。

获取本章代码：

.. code-block:: console

   $ git clone https://github.com/rcore-os/rCore-Tutorial-v3.git
   $ cd rCore-Tutorial-v3
   $ git checkout ch1

在 qemu 模拟器上运行本章代码，看看一个小应用程序是如何在QEMU模拟的计算机上运行的：

.. code-block:: console

   $ cd os
   $ make run

将 Maix 系列开发板连接到 PC，并在上面运行本章代码，看看一个小应用程序是如何在真实计算机上运行的：

.. code-block:: console

   $ cd os
   $ make run BOARD=k210

.. warning::

   **FIXME: 提供 wsl/macOS 等更多平台支持**

如果顺利的话，以 qemu 平台为例，将输出：

.. code-block::

   [rustsbi] RustSBI version 0.1.1
   .______       __    __      _______.___________.  _______..______   __
   |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
   |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
   |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
   |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
   | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|

   [rustsbi] Platform: QEMU (Version 0.1.0)
   [rustsbi] misa: RV64ACDFIMSU
   [rustsbi] mideleg: 0x222
   [rustsbi] medeleg: 0xb1ab
   [rustsbi-dtb] Hart count: cluster0 with 1 cores
   [rustsbi] Kernel entry: 0x80200000
   Hello, world!
   .text [0x80200000, 0x80202000)
   .rodata [0x80202000, 0x80203000)
   .data [0x80203000, 0x80203000)
   boot_stack [0x80203000, 0x80213000)
   .bss [0x80213000, 0x80213000)
   Panicked at src/main.rs:46 Shutdown machine!

除了 ``Hello, world!`` 之外还有一些额外的信息，最后关机。


.. note::

   RustSBI是啥？
   
   戳 :doc:`../appendix-c/index` 可以进一步了解RustSBI。

本章代码树
------------------------------------------------

.. code-block::

   ./os/src
   Rust        4 Files   118 Lines
   Assembly    1 Files    11 Lines

   ├── bootloader(内核依赖的运行在 M 特权级的 SBI 实现，本项目中我们使用 RustSBI) 
   │   ├── rustsbi-k210.bin(可运行在 k210 真实硬件平台上的预编译二进制版本)
   │   └── rustsbi-qemu.bin(可运行在 qemu 虚拟机上的预编译二进制版本)
   ├── LICENSE
   ├── os(我们的内核实现放在 os 目录下)
   │   ├── Cargo.toml(内核实现的一些配置文件)
   │   ├── Makefile
   │   └── src(所有内核的源代码放在 os/src 目录下)
   │       ├── console.rs(将打印字符的 SBI 接口进一步封装实现更加强大的格式化输出)
   │       ├── entry.asm(设置内核执行环境的的一段汇编代码)
   │       ├── lang_items.rs(需要我们提供给 Rust 编译器的一些语义项，目前包含内核 panic 时的处理逻辑)
   │       ├── linker-k210.ld(控制内核内存布局的链接脚本以使内核运行在 k210 真实硬件平台上)
   │       ├── linker-qemu.ld(控制内核内存布局的链接脚本以使内核运行在 qemu 虚拟机上)
   │       ├── main.rs(内核主函数)
   │       └── sbi.rs(调用底层 SBI 实现提供的 SBI 接口)
   ├── README.md
   ├── rust-toolchain(控制整个项目的工具链版本)
   └── tools(自动下载的将内核烧写到 k210 开发板上的工具)
      ├── kflash.py
      ├── LICENSE
      ├── package.json
      ├── README.rst
      └── setup.py


本章代码导读
-----------------------------------------------------


操作系统虽然是软件，但它不是常规的应用软件，需要运行在没有操作系统的裸机环境中。如果采用通常编程方法和编译手段，无法开发出操作系统。其中一个重要的原因编译器编译出的应用软件在缺省情况下是要链接标准库（Rust编译器和C编译器都是这样的），而标准库是依赖于操作系统（如Linux、Windows等）的。所以，本章主要是让读者能够脱离常规应用软件开发的思路，理解如何开发没有操作系统支持的操作系统内核。

为了做到这一步，首先需要写出不需要标准库的软件并通过编译。为此，先把一般应用所需要的标准库的组件给去掉，这会导致编译失败。然后在逐步添加不需要操作系统的极少的运行时支持代码，让编译器能够正常编译出不需要标准库的正常程序。但此时的程序没有显示输出，更没有输入等，但可以正常通过编译，这样就为进一步扩展程序内容打下了一个 **可正常编译OS** 的前期基础。具体可看 :ref:`移除标准库依赖 <term-remove-std>` 一节的内容。

操作系统代码无法象应用软件那样，可以有方便的调试（Debug）功能。这是因为应用之所以能够被调试，也是由于操作系统提供了方便的调试相关的系统调用。而我们不得不再次认识到，需要运行在没有操作系统的裸机环境中，当然没法采用依赖操作系统的传统调试方法了。所以，我们只能采用 ``print`` 这种原始且有效的调试方法。这样，第二步就是让脱离了标准库的软件有输出，这样，我们就能看到程序的运行情况了。为了简单起见，我们可以先在用户态尝试构建没有标准库的支持显示输出的最小运行时执行环境，比较特别的地方在于如何写内嵌汇编完成简单的系统调用。具体可看 :ref:`构建用户态执行环境 <term-print-userminienv>` 一节的内容。


接下来就是尝试构建可在裸机上支持显示的最小运行时执行环境。相对于用户态执行环境，读者需要能够做更多的事情，比如如何关机，如何配置软件运行所在的物理内存空间，特别是栈空间，如何清除 ``bss`` 段，如何通过 ``RustSBI`` 的 ``SBI_CONSOLE_PUTCHAR`` 接口简洁地实现的信息输出。这里比较特别的地方是需要了解 ``linker.ld`` 文件中对OS的代码和数据所在地址空间布局的描述，以及基于RISC-V 64的汇编代码 ``entry.asm`` 如何进行栈的设置和初始化，以及如何跳转到Rust语言编写 ``rust_main`` 主函数中，并开始内核最小运行时执行环境的运行。具体可看 :ref:`构建裸机执行环境 <term-print-kernelminienv>` 一节的内容。