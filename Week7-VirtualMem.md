# What we're doing today
+ [Why Virtual Memory?](#why-vmem)


## Why Virtual Memory? <a name = "why-vmem"></a>
In class, Loui has talked about virtual memory and how virtual memory addresses _map_ into physical addresses. **If we end up using physical memory in the end though, what is the point of having virtual memory?**

This is just some thinking I'm putting down to be organized later...
+ Virtual memory is just an abstraction--an illusion--that allows processes believe that they own all the memory on the system
+ It's about isolating processes from each other's memory (be more specific)


## Implementation for Malloc
We want to store things on the heap, but that requires bookkeeping, which also takes up memory on the heap. Kind of like who came first, the chicken or the egg. 

What can be done is the use of **headers**.
For every allocation done with `malloc()`, the size of that memory will be reserved. In order to keep track of how big the chunk is, a `struct header` is placed before EVERY allocation, and it has an int field called `memory_chunk_size`. That means we're spending 4 bytes before every allocation to store bookkeeping info about that allocation.

After that allocation, I can check whether the space succeeding it is allocated or not. If it is allocated, then I don't have to do anything. Otherwise, if it's free, then we need to add a `struct freelist
