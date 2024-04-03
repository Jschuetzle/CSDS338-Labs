# What we're doing today
+ [Virtual Memory](#vmem)
+ [Paging](#paging)
+ [Page Tables](#table)
+ [Improving Page Tables](#tlb-and-hpts)


## Virtual Memory <a name = "vmem"></a>
Yet again, we've ran into another OS abstraction that can intuitively be confusing. Loui has definitely talked about virtual memory at this point, but I want to go over it once more, as it is crucial to memory managament.

**Virtual memory** is just a fabricated address space given to every single process on the system...think of it as a seperate entity from the physical RAM on the system. Both the virtual address space and the physical address space are cut up into some granularity of size, which is usually 4kB large (4096 bytes). 

![virtual pages to physical frames](/images/page-frame.png)

We refer to these 4kB spaces as **pages** when they are in the VAS and **frames** when they are in the PAS. Ultimately, each page address gets **translated** into a physical memory address, that way the contents of virtual addresses exist in some shape or form on RAM. 

The size of the virtual address space is dependent on the system you're working on. You've probably heard the term **32-bit** or **64-bit** before, and this describes the size of the virtual address space, i.e.
- 2<sup>32</sup> = 4,294,967,296 addresses = **4GB**
- 2<sup>64</sup> = 18,446,744,073,709,552,000 addresses = **16 Exibytes**

Note: 64-bit machines really only use about 48 bits for each virtual address, which really equates to 256 TB of virtual space. The general trend is that modern virtual address spaces tend to be much much larger than the size of physical RAM that accompanies them (~16GB). As a result, **not all the virtual addresses will get mapped to physical ones**. 


## Why Vmem?
Virtual addresses themselves aren't super confusing, but they seem unneeded at first...so what's the whole point of having this huge address space that doesn't actually exist? It all really boils down to **memory protection**.

Virtual memory lures each process into thinking that it is the only entity using memory on the system. This gives processes the illusion that it has access to more memory than the real amount the system contains, which is great for multiprocessing and virtual machines.

In the end, we don't want processes to be able to access each other's memory, or else applications might cause each other to crash, fault, etc. So even though two processes may contain information at the same virtual address, in reality those identical virtual addresses will be mapped to completely different parts of RAM, that way the processes don't interfere with each other.

![identical virtual addresses mapping to different physical addresses](/images/virtual-isolation.jpg)


A process can access all of its virtual addresses, AND ONLY A SUBSET OF THE VIRTUAL ADDRESS ARE ACTUALLY MAPPED TO PHYSICAL MEMORY. The operating system has the final say in what regions of physical memory a process can access. Hopefully, the regions of physical memory that processes map into are disjoint, that way they are isolated from each other.

## Paging
So we know the layout of both virtual and physical memory at this point. But how do we actually translate/map virtual addresses to physical address? The process of doing so is called **paging** and it boils down to a mix of hardware and software. To start, we're going to look at paging from a simple view, i.e. how we would map a process's memory into **contiguous chunks on RAM**.

![contiguous allocations on physical memory for each process](/images/contiguous-ram.png)

One of the hardware components involed in paging is the **Memory Management Unit (MMU)**. It takes **virtual address and PID** as input and outputs a physical address. Think about why it takes PID in as an argument..

The whole point of the MMU is to do 2 jobs...

1. Do bounds checking on an address
2. Provide the physical offset for the running process's vmem

![simple hardware diagram for mmu that maps processes contiguously in physial mem](/images/mmu-hardware-diagram.png)

#### Bounds Checking
Bounds checking is a large part of memory protection. Essentially, it checks whether the virtual address being used in a load/store instruction (dereference) is in between a valid bound, otherwise the system will produce a segmentation fault. This could be easily done by the compiler, e.g. if we try to assign `int y` by dereferencing some int pointer `x`, the compiler could produce two sets of instructions...

(#compiler-bounds)
![compiler with and without bounds checks](/images/bounds-checking.png)

The bottom left is unprotected code, whereas the code on the right performs a bounds check. This provides good memory protection, but the overhead of doing so IS INSANE. Think about all the extra instructions that are required for every load/store instruction! We don't want this! That's why the hardware of the MMU deals with this instead of the compiler...

#### Physical Offset
This is only relevant for when we are mapping entire processes contiguously into physical memory. I like to think about it in the following way...

Let's say we want to map the first virtual address. In order to access that in physical memory, assuming the bounds have allowed us, all we have to do is add some offset to the virtual address; the offset being the start of the contiguous chunk in physical memory. For other virtual addresses, you will keep translating and adding the offset. That's exactly what the **relocation register** does in this diagram...it's a register because the offset will be different for different processes, so this register can be loaded with some different offset in that case. 

![offset for contiguous physical allocations](/images/mmu-offset.png)

Mapping entire process's into contiguous physical chunks is somewhat simple for the MMU because of these offsets. Contiguous chunks also provide a simple method of isolating different processes. HOWEVER, as more and more processes start to exist on the system, **external fragmentation** begins to build up.

This motivates a system that maps virtual addresses into **non-contiguous chunks**.

### Non Contiguous Physical Allocations
This is more complicated. But, the whole motivation for doing this is because fragmentation is a pain for contiguous physical allocations. Here is a diagram showing what the situation looks like...

![diagram of the noncontiguous mappings were aiming for](/images/noncontiguous-physical.png)

A really simple implementation of this is to break each process's VAS into segments, e.g. the binary, the code, the heap, the stack. In order for the MMU to map these segments without overlap, we can assign each of the segments some ID. If we do this, all the bits in the virtual address will be composed of 1) the segment ID and 2) the offset of how far into the segment a certain address is. 

(#segment-id)
![segments of vmem getting mapped](/images/noncontiguous-segments.png)

This isn't truly how the segments are split up though, but I think it provides as a great stepping stone. In reality, the segments in the OS are all the **pages**, each sized at 4096 bytes. So the 12 least significant bits (2<sup>12</sup> = 4096) will be devoted to the offset into each page, and the rest of the bits will act as a "segment ID" or "page number". 

![virtual address components for paging](/images/addressing-scheme.png)

So now we've invented this system that allows for non-contiguous physical memory allocations, and fragmentation won't be a worry as it was before. In addition, memory protection/isolation is provided to each of the virtual pages, as the MMU makes sure they can't overlap. However, **have we provided memory protection for allocations of size less than 4096?**

## Page Tables <a name = "table"></a>
One of the diagrams above display how a **bounds register** and **relocation register** work with contiguous chunks. This worked well because each process's VAS was treated entirely as a segment, therefore each process only needed a single bounds register and a single relocation register. However, when we start to segment the VAS, we would need a bounds register and relocation register **for each page**. Based on the amount of pages that exist in a VAS, THIS IS BASICALLY IMPOSSIBLE to implement in hardware. Registers are extremely fast, but the tradeoff is that they are extremely complicated, which is the reason why there are only about 20-40 registers per modern CPU.

We need another solution. That solution is **page tables**.

![basic diagram of page tables](/images/page-table.png)

Each segment of the VAS (each page) will get it's own entry in the page table, and each entry will store the physical address that is being mapped to. Try not to confuse the MMU and the page table...the MMU is hardware that simply does a conversion from virtual to physical. On the other hand, page tables are simply an array that stores all the mappings for each page with some additional information (such as whether the page is valid and its permissions). Page tables are managed by the kernel and can be expanded accordingly if more virtual addresses are created. In order to isolate virtual pages of differing processes from each other, **each process has its own page table**. 

But, regular page tables have huge downsides that we'll spend the rest of lab talking about...

## Improving Page Tables <a name = "tlb-and-hpts"></a>
We're going to discuss the cons of page tables in terms of _latency_ and _storage size_.

#### Latency
If we look back at how the [compiler dealt with keeping loads/stores protected](#compiler-bounds), the number of instructions increases about 5x. When we doing loading/storing with page tables, the number of instructions will **at least double**. The first instruction will access the page table in order to get the physical frame, and the second instruction will complete the load/store. As you'll see with the more modern page table designs that we'll talk about shortly, the number of instructions may be 2-4x greater in some situations.

Is there any way to reduce this latency? The solution is the **translation lookaside buffer**. The TLB is _a cache for page translations_ and is contained inside the MMU. But how does using a cache in this situation speed things up? Wouldn't you still have to spend an instruction accessing an element in the cache?

![diagram of TLB](/images/tlb.png)

The trick is that the TLB is a **hardware cache**. It is solely implemented by circuits, which allows for parallelization, i.e. all of the entries in TLB are examined simultaneously. The TLB a really complicated piece of hardware, and as a result the number of entries in the cache is extremely small (something like 64 entries). 

Since the TLB is hardware, it can operate extremely fast, and therefore we can include that functionality with _any call_ to a load/store. As a result, if a TLB hit occurs, then it will only cost us _1 instruction_.


#### Storage Size
A naive approach to page tables is thinking about them as an _array of mappings_. If this was true, then we can calculate it's predicted size...

Since each virtual page should technically have an entry in the page table, we can **divide the entire virtual space by 4kB, and then multiply by how many bytes each address has**. On a 32-bit system,

+ 2<sup>32</sup> / 2<sup>12</sup> = 2<sup>20</sup> pages
+ 2<sup>20</sup> * 4 bytes = 4MB

So each page table would contain nearly 4 MEGABYTES worth of CONTIGUOUS SPACE...this is terrible! And that's just for one process! If we multiple this number by the number of processes on the system, then nearly 1/3 of the entire RAM will just contain page tables...

The solution is **hierarchical page tables**.

![diagram of hpt](/images/hpt.png)

Hierarchical page tables are structured like a tree. The root of the tree will contain entries the point to _inner page tables_. There could be two or three levels of indirection, but eventually the _leaf nodes will act as normal page tables_...containing actually mappings. 

If you do the calculations, we actually aren't decreasing space...storage size is really increasing! So how is this design better? HPT's take advantage of the fact that a large number of virtual addresses are invalid, i.e. they are apart of the large holes in the VAS. Therefore, we can nullify entries in the root page table, which then destory the inner page tables that are pointed to. This saves us a lot of space! Here's a great [2 min video](https://www.youtube.com/watch?v=8kBPRrHOTwg) that explains it about as simply as possible.

## Advice
I think this is probably one of the most confusing topics in OS, and this lesson certainly took me a long time to write this lesson (as I wasn't super knowledgeable about Vmem). Even though it's confusing, I think it's worth spending the time to study this lesson until you feel like you have a decent grasp on virtual memory and paging. 

From here on out, Loui's lessons will be quite dependent on an understanding of these topics, and there's less probability you will get confused in lecture if you know what's going on in this lesson. Again, just my advice... :)


