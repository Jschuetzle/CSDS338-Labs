# What we're doing today
+ [Virtual Memory](#vmem)


## Virtual Memory <a name = "vmem"></a>

The Virtual Memory Address space gives each process the illusion that it is the only entity utilizing memory, therefore each process believes it has access to all memory

Before, with bounds checking, two processes couldn't access the same addresses AND get their memory. With a VAS, you can have 2 processes that are both accessing addresses within what they think is their heap, and it will access their heaps memory, not the other processes heap's memory even though it's the same virtual address.

Virtual addresses are just the address that we use to do loads and stores. Proper management of VAS will ensure that faults will occur if we try to access memory that should be inaccessible for that moment


A process can access all of its virtual addresses, AND ONLY A SUBSET OF THE VIRTUAL ADDRESS ARE ACTUALLY MAPPED TO PHYSICAL MEMORY. The operating system has the final say in what regions of physical memory a process can access. Hopefully, the regions of physical memory that processes map into are disjoint, that way they are isolated from each other.

## Paging
How do we convert virtual addresses to physical address? You use the **Memory Management Unit (MMU)**. This is a piece of hardware. It takes **virtual address and process id** as input and outputs a physical address...PID added in there so processes can be kept isolated.

![diagram of mmu and cache of addresses](/images/mmu-placement.png)

The cache could be either on the right or the left of the MMU in theis diagram. If it's on the left, it stores virtual addresses, and contents of the cache are dedicated to a single process, in order to keep isolation. Therefore, if there's a context switch (switched processes) this cache must be flushed which is a HUGE overhead. Instead, we can keep it on the right, but I not really sure what it stores and what the arrow pointing the 'memory' means

## MMU Implementation
If we're an MMU, how do we decide to map process's memory into physical memory? We could give each process it's own **contiguous chunk**, so it's kinda of like malloc in a sense. However, this can lead to really poor memory organziation/utilization...fragmentation, poor predictions of how much memory a process will use...could overuse, could underuse. BUT it does a great job of providing protection, and it's simple. 

![simple hardware diagram for mmu that maps processes contiguously in physial mem](/images/mmu-hardware-diagram.png)

The reason why contiguous mmu mappings are easy to implement in hardware is because of offsets. In this diagram, the bounds register will check the bound on the virtual address, like normal.

Let's say we want virtual address 0. In order to access that in physical memory, assuming the bounds have allowed us, all we have to do is add some offset to the virtual address. This offset is the start of the contiguous chunk in physical memory. For other virtual addresses, you will keep adding the offset. That's exactly what the **relocation register** does in this diagram...it's a register because the offset will be different for different processes, so this register can be loaded with some different offset in that case. 

Accessing those register is a sensitive instruction, therefore it would only be performed by the kernel
