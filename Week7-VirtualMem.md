# What we're doing today
+ [What is `malloc()`?](#malloc-intro)
+ [Simple Implementation](#simple)


## Malloc Introduction <a name = "malloc-intro"></a>
Hopefully Loui has gone over malloc by now...I know I've brushed over it quite a bit in labs, but we're really going to dissect what goes on during the malloc function call today. By the name, it refers to memory allocation, that is, dynamic memory allocation. **Static memory allocation happens at compile time, and dynamic memory allocation happens at runtime.** So all the stack operations and assembly code we looked at in [week 3](/Week3-SystemCalls.md#stack) were related to static allocation.

![detailed memory layout](/images/heap-diagram.png)

Just like how the stack has a stack pointer that may not be crossed over into, the heap has a **program break** that may not contained dynamically allocated memory above it. There exist syscalls `sbrk()` and `mmap()` that can obtain/change the location of the program break...`malloc()` and it's companions are just library calls (executed in user space) that efficiently utilize these syscalls when necessary.

![man pages for malloc() and free()](/images/man-malloc.png)

Notice, malloc only returns _a pointer_ to the new allocated chunk of memory, exactly like a pointer to the beginning of an array. All of the memory in the allocated chunk can deferenced, but if you try to dereference an address that wasn't allocated, you're breaking the rules and [_might_ end with a segmentation fault](https://stackoverflow.com/questions/6441218/can-a-local-variables-memory-be-accessed-outside-its-scope/).

```
#include <stdlib.h>
#include <stdio.h>

int main() {
  char* arr = srbk(3);
  arr[0] = 's';
  arr[1] = 'h';
  arr[2] = '\0';

  printf("%c\n", arr[1]);    //prints 'h'
  printf("%c\n", arr[10]);   //possible segmentation fault
}
```


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
