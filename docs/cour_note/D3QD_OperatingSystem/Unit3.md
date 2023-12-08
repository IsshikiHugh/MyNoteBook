# Unit 3: 内存 | Memory [完成一半]

!!! info "导读"

    内存是计算机中最重要的部件之一，在 Von Neumann 架构中，内存是程序和数据的载体，也是 CPU 访问数据的重要途径（CPU 能够直接访问的存储结构一般只有寄存器和内存，以及作为中介的缓存）。此外，CPU 执行的**指令(instructions)**，只有在物理内存中时才能被执行。

    而我们知道，内存 I/O 通常是比较慢的，如果再进一步对内存之外的存储设备做 I/O（内存毕竟也是有限的），则会更慢。因此，就像榨干 CPU 的性能一样，我们也要尽可能地利用好内存。

    除了性能，内存还需要实现一些保护措施，防止程序越界访问内存，或者程序之间互相干扰。

    同时，由于计算机运行程序是一个动态的过程，而我们使用内存往往需要的是连续的、大块的内存，所以在运行过程中如何保证内存的分布是相对完整的，也是一个重要的问题。

    在引入[帧 & 页](#帧--页){target="_blank"}设计后，我们不再需要以进程为单位去观察内存，而是以页为单位，这意味着粒度更小，我们可以更加灵活地去管理内存。

    在[交换技术](#交换技术){target="_blank"}的支持下，**不是正在被使用的**虚拟内存可以**实际**被映射到物理内存或[后备存储](#交换技术){target="_blank"}中。更具体的来说，我们会将“暂时用不到”的东西暂放在[后备存储](#交换技术){target="_blank"}中，而在需要的时候将它们换到物理内存中。而接下来我们要关注的重要问题就是，这些操作具体如何执行、如何优化。
    
## 内存基础设计

### 内存保护

每一个进程在内存中都应当有一块连续的内存空间，而单个进程应当只能访问自己的内存空间，而不能访问其他进程的内存空间。这就是内存保护的基本要求。

我们通过引入 base 和 limit 两个寄存器来实现框定进程的内存空间，当前进程的内存空间始于 base 寄存器中存储的地址，终于 base + limit 对应的地址，即：

<figure markdown>
<center> ![](img/27.png) </center>
A base and a limit register define a logical address space. (left)<br/>
Hardware address protection with base and limit registers. (right)
</figure>

两个特殊的寄存器只能由内核通过特定的特权指令来修改。而内存的保护，通过**[内存管理单元(memory management unit, MMU)](#MMU){target="_blank"}**来实现，MMU 会在每次访问内存时，检查访问的地址是否在 base 和 limit 寄存器所定义的范围内，如果不在，则会产生一个异常，中断程序的执行。

### 内存绑定

我们在[总览#链接器和装载器](./Unit0.md#链接器和装载器){target="_blank"}中提到过，静态的代码程序成为动态的进程，可能会需要图中这么几步。

<center> ![](img/28.png){ width=40% align=right} </center>

具体来说有三个阶段：编译时间(compile time)，装载时间(load time)和执行时间(execution time)。而内存也分三种：符号地址(symbolic addresses)，可重定位地址(relocatable addresses)（类似于一种相对量）和绝对地址(absolute addresses)。

- 通常来说，在 compile time，compiler 会将代码中的 symbol 转为 relocatable addresses；而如果在 compile time 就知道了进程最终会被安置在何处，那么在 compile time 就将 symbol 转为 absolute addresses 也是可能的，只不过如果此时起始地址发生改变，就需要重新编译。
- 而一般在 load time，relocatable addresses 会转为 absolute addresses，当进程起始地址发生改变时，我们只需要重新装载即可。
- 如果进程在 execution time 时，允许被移动，那么可能从 relocatable addresses 转为 absolute addresses 这一步就需要延迟到 execution time 来执行。

### 动态装载

由于引入了多道技术，操作系统的内存中可能同时存在多个进程。为了更加灵活地使用内存资源，我们引入**动态装载(dynamic loading)**机制。

动态装载指的是，如果一个例程还没有被调用，那么它会以**可重定位装载格式(relocatable load format)**[^5]存储在磁盘上；当它被调用时，就动态地被装载到内存中。即，例程只有在需要的时候才被载入内存。对于大量但不经常需要访问的代码片段（例如错误处理代码），这种方式可以节省大量的内存空间——这种只有偶尔会被访问的代码也不应当长久地占有内存。

> 需要注意的是，动态装载并不需要操作系统的支持，而是由开发者来负责实现。

### 动态链接和共享库

我们在[总览#链接器和装载器](./Unit0.md#链接器和装载器){target="_blank"}中已经谈论过动态链接了。而能被动态链接的库就被称为**动态链接库(dynamically linked libraries, DDLs)**，由于它们可以被多个进程共享，所以也被称为**共享库(shared libraries)**。

> 区别于动态装载，动态链接需要操作系统的支持。

### 连续分配及其问题

包括操作系统本身，内存中能存下多少东西，决定了操作系统能同时运行多少进程。而进程需要的内存需要是连续的，而内存的分配于释放又是个动态的过程，所以我们需要想一个办法高效地利用内存空间。

<a id="why-continuous"/>

!!! question "为什么进程所需要的内存是连续的？"

    请读者思考，为什么装载进程所需要的内存需要是完整、连续的，而不能是东一块而西一块的呢？
    
    或者说，如果你认为它可以不连续，为了实现让它能正常运作，你可能需要哪些措施呢？你的设计相比使用朴素的连续内存分配，有什么优势和劣势呢？

    ??? success "提示"

        Von Neumann 架构中，CPU 和内存是如何互动，从而实现其功能的？关注取指过程和汇编中的地址操作！

        - 参考阅读：[Why does memory necessarily have to be contiguous? If it weren't, wouldn't this solve the issue of memory fragmentation?](https://stackoverflow.com/questions/73197597/why-does-memory-necessarily-have-to-be-contiguous-if-it-werent-wouldnt-this){target="_blank"}

通常来说，主存会被划分为用户空间和内核空间两个部分，后者用于运行操作系统软件。主流操作系统倾向于将高位地址划为给操作系统，所以我们此处的语境也依照主流设计。

在连续内存分配(contiguous memory allocation)问题中，我们认为所有进程都被囊括在一段完整的内存中。而在内存分配的动态过程中，整个内存中空闲的部分将有可能被分配给索取内存的进程，而被分配的内存在释放之前都不能被分配给其它进程。在进程执行完毕后，内存会被释放，切我们对于进程何时释放内存不做假设。

最简单的是一种**可变划分(variable partition)**的设计，即不对内存中的划分方式做约束，只要是空闲且足够大的连续内存区域都可以被分配。但我们可以想象，在内存被动态使用的过程中，原本完整的内存可能变得支离破碎。如果我们记一块连续的空闲内存为一个 hole，则原先可能只有一个 hole，而在长时间的运行后，内存中可能存在大量较小的，难以利用的 holes。这就是**外部碎片(external fragmentation)**，在最坏的情况下，每个非空闲的内存划分之间都可能有一块不大不小的 hole，而这些 hole 单独来看可能无法利用，但其总和可能并不小，这是个非常严重的问题。

<figure markdown>
<center> ![](img/30.png) </center>
Variable partition. 1 hole to 2 holes.
</figure>

但是显然我们不能频繁地要求操作系统去重新整理内存，所以我们需要想办法来减少外部碎片的产生。我们考虑三种分配策略：

- First Fit
- Best Fit
- Worst Fit

关于这三个是什么，可以参考[这篇 ADS 笔记](../D2CX_AdvancedDataStructure/Lec11#online-first-fit-ff.md){target="_blank"}。

实验结果表明，FF 和 BF 的速度都比 WF 快，但通常 FF 会更快一些；而看内存的利用效率，两者则没有明显的区别，但是 FF 和 BF 都深受外部碎片之害。

除了 variable partition 的设计以外，还有一些别的设计，例如**固定划分(fixed partition)**，内容比较简单我就不介绍了。读者可以通过 [xxjj 的笔记](https://xuan-insr.github.io/%E6%A0%B8%E5%BF%83%E7%9F%A5%E8%AF%86/os/IV_memory_management/9_main_memory/#921-fixed-partition){target="_blank"}来做一些了解。

在 fixed partition 中，还会产生另外一种碎片，叫**内部碎片(internal fragmentation)**，它指的是在类似 fixed partition 的策略中，操作系统分配给进程的内存往往是成块的，这就会导致需求的分配量大于进程实际需求量，而那些被分配了但实际闲置的内存，就被称为内部碎片。我们之后介绍的分页技术，也同样会产生内部碎片。

### 物理地址和虚拟地址

为了让内存具有更强的灵活性，我们区分内存的**物理地址(physical address)**和**虚拟地址(virtual address)**，后者也叫**逻辑地址(logical address)**。

物理地址实际在内存设备中进行内存寻址，主要反应内存在硬件实现上的属性；而 CPU 所使用的一般指的是虚拟内存，主要反应内存在逻辑上的属性。物理地址和虚拟地址存在映射关系，而实现从虚拟地址到物理地址的映射的硬件，是**内存管理单元(memory management unit, MMU)**<a id="MMU"/>，除了是实现虚拟地址->物理地址的映射外，MMU 还负责内存访问的[保护](#内存保护){target="_blank"}。我们在之后会将了解到，[TBL](#硬件支持){target="_blank"} 也属于 MMU 的一部分。

<figure markdown>
<center> ![](img/29.png){ width=60% } </center>
Dynamic relocation using a relocation register.
</figure>

物理地址和虚拟地址的区分让使得用户程序不再需要（也不被允许）关注物理地址。此外，通过利用虚拟地址和物理地址的灵活映射，我们可以通过分页来实现良好的内存管理。

!!! tip "头脑风暴"
    
    请读者尝试设想，利用虚拟地址和物理地址的映射关系，我们能做到哪些事？
    
    ??? success "提示"
    
        1. 考虑数学上如何分类“函数映射”；
        2. 考虑如何实现地址连续；

!!! tip "头脑风暴"
    
    请读者思考，物理地址和虚拟地址的长度需要一样吗？

<a id="virtual-address-space"/>
一个进程的**虚拟地址空间(virtual address space)**，指的是在虚拟内存的语境下，进程的内存结构。通常进程在虚拟地址空间中的[大致结构](./Unit1.md#进程的形式){target="_blank"}和地址分布都是相同的，例如可能都是从 0 地址开始放 text 段，栈底一般都在末尾等——这就意味着进程的虚拟地址空间应当是**互不相关**的，由将若干互相隔离的虚拟地址空间映射到各自的物理地址这个任务，则由 [MMU](#MMU){target="_blank"} 完成。（在我们之后介绍了[页表](#页表){target="_blank"}后，这意味着每个进程都应当有自己的页表。）

## 分页技术

分页技术想要解决的问题是减轻进程“必须要使用连续内存”这一限制。我们在[前面的思考题](#why-continuous){target="_blank"}中已经提到，需要使用连续内存是需要一种逻辑上的连续，因此，在[物理地址和虚拟地址](#物理地址和虚拟地址){target="_blank"}的语境下，我们只需要保证虚拟地址是连续的即可。当然，这并不意味着物理地址的连续就是毫无意义的了，物理地址的连续是实际上提供高效内存访问的基础。

> 显然这里的 “page” 是基于 [Definition 2](#page-frame-def-2){target="_blank"}。

### 基本设计

换句话来说，我们不再严格需要物理地址也是完整的、大块的、完全连续的了。听起来 external fragmentation 的问题已经解决了，貌似我们只需要每次从里面慢慢捡垃圾，凑出一整块就行了……

嘿！想的有点太美了！虽然逻辑上物理地址不需要连续，但过于稀碎的物理地址会导致内存访问缓慢，捡垃圾凑出来的虚拟内存块也像垃圾一样食之无味。内存映射关系过于琐碎，虽然灵活性上升但效率下降；如果映射关系较为大块、完整，那么效率上升但灵活性下降。我们需要一个中庸的方案。

#### 帧 & 页

因此，我们将两者的优点合并，我们将物理内存划分为固定大小的块，称为**帧(frames)**（类似于 fixed partition），每个帧对应虚拟地址中等大的一块**页(pages)**，用这些帧来作为连续的虚拟地址的物理基础，用虚拟的页号来支持连续虚拟地址（马上就会细说），这样保证了在一定限度内页分配的自由度，利用了虚拟地址的灵活性；又保证了内存相对来说还是成块连续的，提供了物理地址连续的高效性。而帧与页的对应关系，是通过**页表(page table)**来实现的，在页表中，实际上是一个以页号为索引的帧号数组，按照页号顺序排列，因此，页号就是对应的表项在数列中的位次。

???+ key-point "pages v.s. frames"

    !!! warning "下面的内容是在扣定义扣字眼，如果读者认为这毫无意义，可以直接跳过，但我个人认为这些事是构成流畅逻辑的一个基础。"

    虽然我们已经给出了明确的 page 和 frame 的定义，但现实很混乱，我主要查找到关于 page 和 frame 有连套不同的定义。

    <a id="page-frame-def-1"/>
    ???+ definition "Definition 1"

        参考这个[链接](https://cs.stackexchange.com/a/85626){target="_blank"}，这种流派的定义就是上面提到过的这种：

        1. page 表示**虚拟内存中**的完整一块；
        2. frame 表示**物理内存中**的完整一块；

        > 这里我们抓住主要矛盾，不准确描述是怎样“完整一块”，主要区别在于加粗部分。

    <a id="page-frame-def-2"/>
    ???+ definition "Definition 2"

        参考这个[链接](https://cs.stackexchange.com/a/11670){target="_blank"}，这种流派的定义不同，在这套定义里，**准确的 page** 和 frame 不是对等的概念，而是说：

        1. 作为缩写的 page 指代 virtual page，即虚拟内存中的一块；
        2. 作为缩写的 frame 全称是 page frame，也被定义为 physical page；

        !!! warning "为了避免歧义，本 block 中，我们只使用 page，virtual page，physical page 这三个术语！"
        
        在这个定义里，page 表示的实际上是抽象的数据块（注意，不是虚拟的），换句话来说：
        
        1. page 的本质是“数据信息”；
        2. physical page 是 page 在物理内存上的实际存储形式；
        3. virtual page 是 page 的在虚拟内存上的逻辑映象，也 physical page 的一个 view；

        可以发现，虽然用词改变，但是 “physical page” 和 “virtual page” 的关系和之前是一样的，只是 “page” 这个词的含义不一样了。

        所以最违和的就是，作为缩写的 page 和准确的 page 的含义是不一致的，甚至区别巨大。所以我不喜欢这个定义。但是没办法，paging 技术的命名反而就是基于这套定义的，这里大概存在一个非常恶心的历史遗留问题在，请读者留个心眼，在之后的内容中仔细辨别。

回忆[虚拟地址空间](#virtual-address-space){target="_blank"}的相关概念，**每个进程应当都有自己的页表**，即我们称页表是 per-process data structures。

!!! tip "头脑风暴"

    由于帧和页的大小是固定的，所以虽然理论上我们需要的是每一帧的首地址，但所谓的“首地址”实际上是 $m * FrameSize$，因此，只需要用 $m$ 就可以唯一确定了（就像数组的 random access）。

    现在，请读者思考，当 $FrameSize = 2^k$ 时，会有怎样良好的性质？
    
    ??? success "提示"

        联系[页 & 虚拟地址](#页--虚拟地址){target="_blank"}，考虑整个地址的二进制表示中表示页号的部分在整个二进制串中的构成！

<figure markdown>
<center> ![](img/31.png){ width=50% } </center>
Paging model of logical and physical memory.<br/>
以 page table 中的第一项为例：<font color="blue">0</font>:<font color="green">5</font> 表示虚拟地址中的第 <font color="blue">0</font> 页对应物理地址中的第 <font color="green">5</font> 帧。
</figure>

!!! tip "头脑风暴"

    不知道你看了这个寻址模式是否感觉有些微妙的点？在继续之前，请尝试发现这个违和的地方在何处。

    ??? success "黄油猫！"

        <center> ![](img/36.png){ width=60% } </center>

        首先一个结论是，我们显然不能拿着虚拟地址去找页表，因为会陷入：『找页表需要访问页表的物理地址、找虚拟地址对应的物理地址需要页表、找页表需要访问页表的物理地址……』的黄油猫[^3]中。

        就 RSICV 来说，你可以在实验三的手册里找到一段描述 [satp 寄存器](https://zju-sec.github.io/os23fall-stu/lab3/#risc-v-virtual-memory-system-sv39){target="_blank"}的部分。

        我们可以注意到，satp 寄存器的末尾存储的是物理页号，也就是帧号，所以非常显然的，我们需要特别地去存储页表的物理地址信息，并用这个物理地址来访问页表。

#### 页 & 虚拟地址

我们来看虚拟地址是如何在连续性上发挥作用的：一个程序载入内存可能需要多个页，这些页按顺序被分配了**页号(page number)**，实际使用的地址会落在某一页中，就通过 page number 进行索引。而由于一页中包含一大块内存（page size 常常取 4KB），而我们所需要寻的址总是其中的一个 Byte，所以我们需要一个**页内偏移(page offset)**来索引我们所需要的地址在页中的位置，对于 page size 为 4KB 的页，page offset 需要有 $\log_2{4096} = 12$ 位。

因此，实际在 paging 逻辑中，一个虚拟地址的可以被分为两个部分：

```
┌───────────────┬──────────────┐
│page number    │page offset   │
└───────────────┴──────────────┘
 p: m-n bits     d: n bits
```

显然，由于帧和页是一体两面、一一对应的，所以单个页内的连续内存页对应帧上的连续内存。使用 page offset 来标识页内地址，实际就得到了目标物理地址相对于帧中起始地址的偏移量。而对于页间的地址，假设页末地址是：

```
┌───────────────┬──────────────┐
│page number    │11111111111111│
└───────────────┴──────────────┘
 p: m-n bits     d: n bits
```

由于虚拟地址表现上还是个正常的二进制数，所以其下一个地址就是：

```
┌───────────────┬──────────────┐
│page number + 1│00000000000000│
└───────────────┴──────────────┘
 p: m-n bits     d: n bits
```

而其含义就是下一张页表的 0 号位。而我们知道，相邻页对应的帧不一定是连续的，但这个不连续的性质对虚拟地址是透明的。

#### 总体梳理

稍微对上面的内容做一下总结，我们拥有了**逻辑的页**到**物理的帧**的映射关系，这个映射关系存在**页表**里，实现逻辑上连续、物理上离散的内存**块**索引；而利用 page number + offset 的结构定位了内存块中的具体地址，其中 offset 在帧和页中都表示对于块首地址的偏移，因此可以直接迁移使用。

因此，从虚拟地址到物理地址的映射，实际上就是在页表中查询虚拟地址中的 page number，将其换为 frame number，再直接拼接 offset 就行了。

<center> ![](img/32.png){ width=80% } </center>

> 实际上这是个非常自然的过程：整体地看虚拟地址，就是直接在连续的虚拟内存中找到对应的 Byte；整体地看物理地址，同样也是直接在连续的物理内存中找到对应的 Byte。现在通过置换二进制地址字符串的前缀，实现了一个寻址空间的映射。而这个映射中，表示 offset 的后缀不变，正对应着页和帧中偏移寻址规则的统一。

!!! warning "Protection"

    请注意，使用过程中有些页可能尚未与实际的帧建立映射关系，换句话来说是不可用的。所以我们需要一个手段来标识表项是否有效，于是在页表中引入 valid bit，用来标识页是否有效，如果试图访问 invalid 的地址，则会出现异常，以此实现了 protection。

    <center> ![](img/34.png){ width=80% } </center>

    也有一些操作系统通过维护 page-table length register, PTLR 来实现 protection，这里不重点介绍。

!!! tip "page size 的选择"

    容易理解，page size 较大时，页表项更少，而页更容易被浪费，但对于磁盘来说，单次大量的传输效率更高；page size 较小时，页表项更多，需要更多内存和时间来处理页表，所以具体 page size 的大小要具体问题具体分析、与时俱进。

在之后[页表设计改进](#页表设计改进){target="_blank"}一节中，我们将继续对页表的结构进行修改，但整体使用逻辑不变。

### 硬件支持

!!! info "导读"

    本节侧重于从硬件实现的角度来看分页技术。

我们前面说过，页表是 per-process data structures，所以页表应当作为一个进程的元信息被维护。显然我们不能直接用大量寄存器来维护页表（理论上很快，但是太贵、设计上也不现实），所以页表实际上应当被放在<u>内存</u>中（进一步的，为了保证效率，我们将页表放在主存中），我们通过用寄存器维护一个指向页表的指针来维护页表，这个特殊的寄存器被称为**页表基址寄存器(page-table base register, PTBR)**，当进程不处于 running 态时，PTBR 应当被存储在 [PCB](Unit2-Part1.md#PCB){target="_blank"} 中，在 [context switch](./Unit1.md#context-switch){target="_blank"} 的过程中，我们也应当对 PTBR 进行交换。

!!! question "嘿！可是内存真的好慢！"
    不仅如此，由于地址映射的实现逻辑，我们首先需要利用页表查询帧号，利用帧号去得到物理地址，再去内存里做查询，这里有足足两次内存访问操作！

    不仅如此*2，你可能还需要遍历整个页表才能达到你的目的！

#### TLB

为了解决这个问题，我们引用计组里学到的 Eight Great Ideas 之 Make Common Case Fast！引入一个缓存来加速页表的维护：**页表缓存(translation look-aside buffer, TLB)**，它实际上是 MMU 的一部分[^1]，页号和帧号以键值对的形式存储在 TLB 中。除了访问速度快以外，TLB 允许并行地查询所有键值对，这意味着你不再需要一个一个遍历页表中的内容了！从效率上来说，现代的 TLB 已经能够在一个流水线节拍中完成查询操作。

但是这么厉害的东西肯定还是有局限性的，TLB 一般都比较小，往往只能支持 32 - 1024 个表项。而且，作为一个“缓存”，它有可能产生 miss（即没在 TLB 中找到待查的页号），当 TLB miss 出现的时候，就需要访问放在内存中的页表，并做朴素的查询。同时，按照一定策略（如 LRU、round-robin to random 等[^2]）将当前查询的键值对更新到 TLB 中。

<center> ![](img/33.png){ width=80% } </center>

此外，TLB 允许特定的表项被线固(wired down)，<u>被线固的表项不再允许被替换</u>。（~~这个中文是我自己才华横溢出来的，请不要到处用容易被当没见识。~~）

!!! warning "虽然页表是 per-process data structures，但 TLB 并不是！"
    
正因如此，在 context switch 的时候，我们需要清空 TLB，即进行 flush 操作，否则下一个进程就会访问到上一个进程的页表。又或者我们不需要每次都清空 TLB，而是在 TLB 的表项中加入一个**地址空间标识符(address-space identifier, ASIDs)**字段；在查询页号时，也比较 ASID，只有 ASID 一致才算匹配成功。

<center> ![](img/35.png){ width=80% } </center>

!!! section "定量分析"

    我们使用**击中比例(hit ratio)**来描述我们在 TLB 中成功找到我们需要的页帧键值对的概率，那么假设访问一次内存需要 $t \text{nanoseconds}$，那么使用该 TLB 的**有效内存访问时间(effective memory-access time)**为：

    $$
    \begin{aligned}
        &\text{effective memory-access time} \\
        &= \underbrace{\text{hit ratio} \times \text{memory-access} }_\text{TLB hit}
         +  \underbrace{(1 - \text{hit ratio}) \times 2 \times \text{memory-access}}_\text{TLB miss} \\
        &= p_{\text{hit}} \times t + (1 - p_{\text{hit}}) \times 2t \\
        &= (2 - p_{\text{hit}})t
    \end{aligned} 
    $$

    > 在现代计算机中，TLB 的结构可能会更加复杂（可能有更多层），所以实际的计算可能比上述更加复杂。

### 共享页

虚拟地址与物理地址的映射并非需要是单射，换句话来说，多个页可以对应同一个帧，这就是**共享页(shared page)**。

共享页可以用来提高代码重用率，例如，多个进程可能会使用同一个库，那么这个库就可以被共享，而不需要每个进程都各自在物理内存中准备一份。[共享库](#动态链接和共享库){target="_blank"}就通常是使用共享页来实现的[^7]^,^[^8]。

再比如，我们在[进程管理#进程间通信](./Unit1.md#进程间通信){target="_blank"}中提到过通过共享内存来实现进程间通信，在某些操作系统中，共享内存就是通过共享页来实现的。

### 页表设计改进

我们回顾一下目前的[页表的设计](#帧--页){target="_blank"}：现在的页表是以页号为索引、帧号为值的一维数组，而由于我们直接将页表存在物理内存中（否则会黄油猫[^3]！），所以我们其实需要一块完整的连续物理内存来存储整个页表——每一个虚拟地址我们都得存。

假设我们的虚拟地址一共 32 位，而 page size 为 4 KB = 2^12^ B，即 offset 对应虚拟地址 32 位中的后 12 位，那么我们就需要连续的 2^20^ 个表项（对应一共 2^20^ 个虚拟地址）来存储页帧的映射关系。假设一个表项 4 Bytes，那么光一个页表就要占据我们 4 MB 的物理内存——而且是连续的物理内存。这实在是太夸张了！

现在我们要冷静地解决这个问题！现在问题有两个：⓵ 页表实在太大了，⓶ 它不仅大，而且必须是连续的。其中第二点是最关键的，在本节之后的内容中，我将称之为“连续内存约束问题”（非正式表述）。

我们介绍三个方法来解决上述问题：[分层页表](#hierarchical-paging){target="_blank"}、[哈希页表](#hashed-pgtb){target="_blank"}和[反转页表](#inverted-pgtb){target="_blank"}。

<a id="hierarchical-paging"/>
???+ section "分层页表"

    同样，我们首先来思考为什么这里需要的内存是连续的——作为一个一维数字，只有内存连续才能保证 random access。类似的问题我们在探索[连续分配](#连续分配及其问题){target="_blank"}的过程中已经遇到过了：在物理地址空间中寻求连续，一个重要就是因为只有物理地址的设计中，只有保证连续才能保证能 random access 地去访问地址，而现在这个一维数组太大块了，我们希望它碎一点；而我们通过保证分块地连续（帧内物理地址的连续），再保证块索引的连续（虚拟地址空间中页号的连续）的方式解决了这个问题，就好像把一个一维数组变成了一个**指针数组**，或者说逻辑上的二维数组。

    现在我们遇到的问题实际上就是这个“指针数组”也太大块了，希望它能碎一点，所以解决方法已经呼之欲出了——将这个指针数组再进行拆分，变成一个维护指针数组指针的数组，或者说逻辑上的三维数组：

    ```
    page number     page offset
    ┌───────┬───────┬──────────────┐
    │ p1    │ p2    │ d            │
    └───────┴───────┴──────────────┘
    ```

    类似的，我们可以将它看作在原先维护 p2 -> d 的 inner 页表外，再维护一个 p1 -> inner 的 outer 页表。通过这种方式，我们减少了单个页表所需要包含的表项数（原先一个页表需要有 2^p^ 个表项，现在只需要有 2^p1^ 或 2 ^p2^ 个即可）；除此之外，虽然看起来表总量增加了（现在一共需要 2^p1+p2^ + 2^p1^ 个表，原来只需要 2^p1+p2^ 个表），但是 ⓵ 一方面这个增加是可以忽略的相对小量，⓶ 另外一方面，实际上我们并不总是需要创建所有的表——假设某个 inner 表里的虚拟内地址我们都用不到，那么我们就不需要创建这个 inner 表，只需要在 outer 表中标记这个 inner 表是 invalid 就可以了。

    通过这种设计，我们成功地节省了维护页表所需要的内存空间，同时减小了连续内存对页表维护的约束。

    如上这种设计，就是**分层分页(hierarchical paging)**，而上面这个就是二级页表(two-level page table)设计。

    显然，有二就可以有三，有三就可以有四，具体使用哪种，应当秉持具体问题具体分析的原则。

    !!! extra "Risc-V"

        有兴趣的读者可以参考 xg 的这篇[《RISC-V 页表相关》](https://note.tonycrane.cc/cs/pl/riscv/paging/){target="_blank"}笔记，来了解 Risc-V 中的分页设计，写得很清楚，推荐阅读。

        同时，实验三指导手册也提供了关于 [Risc-V Sv39](https://zju-sec.github.io/os23fall-stu/lab3/#risc-v-sv39-page-table-entry){target="_blank"} 的一些介绍。

<a id="hashed-pgtb"/>
???+ section "哈希页表"

    简单回顾一下我们遇到的问题：页表太大，而且必须是连续的。但是实际上我们使用的映射关系，从虚拟地址来看是集中的，从物理地址来看是稀疏的，反正页表中有大量表项是 invalid 的，所以想办法不存这些用不到的表项，也是一种解决思路。

    ???+ quote "Links"

        - [Hash function | wikipedia](https://en.wikipedia.org/wiki/Hash_function){target="_blank"}
        - [哈希 | 鹤翔万里的笔记本](https://note.tonycrane.cc/cs/algorithm/ds/summary2/){target="_blank"}
        - [Hashing | sakuratsuyu's Notes](https://sakuratsuyu.github.io/Note/Computer_Science_Courses/FDS/Hashing/){target="_blank"}

    **哈希页表(hashed page table)**维护了一张哈希表，以页号的哈希为索引，维护了一个链表，每一个链表项包含页号、帧号、和链表 next 指针，以此来实现页号到帧号的映射。此时，一方面我们没必要再维护一个大若虚拟地址总数的表，另一方面由于引入链表，大量的指针操作导致对地址连续性的要求降低，也能变相地减轻连续内存约束。

    <center> ![](img/37.png){ width=80% } </center>

    ???+ section "clustered page tables"

        A variation of this scheme that is useful for 64-bit address spaces has been proposed. This variation uses **clustered page tables**, which are similar to hashed page tables except that **each entry in the hash table refers to several pages** (such as 16) rather than a single page. 
        
        Therefore, a single page-table entry can store the mappings for multiple physical-page frames. Clustered page tables are particularly useful for **sparse** address spaces, where memory references are **noncontiguous and scattered** throughout the address space.

<a id="inverted-pgtb"/>
???+ section "反式页表"

    我们之前的页表通过维护虚拟地址的有序来实现对页号的 random access，但是代价是需要维护大量连续虚拟地址。反式页表(inverted page table)则直接大逆不道地修改了整套思路——以物理地址为索引维护映射关系。

    同时，在这种设计下，整个操作系统只维护一张反转页表。由于不需要每个进程都存储一张页表，整体只存储物理地址数量个表项，所以相对来说节省了内存空间。

    但是显然，这样做我们就没法自然地支持[共享页](#共享页){target="_blank"}了[^4]，因为索引应当是 unique 的。不仅如此，由于我们只做虚拟地址 -> 物理地址的查询，所以在这种结构下我们只能遍历整个表来找映射关系。诸如此类还有不少限制。

    > 总而言之，我觉得这个方法很臭。

    !!! extra "其它"

        可能还会涉及一些段式设计以及相关设计，但是并不主流，但考试可能会考，大家可以选择性去了解一下。

## 交换技术

我们知道，只有在内存中的指令(instructions)才能被 CPU 执行，因而内存大小一定程度上限制了多道程度(degree of multiprogramming)。但是，大部分内容并不需要全程待在内存中[^6]，即不会频繁地被使用。

所以，我们可以考虑在不需要的时候将部分内容放在后备存储(backing store)中，而在需要的时候再将它们弄到内存里——这就是**交换(swap)技术**。在应用交换技术后，那些实际放在后备存储里的 instructions，可以“假装也在内存中”，即 high level 的看，我们并不知道它到底是放在内存还是后备存储里，但是保证当 CPU 需要访问这一块内容时，这些内容会被载入内存。

> 在这里我们只简单介绍一下交换的思想，而具体的细节与实现，将会在之后连同更明确的定义给出。

<figure markdown>
<center> ![](img/38.png){ width=60% } </center>
Standard swapping of two processes using a disk as a backing store.
</figure>

在标准的 swap 操作中，我们以进程为单位进行 swap，这意味着我们要把所有 per-process 的东西都一同 swap，相当于“冻结”整个 process 或“解冻”了整个 process，就好像跨内存和后备存储进行 context switch。可想而知，这个开销是巨大的。

如今我们有分页技术，我们完全可以以页/帧为单位进行 swap，只不过我们称这种以页/帧为单位的**交换(swap)**叫**换页(page)**。

> 显然这里的 “page” 是基于 [Definition 2](#page-frame-def-2){target="_blank"}。

<figure markdown>
<center> ![](img/39.png){ width=60% } </center>
Swapping with paging.
</figure>

!!! property "优势"

    利用页置换技术和虚拟内存的组合拳，我们可以让进程所使用的内存空间总和看起来大于硬件支持的物理内存空间大小上限，**扩展**抽象的“内存”的容量。

    <figure markdown>
    <center> ![](img/40.png){ width=80% } </center>
    Diagram showing virtual memory that is larger than physical memory.
    </figure>

宏观地来看，我们可以抓住主要矛盾，只将每个进程中最迫切需要的那些页留在物理内存中，对进程进行页级的内存管理，于是平均每个进程需要在物理内存中的数据量更小、物理内存中可以存放的“进程”数量更多、多道程度(degree of multiprogramming)得以提高。

### swap 空间

进行 swap 需要从后备存储中来获取进程内容。在后备存储中，有一块专门用来做这件事的地方，叫**交换空间(swap space)**，通常和 swap space 进行交换会更快。但是，代码并不是一开始就在交换空间的，我们需要找一个时机把代码放进去以后，才能纵享丝滑。

> 一种 naive 的做法是，在进程创建的时候就把整个代码镜像放进交换空间，这个做法的缺点就是它的定义，我们会需要在一开始做一个大规模的复制，这个做法有诸多显然的弊端。
>
> 另一种做法是，当一个 page 第一次被使用的时候，它从文件系统中被 page in；而在被 replace 而 page out 的时候，将它写入 swap space。这样，下次需要这个 page 的时候就可以从 swap space 里 page in。
>
> 还有一种策略是，当操作系统需要某个页面时，它会直接从文件系统中将这些页面加载到内存中。这些页面在内存中的副本是不会被修改的，因此当这些内存需要被替换的时候，可以直接被覆盖。（在这种情况下，文件系统本身就像一个后备存储）但是，对于那些不与文件相关联的页面，我们称之为匿名内存(anonymous memory)，例如进程的栈和堆，仍然需要使用交换空间。
>
> **这一部分的内容写的比较简略，对应课本 10.2.3 的后半部分。**

> 但是无论如何，硬盘的速度还是不如内存，所以在内存足够的情况下我们一般不使用 swap。

接下来，我们引入一套更完善的虚拟内存管理系统：demand paging。

## 按需换页系统

**按需换页(demand paging)**^[Wiki](https://en.wikipedia.org/wiki/Demand_paging){target="_blank"}^和[交换技术中的页置换](#交换技术){target="_blank"}很类似，指只把**被需要**的页载入内存，是一种内存管理系统。

!!! extra "pure demand paging"

    如果激进一点，如果在被需求之前页不会被载入内存，只有在内存被需求后才会被载入内存，我们称之为纯按需换页(pure demand paging)。

    > This scheme is pure demand paging: **never** bring a page into memory until it is required.

可以想象，现在给定任意一个虚拟地址，有三种可能：

1. 该“地址”在物理内存中，页表中**存在**虚拟内存到物理内存的映射关系，是 valid 的；
2. 该“地址”在后备存储中，页表中**不存在**虚拟内存到物理内存的映射关系，是 invalid 的；
3. 该“地址”并没有被分配，页表中**不存在**虚拟内存到物理内存的映射关系，是 invalid 的，或权限不符（如试图写入一个只读页）；

与引入交换技术之前的页表设计相比，多出来的就是情况 2.。如果系统访问了一个在页表中是 invalid 的页，就会生成异常，我们称这种情况为**缺页(page fault)**。

??? extra "major & minor page fault"

    实际上，情况 2. 还可以细分为两种，一种是**major/hard page fault**，一种是**minor/soft page fault**。

    Major page fault 指的是缺了的页不在内存中的情况；而 minor page fault 指的是缺了的页在内存中存在，只不过没在当前页表中建立映射。

    这里稍微细说一下 minor page fault，出现 minor page fault 有两种可能：

    1. 进程可能需要引用一个共享库的 page，而这个共享库的 page 已经在内存中，我们只需要更新一下页表把它链上去就行了；
    2. 进程可能需要引用一个之前被释放了的 page，而那个被释放的 page 还没有被 flush 或分配给别的进程，此时我们可以直接使用这个 page（~~捡垃圾！五秒原则？~~）；

区别于之前的页表设计——访问 invalid 的表项是一种预期外行为，现在产生缺页反而**更多**是一种预期内的行为——系统对某个被 page out 了的页产生了“需求”。当然，情况 3. 这种非法操作也会引起异常，所以我们需要在之后的异常处理过程中对此做区分并分别处理。

> 说实话其实我感觉这里的逻辑稍微有点绕，可能是一些历史原因。

操作系统就需要去处理这个异常的大概流程如下：

<a id="page-fault-handling"/>
!!! section "page fault 处理流程"

    1. 检查一张 PCB 里的内部表，来区分这个地址到底是情况 2. 还是情况 3.；
        1. 如果是情况 2.，则继续如下操作以将其 page in；
        2. 如果是情况 3.，则终止进程；
    2. 从[可用帧列表](#可用帧列表){target="_blank"}里拿出 frame 用来写入；
        1. 如果可用帧列表为空，则进行[页置换](#置换策略){target="_blank"}；
    3. 开始从后备存储读取内容，并写入 frame；
    4. 完成读写后，更新内部表和页表等元信息；
    5. 重新执行引起 page fault 的 instruction；
        1. 该操作十分关键，类似于死锁里的回滚操作，支持这项操作也具有一定的难度，包括如何确切地恢复回指令执行之前的状态、如何消除执行了一半的指令的效果等；

<figure markdown>
<center> ![](img/41.png){ width=80% } </center>
Steps in handling a page fault.
</figure>

!!! warning "慢！"

    想象一下，每当发生一次 page fault，并且我们假设这些 page faults 都属于情况 2.，那么它可能会经历这些过程：

    1. 产生异常后，进行一次 **context switch** 后进入异常处理程序；
        1. 处理异常，包括决定异常类型、在内部表里寻找地址对应于后备存储中的何处；
        2. 发起后备存储 -> 内存的 I/O 请求；
    - （等待过程中 CPU 被调度）；
    1. I/O 中断产生，此时也会有一个 **context switch**；
        - 处理中断，包括决定中断类型、更新页表和其他内部表；
    2. 等待 CPU 再次调度到该进程，显然这里也有个 **context switch**；
        - 做一些回滚操作，然后重新执行引起 page fault 的指令；

    可以发现，处理 page fault 超慢的！因此，我们应当尽可能减少 page fault rate。

程序执行的局部性保证了 page fault 不会太频繁，导致带来不可接受的额外开销。需要注意，单条指令是有可能带来若干次 page fault 的（例如可能在 instruction fetch 的时候产生、可能在 operand fetch 的时候产生等）。

### 回顾 copy on write

我们在[内存管理](./Unit1.md){target="_blank"}一节中提出了 [copy on write](./Unit1.md#copy-on-write){target="_blank"} 技术和 [vfork](./Unit1.md#vfork){target="_blank"} 技术，现在读者可以尝试再回顾一下这两个知识点与本节内容的联系。

### 可用帧列表

在 demand paging 系统里，页是动态地被映射到帧的，所以我们需要维护一个**可用帧列表(free-frame list)**，用来记录当前哪些帧是空闲的。

<figure markdown>
<center> ![](img/42.svg) </center>
Example of free-frame list.
</figure>

在系统启动后，我们需要将所有可用的帧都加入到 free-frame list 中；当有用户需要物理内存时候，就从 free-frame list 中取出一项，对其进行擦除，即被需求时清零(zero-fill-on-deman)。

!!! question "考虑为什么要执行 zero-fill-on-deman"

如果读者足够敏锐就会发现，我们只说了怎么取出 free-frame，而没说 free-frame 如何“再生”。

当我们发现 free-frame list 为空，即没有空闲的 frame 时，我们考虑将一些先前已经被分配的 frame 给 page out 走，拿来给当前这个页用。而具体如何选择换走哪个 frame，我们会在[置换策略](#置换策略){target="_blank"}一节中介绍。

<a id="free-frame-buffer-pool"/>
!!! section "free-frame buffer pool"

    虽然我们还没介绍[置换策略](#置换策略){target="_blank"}，但是想象一下，如果等到没有 free-frame 的时候再去做置换，那么进程就需要**等待置换完成**以后再分配。
    
    我们可以考虑在这里引入一个抽象的 buffer，我们可以保证 free-frame list 始终有一定数量的空闲帧，例如 3 个。这样当进程来索取 free-frame 的时候，free-frame list 大概率总是能够直接给出一个 free-frame 的，而给出 free-frame 后，如果发现 free-frame list 中的剩余帧数小于 3，那么就可以独立地开始进行置换，而不必阻塞进程。

    ---

    又或者，我们不使用一个确切的界，而是通过一种动态调节，维护一个上界和下界：当 free-frame 数小于下界时，一类叫**收割者(reapers)**的内核例程就开始使用 [replacement algorithm](#置换策略){target="_blank"} 来 reclaim 已经被分配的 frame，直到 free-frame 的数量触碰到上界。

    <figure markdown>
    <center> ![](img/43.png){ width=60% } </center>
    Reclaiming pages.
    </figure>

    ---

    进一步的，万一此时出现了一些特殊情况，导致实际的 free-frame 数非常少，达到了一个非常低的界，此时就出现了 **OOM(out-of-memory)**。此时，一个叫做 OOM killer 的例程就会杀死一个进程，以腾出内存空间。
    
    在 Linux 中，每个进程会有一个 OOM score，OOM score 越高约容易被 OOM killer 盯上，而 OOM score 与进程使用的内存的百分比正相关，所以大概的感觉就是谁内存用的最多就杀谁。[^9]如果读者对 Linux 的 OOM 机制有兴趣，可以看看角注 9。

!!! info "导读"

    > 准确来说接下来[分配策略](#分配策略){target="_blank"}和[置换策略](#置换策略){target="_blank"}都应当是 demand paging 的子条目，但是四级标题实在太小了，所以我设为了三级标题。

    我们已经阐述了一个正在运行中的 [demand paging](#按需换页系统概述){target="_blank"} 系统是如何运作的，现在需要补足一些细节。

    1. [分配策略](#分配策略){target="_blank"}：初始化时，如何分配进程所需要的 frame？
    2. [置换策略](#置换策略){target="_blank"}：当 free-frame 不足时，如何进行 replacement？

### 分配策略

在 pure demand paging 里，每个进程通过 page fault 不断“蚕食” free-frame，但如果我们不适用 pure 的 demand paging，那么就需要决定一开始分配多少 frames 给一个 process。

首先，对于单个进程的分配，存在一个较严格的上下界：

!!! section "lower bound & upper bound"

    1. 分配给一个 process 的 frames 数量不能大于 free-frame 总量；
        - 即 the maximum number of frames per process is defined by the amount of available physical memory；
    2. 分配给一个 process 的 frames 数量不能小于 process「执行每一条指令所需要涉及的 frames」的最大值；
        - 这句话有点绕，稍微解释一下：
            - 一些指令可能会需要引用其它 frame（例如的 load，move，以及会产生 indirect references 的指令等），而且 instruction fetch 以外的额外 memory reference 可能不止一个；
            - 我们应当保证涉及的若干 page 都能被存在内存中；
            - 因此，从某种角度来说：the minimum number of frames per process is defined by architecture；

**分配算法(frame-allocation algorithm)**按照分配的帧的大小来分，主要有这么两种：

???+ section "equal allocation"

    顾名思义，每一个进程被分配的 frame 总量都相同，假设共 $n$ 个 process，$m$ 块可用 frame，那么每个进程被分配的 $\left\lceil\frac{m}{n}\right\rceil$。

???+ section "proportional allocation"

    比例指按进程的大小来分配，假设共 $n$ 个 process，$m$ 块可用 frame，其中每个 process 的大小为 $s_i$，那么每个进程被分配的 frame 大小为 $a_i = \left\lceil \frac{s_i}{\sum_{j}^n s_j} \times m\right\rceil$。

???+ section "proportional allocation with priority"

    注意到， 目前提到的两种做法都和进程的优先级无关，但从需求上来讲，我们可能倾向于让高优先级的进程被分配更多的 frame 以降低 page fault rate 来增加它们的效率。
    
    所以，我们可以在 proportional allocation 的基础上，在计算 $a_i$ 时综合考虑 priority。

我们发现，上面关于内存分配大小的式子中，有一项 $n$ 表示 #process，区别于其它相对静态的参数，这个参数是会在调度过程中动态变化的，所以实际上分配给每个进程的 frame 数量也是会动态变化的。

在多核设计下，有一种设计叫做 NUMA^[Wiki](https://en.wikipedia.org/wiki/Non-uniform_memory_access){target="_blank"}^，我们在 Overview 其实也提到过。在这种设计里，由于硬件设计问题，不同的 CPU 都有自己“更快”访问的内存。读者可以通过上面的 Wiki 链接做详细了解。

### 置换策略

当 free-frame list 为空，但用户仍然需要 frame 来进行 page in 时，就需要进行**页置换(page replacement)**，将并没有正在被使用的页腾出来给需要 page in 的内容用，而这个“被要求腾出地方”的页，我们称之为**牺牲帧(victim frame)**。

> 显然这里的 “page” 是基于 [Definition 2](#page-frame-def-2){target="_blank"}。

我们细化 [page fault 处理流程](#page-fault-handling){target="_blank"}的 2.a. 项，大概是如下的步骤：

!!! section "page replacement"

    1. 利用**置换算法(replacement algorithm)**决定哪个 frame 是 victim frame；
    2. 如果有必要（dirty），victim frame -> 后备存储；
    3. 更新相关元信息；
    4. 返回这个 victim frame 作为 free-frame；

如果这个 victim frame 在被 page in 以后没有被修改过，那么我们可以直接将它覆盖，不需要写回后备存储，能节省一次内存操作；反之，如果这个 victim frame 被修改过，那么我们需要将它写**回**后备存储，类似于将修改给 “commit” 了。而为了实现这个优化，我们用一个**修改位(dirty bit 或 modified bit)**来记录页是否被修改过，当 frame 刚被载入内存时，dirty bit 应当为 0；而一旦帧内有任何写入操作发生，dirty bit 就会被置 1。

现在我们来讨论具体的**置换算法(replacement algorithm)**。

#### OPT

理论上最优，即 ⓵ 能带来最低的 page fault rate，⓶ 绝对不会遭受 [Belady's anomaly](#Belady-s-anomaly){target="_blank"} 的做法是：<u>在**未来**最久的时间内不会被访问到的页作为 victim frame</u>。这句话说起来有点绕，用英文描述是：Replace the page that **will** not be used for the longest period of time.

换句话来说就是选之后再也不会被用到的或（如果没有前者）下一次用到的时间最晚的页作为 victim frame。

可以发现，我们实际上很难来预测一个 frame 下一次被使用是什么时候，所以该方法只是一个理论上的最优建模，我们在后面应当考虑去逼近这个建模。

???+ tip "头脑风暴"

    这段内容有没有让你想起我们已经学过的某个东西？

    ??? success "提示"

        [Shortest-Job-First Scheduling](./Unit1.md#SJF){target="_blank"}!

        实际上，下面介绍 FIFO 你也应当会想起 [FCFS](./Unit1.md#FCFS){target="_blank"} 调度算法。

#### FIFO

先入先出(first-in, first-out, FIFO)策略，即<u>选择正在使用中的、最早进入内存的 frame 作为 victim frame</u>。我们可以通过在内存中完整地维护一个 FIFO 队列来实现这个策略。

FIFO 策略的优点就是简单，方便实现；缺点是并不够好——早被载入的 page 也可能会被频繁的使用。换句话来说，这个方法用被载入的早晚来建模 page 的使用频率，但是这个建模相比 [optimal](#OPT){target="_blank"} 的建模并不足够接近。

???+ tip "头脑风暴"

    请读者试着思考一下，假设现在最早被载入的 page 正在使用过程中，这时候出现了一个 page fault，会发生什么？会导致出现错误吗？

    ??? success "提示"
            
        会影响效率，但是不会出错哦！

<a id="Belady-s-anomaly">
??? extra "Belady's anomaly"
    
    > 这一段内容没啥用，只是一个有趣的现象。
        
    有一种情况叫做 Belady's anomaly，在 FIFO 策略下（其它 replacement algorithm 可能也会发生），随着 frame 数量的增加，page fault rate **可能**会增加。

    例如如下 page 访问序列：

    $$
    1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5
    $$

    在 3 个 frame 的情况下，page fault rate 为 $\frac{9}{12} = 75.0%$；而在 4 个 frame 的情况下，page fault rate 为$\frac{10}{12} = 83.3%$。

#### LRU

我们的目标是为了逼近难以实现的 [optimal](#OPT){target="_blank"}，而 [optimal](#OPT){target="_blank"} 之所以难以实现，是因为我们很难“预知未来”，我们能利用的只有已经经历过的事情。

Least recently used(LRU) 算法的思路是，基于「很久没被用过的 page 可能在短期不太会被再次使用，刚刚用过的 page 可能在短期被频繁地用」的假设，用“最久没用过”来建模「未来最久时间内不会被访问」，即<u>选择最久没被访问过的 frame 作为 victim</u>。

LRU 是比较常用的 replacement algorithm（实际上是 [LRU-Approximation](#LRU-Approximation){target="_blank"}），因为是被认为比较好的 replacement algorithm。

---

现在我们来考虑如何实现 LRU，或者说，如何来维护一个 frame 有多久没被访问过。

!!! section "stack algorithms"

    1. 计数器：使用一个计数器来标记一个帧有多久没被使用过；
        1. 当一个 frame 被使用的时候将计数器归零；
        2. 需要考虑每个 frame 的计数器都需要被定期更新；
        3. 需要考虑计数器可能溢出；
        4. 在找 least recently used frame 的时候需要去搜索 counter 最大的 frame（你也可以考虑用一个数据结构去维护它，但是会增加设计复杂度）；
    2. 链表序列：使用一个双向链表来维护一个有序序列，frame 在序列中的位置暗示了它们最近被使用的时间；
        - > 貌似主流都用“栈”来建模，但我觉得这不是栈，况且说是栈，其实现还是用双向链表；
        1. 每当一个 frame 被使用的时候：
            1. 如果它在链表中，就将它移动到链表头部；
            2. 如果它不在链表中，就将它加入到链表头部；
        2. 这种设计下每次的 “least used frame” 总是位于序列末尾，因此不需要做额外的搜索；

    上面两种做法都被称为**栈算法(Stack Algorithms)**。

    > 虽然我坚持认为这里和栈没关系。

    !!! warning "缺陷"

        虽然 LRU 对于拟合 [optimal](#OPT){target="_blank"} 是比较好的，但是它也有一些缺陷：
         
        1. 对于计数器做法，维护每个 frame 的 clock 想想就很慢，除非有特定的硬件来优化这个操作（例如不需要由操作系统来操心 clock 的维护）；
        2. 对于两者，由于每次内存被访问的时候都需要进行维护，如果通过 interrupt 来调用 stack algorithms，那么开销将会巨大；

 此外，LRU 算法不会出现 [Belady's anomaly](#Belady-s-anomaly){target="_blank"}。

 ---

<a id="LRU-Approximation"/>

由于我们在 Stack Algorithm 里提到的诸多弊端，我们考虑**近似**地，实现 LRU 算法——实际上是近似实现 Stack Algorithm。

多数操作系统会提供一个叫 reference bit 的功能。所有 frame 都有一个与之关联的 reference bit，在初始化的时候都会被置 0；而每当 frame 被使用时，reference bit 就会被置 1。于是，我们可以通过观察 reference bit 来观察某些 frame 是否被使用过。

开始之前，我们分析 LRU 的限制，主要体现在两个方面：⓵ 需要全部历史信息，维护成本较大（需要设计数据结构来存储）；⓶ 数据维护过于频繁，每次使用 frame 都需要用一段不小的开销去更新状态。

对应的解决方案是：⓵ 我们完全可以只关注一个邻域里的历史信息，⓶ 我们可以降低更新 frame 信息的频率。（虽然对于后者，实际上如果对应于 reference bit 的更新，其实并没有降低频率。）

???+ section "Additional-Reference-Bits Algorithm"

    只有 reference bit 的话没法反应出 frame 使用的“远近”，也就是使用的顺序。

    既然缺的是顺序，我们就考虑建模 frame 的使用远近。其中最首要的一个任务就是获取历史信息，所以我们需要用一些 bits 来存储每个 frame 的历史使用信息；然后定期（利用时钟中断）地去检查、存储当前时间当前 frame 的 reference bit，相当于检测上一个采样间隔中该帧有没有被用过。

    具体来说，我们可以给每个 frame 一个 $k$ bits 的 bits vector $h = (h_{k-1}h_{k-2} \dots h_{1}h_{0})_2, \quad h_i \in \{0, 1\}$ 用来存储历史的 reference bits；然后每过 $\delta t$ ms，就产生一次时钟中断，检查 frame $f_i$ 的 reference bit $r_{t}$（为了简洁我们省略这个 $i$），此时我们更新 $h' = (r_{i,t}h_{k-1}h_{k-2} \dots h_{2}h_{1})_2$，即将 $h$ 右移一位，在高位补 $r_{i,t}$。

    实际上 $h$ 是一个类似队列的存在，而每次检查会把 reference bit 给 push 进这个队列里。因此，$h$ 换一个写法就是：$h' = (\underbrace{r_{t}r_{t-1}r_{t-2}r_{t-3} \dots }_{k\text{ bits}})_2$，也就是最近的 $k$ 次检测的历史记录。

    而之所以从高位开始，是因为在数值大小体系中，高位代表着高权重，正好对应 reference 出现的越近，frame 越新。所以，我们可以直接找这个 bits vector 中最小的那个 frame 作为 victim frame。

???+ section "Second-Chance Algorithm"

    Second-Chance Algorithm 只利用 reference bit 来进行置换，类似 [FIFO](#FIFO){target="_blank"} 的改进版，在 FIFO 的基础上，我们引入了 reference bit，并且定期擦除 reference bit。

    我们**循环地**遍历 frames，并逐一检测 reference bit：

    1. 如果 reference bit 为 0，说明采样间隔中这个 frame 没被用过，那么这个 frame 就是我们要找的 victim frame；
    2. 如果 reference bit 为 1，说明采样间隔中这个 frame 被用过，于是给这个 frame 一次“豁免”的机会，将它的 reference bit 设置为 0，并继续寻找下一个 frame；

    !!! tip "头脑风暴"

        在 2. 中为什么要将它置 0？这个置 0 的含义和在时钟中断采样的时候的置 0 一样吗？

        ??? success "提示"

            这就是 Second-Chance Algorithm 中的 “second” 的来源。将它置 0 后下一循环再碰到它的时候，就不会再被“豁免”了。

    显而易见的，在某些情况下，该算法可能会退化为 FIFO，甚至更差，找到 victim page 之前，该算法最多可能会遍历两遍 frames。而该算法也可以看作 Additional-Reference-Bits Algorithm 的简化版，如果说 Additional-Reference-Bits Algorithm 是通过比较若干轮采样的历史采样记录来对 frames 做排序，以决定哪一个是 “LRU”；那么 Second-Chance Algorithm 就是在一个采样周期里，将 frames 做二值分类，在“远近”这件事的建模上，更加激进和粗粒度。

???+ section "Enhanced Second-Chance Algorithm"

    > 书中有一句话令人费解，我发现已经有人在 StackOverflow 上问了这个问题，可以参考一下这个[问题](https://stackoverflow.com/questions/58496569/how-does-the-enhanced-second-chance-algorithm-has-a-preference-to-the-changes-th){target="_blank"}。

    既然 Second-Chance Algorithm 的建模有点太激进和粗粒度，那我们考虑把粒度再弄细一点。

    之前我们都只看 reference bit，现在我们把 dirty bit 也纳入考虑。考虑二元组 $(reference, dirty)$，两个 bit 有四种组合：

    1. $(0, 0)$：没被用过，也没被修改过；
    2. $(0, 1)$：没被用过，但被修改过；
    3. $(1, 0)$：被用过，但没被修改过；
    4. $(1, 1)$：被用过，也被修改过；

    由于被修改过的 frame 在被置换的时候需要执行写回，所以我们希望尽量晚一点使用这类 frame。在这种分类下，前两种合并，后两种合并，就是先前的 Second-Chance Algorithm 了。

    在 Enhanced Second-Chance Algorithm 中，我们找到第一个 1. 情况的 frame 作为 victim；如果没有，就去找第一个 2. 情况的 frame 作为 victim……以此类推。[^10]

    因此，Enhanced Second-Chance Algorithm 可能最多会遍历 4 次 frames。在已经介绍的三个算法里，它是唯一一个考虑了 dirty bit 的。

    > ~~总感觉 LRU 已经被改的面目全非了x~~

#### 基于计数的置换

我们考虑用 counter 来记录正在使用的 frame 中每个 frame 被使用的次数，用 counter 的值来建模“使用频率”。按照后续操作不同分为这两种：

- Least frequently used(LRF) 选择 counter 最小的 frame 作为 victim frame；
- Most frequently used(MRF) 选择 counter 最大的 frame 作为 victim frame；
    - 这个做法基于「counter 小的 frame 可能才刚刚被 load 进来」这个假设；

显而易见的，这两个设计对 [optimal](#OPT){target="_blank"} 的拟合都不是很好，开销也都很大。


!!! question "能否置换其它 process 的帧？"
    
    在[分配](#分配策略){target="_blank"}的时候，我们为进程分配了一些帧以用于必要的运算活动。但我们知道，置换操作会动态地更新 frame 的使用情况。不知道你是否疑惑过：置换的时候，我们能否置换其它进程的 frame？以及我们要如何实现和维护这些策略呢？

    实际上，replacement 分为 local 和 global 两种：

    1. 使用 local replacement 时，<u>replacement 只发生在属于当前进程的帧中</u>，因而也就不会影响其它进程的内存使用情况；
    2. 而对应的，<u>global replacement 的 scope 是所有帧</u>，甚至可能一部分原来属于操作系统的帧，因而它能实现一些类似“抢占”的效果；
        - [Free-frame buffer pool](#free-frame-buffer-pool){target="_blank"} 就是一种天然的 global replacement 的实现方式；
    3. 如果我们稍微做一些设计，比如**只**允许高优先级的进程能够置换低优先级的 frame，则高优先级的进程使用的 frame 可能越来越多，进而不断优化高优先级进程的效率；

    当然，我们需要维护每个进程所拥有的 frame 数量被 minimum frame number 给 bound 限住。

    对比 local replacement 和 global replacement，主要就是一个封闭性和灵活性的 trade-off，很直观：global replacement 分配更灵活，内存的利用率更高，但对于 frame 被“抢”的进程来说，整体运行的效率就不稳定了；反观 local replacement，虽然由于能够利用的内存有限，可能出现别的进程省了不少但是自己很吃紧的情况，出现内存利用率较低的情况，但整体来说较为稳定。

    > 上面是书上的意思，但我认为这个评价还是不公平的，因为所谓的、属于进程的 frames 的数量是会变的，在新的进程被 allocation 后，进程总数会增加，而这个新进程只能从别的地方刮一些内存来用，所以就算用的是 local replacement，也说不上稳定。

### Trashing

### Memory Compression

### Allocating Kernel Memory

[^1]: [Translation lookaside buffer | Wikipedia](https://en.wikipedia.org/wiki/Translation_lookaside_buffer){target="_blank"}
[^2]: [Cache replacement policies | Wikipedia](https://en.wikipedia.org/wiki/Cache_replacement_policies){target="_blank"}
[^3]: 出自 [Buttered cat paradox | Wikipedia](https://en.wikipedia.org/wiki/Buttered_cat_paradox){target="_blank"}，我在这里表示死循环。
[^4]: [how does an inverted page table deal with multiple process accessing the same frame | Stack Overflow](https://stackoverflow.com/questions/44159535/how-does-an-inverted-page-table-deal-with-multiple-process-accessing-the-same-fr){target="_blank"}
[^5]: [What is the difference between executable and relocatable in elf format? | Stack Overflow](https://stackoverflow.com/questions/24655839/what-is-the-difference-between-executable-and-relocatable-in-elf-format){target="_blank"}
[^6]: 例如 ⓵ 异常处理程序，异常类型可能很多，对应的处理方案可能也会有很多，但均摊下来每一个异常处理程序的使用频率都不会很高；⓶ 数组列表等相对不那么智能的数据结构在定义声明的时候可能开了一大块内存，基本都是为了 bound 住可能需要的内存量的上界，但是可能实际上经常碰不到上界，例如我申请了一个 1024 长的 int 数组，但是我可能一般只会用到其中的前 128 个元素；⓷ 一些可能单纯不怎么常用的功能，道理和第一点是类似的。
[^7]: [Shared Memory "Segment" in Operating System | Stack Overflow](https://stackoverflow.com/questions/34545631/shared-memory-segment-in-operating-system){target="_blank"}
[^8]: [Where is linux shared memory actually located? | Stack Overflow](https://arc.net/l/quote/qjkjycww){target="_blank"}
[^9]: [Linux OOM (Out-of-memory) Killer | Medium](https://medium.com/@adilrk/linux-oom-out-of-memory-killer-74fbae6dc1b0#9707){target="_blank"}
[^10]: [Enhanced Second-Chance Algorithm](http://www.faadooengineers.com/online-study/post/ece/operating-systems/1165/enhanced-second-chance-algorithm){target="_blank"}