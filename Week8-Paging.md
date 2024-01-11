# What we're doing today
+ [Virtual Memory](#vmem)
+ [Paging](#paging)


## Virtual Memory <a name = "vmem"></a>
Yet again, we've ran into another OS abstraction that can intuitively be confusing. Loui has definitely talked about virtual memory at this point, but I want to go over it once more, as it is crucial to memory managament.

**Virtual memory** is just a fabricated address space given to every single process on the system...think of it as a seperate entity from the physical RAM on the system. Both the virtual address space and the physical address space are cut up into some granularity of size, which is usually 4kB large (4096 bytes). 

![virtual pages to physical frames](/images/page-frame.png)

We refer to these 4kB spaces as **pages** when they are in the VAS and **frames** when they are in the PAS. Ultimately, each page address gets **translated** into a physical memory address, that way the contents of virtual addresses exist in some shape or form on RAM. 

The size of the virtual address space is dependent on the system you're working on. You've probably heard the term **32-bit** or **64-bit** before, and this describes the size of the virtual address space, i.e.
- 2^{32} = 4,294,967,296 addresses = **4GB**
- 2^{64} = 18,446,744,073,709,552,000 addresses = **16 Exibytes**

64-bit machines really only use about 48 bits for each virtual address, which really equates to 256 TB of virtual space. The general trend is that modern virtual address spaces tend to be much much larger than the size of physical RAM that accompanies them (~16GB). As a result, **not all the virtual addresses will get mapped to physical ones**. 


## Why Vmem?
Virtual addresses themselves aren't super confusing, but they seem unneeded at first...so what's the whole point of having this huge address space that doesn't actually exist? It all really boils down to **memory protection**.

The virtual address space makes it seem to each process that it is the only entity on the system. This gives processes the illusion that it has access to more memory than the real amount the system contains, which is great for multiprocessing and virtual machines.

In the end, we don't want processes to be able to access each other's memory, or else applications might cause each other to crash, fault, etc. So even though two processes may contain information at the same virtual address, in reality those identical virtual addresses will be mapped to completely different parts of RAM, that way the processes don't interfere with each other.

![identical virtual addresses mapping to different physical addresses](/images/virtual-isolation.jpg)


A process can access all of its virtual addresses, AND ONLY A SUBSET OF THE VIRTUAL ADDRESS ARE ACTUALLY MAPPED TO PHYSICAL MEMORY. The operating system has the final say in what regions of physical memory a process can access. Hopefully, the regions of physical memory that processes map into are disjoint, that way they are isolated from each other.

## Paging
So we know the layout of both virtual and physical memory at this point. But wow do we actually translate/map virtual addresses to physical address? The process of doing so is called **paging** and it boils down to a mix of hardware and software. To start, we're going to look at paging from a simple view, i.e. how we would map a process's memory into **contiguous chunks on RAM**.

![contiguous allocations on physical memory for each process](/images/contiguous-ram.png)

One of the hardware components involed in paging is the **Memory Management Unit (MMU)**. It takes **virtual address and process id** as input and outputs a physical address. Think about why it takes PID in as an argument..

The whole point of the MMU is to do 2 jobs...

1. Do bounds checking on an address
2. Provide the physical offset for the running process's vmem

![simple hardware diagram for mmu that maps processes contiguously in physial mem](/images/mmu-hardware-diagram.png)

#### Bounds Checking
Bounds checking is a large part of memory protection. Essentially, it checks whether the virtual address being used in a load/store instruction (dereference) is in between a valid bound, otherwise the system will produce a segmentation fault. This could be easily done by the compiler, e.g. if we try to assign `int y` by dereferencing some int pointer `x`, the compiler could produce two sets of instructions...

![compiler with and without bounds checks](/images/bounds-checking.png)

The bottom left would unprotected code, whereas the code on the right performs a bounds check. This provides good memory protection, but the overhead of doing so IS INSANE. Think about all the extra instructions that are required for every load/store instruction! We don't want this! That's why the hardware of the MMU deals with this instead of the compiler...

#### Physical Offset
This is only relevant for when we are mapping entire processes contiguously into physical memory. I like to think about it in the following way...

Let's say we want to map virtual address 0x0. In order to access that in physical memory, assuming the bounds have allowed us, all we have to do is add some offset to the virtual address; the offset being the start of the contiguous chunk in physical memory. For other virtual addresses, you will keep translateing and adding the offset. That's exactly what the **relocation register** does in this diagram...it's a register because the offset will be different for different processes, so this register can be loaded with some different offset in that case. 

![offset for contiguous physical allocations](/images/mmu-offset.png)

Mapping entire process's into contiguous physical chunks is somewhat simple for the MMU because of these offsets. Contiguous chunks also provide a simple method of isolating different processes. HOWEVER, as more and more processes start to exist on the system, **external fragmentation** begins to build up.

This motivates a system that maps virtual addresses into **non-contiguous chunks**.


## Non Contiguous Physical Allocations (coming from contiguous virtual allocations)
This is more complicated. But, the whole motivation for doing this is because fragmentation is a pain in the ass for contiguous physical allocations. The aim is

![diagram of the noncontiguous mappings were aiming for](/images/noncontiguous-physical.png)


He brought up calling the buddy allocator for multiple rounds...let's say you're trying to allocate some size that ISN"T A POWER OF 2, you would allocate as large a power of two chunk as possible, and then call buddy again on the remainder. In doing this, you would probably get non-contiguous physical allocations.

![virtual address components for paging](/images/addressing-scheme.png)

I'm really confused at what the **page number** and **page offset** represent intuitively. But, I understand the breakdown at the bottom. The entire address contains either 32 or 64 bits. The LSB represent the page offset, and the number of bits used is depedent on the page size of the system (4kB pages means 12 bits will be used). The rest of the MSB will be dedicated the page frame number. The physical address that is mapped is just the output of the MMU function plus the offset (again, pid is accounted as an input as well).


