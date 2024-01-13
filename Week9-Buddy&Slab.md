# What we're doing today
+ [Buddy Allocator](#buddy)
+ [Buddy Implementation](#buddy-implementation)
+ [Problems with Buddy](#buddy-cons)
+ [Slab Allocator](#slab)

Last lesson we went over a simple implementation for `malloc()` and `free()` which involves strategies for memory allocation in user space. This week we're going to look at two specialized strategies that are more directed towards **allocation of kernel memory**. 

## Buddy Allocator <a name = "buddy"></a>
The buddy allocator is an example of a _power-of-two allocator_, i.e. all the allocation chunks that it will hand out are powers of two. The smallest power-of-two size that the buddy allocator can produce is 4kB, which makes it compatible with the physical frame and virtual page sizes. I think the best way to explain the buddy allocator is by example...

Let's say the total amount of memory that's available is 256kB and that no allocations have been made yet. The situation can be pictured as so

![clean slate of memory for buddy allocator](/images/empty-buddy.png)\

Let's say an allocation request of size 14kB is made, i.e. the call `buddy_alloc(14kB)`. The first thing the buddy allocator will do is search for the smallest possible power-of-two size that can fit the allocation. In this case, that size would be 16kB. If there doesn't exist a chunk of that size, then the buddy allocator will **break larger chunks of memory into smaller pieces** in order to fulfill the 14kB allocation. Here's what that breaking down of chunks would look like

![buddy allocator break chunks down](/images/buddy-search.png)

In class, I'm going to talk about how this diagram will change if two more allocations are made: the first being 70kB and the second being 90kB. In short, there won't be enough space for the 90kB allocation, **even though the total amount of available memory is more than 90kB**. In this situation, we can't coalesce free chunks that are different sizes because then we'll end up with a chunk that isn't a power-of-two. 

This seems problematic! Thinking about it more, the power-of-two sizes seem like they will generate A LOT of internal fragmentation. And on top of that, the buddy algorithm runs in logarithmic time...not even constant time! What could possibly be the benfit of this design? The benefit is that the buddy allocator is **very good at coalescing**.

![freeing a page in buddy](/images/buddy-free.png)

If we choose to free the original 14kB allocation we made, **we can examine its buddy** and determine if coalescing is possible. This can be done multiple times up the buddy tree until coalescing isn't possible, like in the following diagram.

![](/images/buddy-coalesce.png)

## Buddy Algorithm Implementation <a name = "buddy-implementation"></a>
Unlike `malloc()` and `free()`, the buddy algorithm is a bit more complicated, so I won't be coding an implementation. I do want to discuss the design though.

### Allocation
Just like we did with `malloc()`, we can use a free list to speed up allocation time. The free list will contain buckets of free chunks for each power-of-two size...kind of like using an array of linkedlists to implement a hashmap.

![free list table for buddy algorithm](/images/buddy-table.png)

If the chunk of the size we're looking for already exists, then we can allocate in constant time. However, if it doesn't exist, then we must go through the process of breaking up larger chunks of memory which is at worst logarithmic time.
The tricky thing about the buddy algorithm is that **we can't afford to embed** our management data structures within the allocations (like how we did with headers). So how does the buddy algorithm know where its buddy is?

### Freeing
We can cleverly use a **bitmap** to implement deallocation...

![bitmap for buddy deallocation](/images/buddy-bitmap.png)

In this design, each possible allocation size has a bit associated to it which describes whether an allocation has been made inside that chunk. In doing this, we can easily access the buddy of a specific chunk: each buddy pair starts with a chunk that has an even index, so the corresponding buddy is the next odd index. Coalescing is still very easy in the bitmap, and in using bits we can speed up our operations quite a lot. Think about how much memory this bitmap will take up...

## Bad Buddy <a name = "buddy-cons"></a>
So with buddy allocation, we can reduce external fragmentation by using power-of-two sizes and the overhead involved isn't that bad. However, the largest downside is that the smallest allocation size possible is 4096 bytes. Using only the buddy algorithm for all kernel allocations would be terrible because oftentimes the kernel is dealing with small sized objects...putting 100 byte objects into 4kB allocations won't go well.

We can obtain information about the current kernel objects on the system by looking at the `/proc/slabinfo`.

![cat /proc/slabinfo](/images/slab-output.png)

For now, we're not going to care about any columns with the name `slab`, and rather just focus on the active objects, num objects, and object size. From this information, we'll write a program that will calculate the total amount of internal fragmentation caused by the buddy allocation system. String processing in C can get ugly, so I wrote this program in python...

```
PAGE_SIZE = 4096
total_frag = 0

with open('slabinfo.txt', 'r') as file:
        for line in file:
                line.strip()
                columns = line.split()

                num_objs = int(columns[2])
                obj_size = int(columns[3])
                internal_frag = 0

                ## if this true, then frag is zero
                if obj_size % PAGE_SIZE != 0:
                        ## find correct page size...as power-of-two
                        correct_page_size = PAGE_SIZE
                        while correct_page_size  < obj_size:
                                correct_page_size *= 2

                        internal_frag = correct_page_size - obj_size

                total_frag += (internal_frag * num_objs)
print(f'Total fragmentation: {total_frag}')
```
![output of total fragmentation](/images/python-output.png)

This is a lot of wasted memory! Clearly the buddy allocator isn't used in some special cases...so what is?

## Slab Allocator <a name = "slab"></a>
Buddy allocation introduced us to a more restrictive allocation strategy, i.e. allocations had to be a power-of-two size. The  slab allocator is _even more_ restrictive becausee it only **allocate chunks of one single size**. This single size though can be beneficial for kernel allocations though; kernel objects (locks, sockets, and many more) tend to have a specific size, and if hundreds of these objects must be allocated, then the specialization of the slab allocator goes to good use. 

![empty slab cache/pool](/images/empty-slab.png)

This is a diagram of an empty slab, i.e. no allocations have been made. You might wonder how we got space for the slab already, and that's where the kernel will use the buddy allocator! In this case, each slab might be a certain number of pages--usually quite low like 1 or 2--and then with those pages the kernel objects will be stored by the slab allocator.

When we start to allocate kernel objects, the slab will start to look like so...

![first allocation done to slab](/images/slab-single-alloc.png)

The dark orange are allocated objects...the light orange are free. Keeping track of the free list as a singly linked list makes allocations/deallocations constant time operations, so very fast. We don't have to worry about coalescing for the slab allocator since it only deals with one single size. But, what will happen if many allocations occur and the slab becomes full?

![multiple slab pools](/images/multislab.png)

In that situation, the buddy allocator is used once again in order to create another slab. As a result, we'd have a **free slab list AND a free list for each slab**, however since many slabs may be full then their free list would be null. Likewise, when all kernel objects have been freed from a certain slab, that slab can be **returned to the OS and will be coalesced by the buddy allocator**. 

The final question I'll pose in class is, "What slabs would be more beneficial to allocate to/free from?"
