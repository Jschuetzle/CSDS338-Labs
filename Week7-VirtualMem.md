# What we're doing today
+ [What is `malloc()`?](#malloc-intro)
+ [Free List and Headers](#headers)
+ [Simple Implementation](#simple)


## Malloc Introduction <a name = "malloc-intro"></a>
Hopefully Loui has gone over malloc by now...I know I've brushed over it quite a bit in labs, but we're really going to dissect what goes on during the malloc function call today. By the name, it refers to memory allocation, that is, dynamic memory allocation. **Static memory allocation happens at compile time, and dynamic memory allocation happens at runtime.** So all the stack operations and assembly code we looked at in [week 3](/Week3-SystemCalls.md#stack) were related to static allocation.

![detailed memory layout](/images/heap-diagram.png)

Just like how the stack has a stack pointer that may not be crossed over into, the heap has a **program break** that may not contained dynamically allocated memory above it. There exist syscalls `sbrk()` and `mmap()` that can obtain/change the location of the program break...`malloc()` and it's companions are just library calls (executed in user space) that efficiently utilize these syscalls when necessary. We won't dive into these syscalls, as they can get quite technical. However, I think a good question to ponder is _does malloc always utilize syscalls during execution?_

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

We can create a simple implementation of `malloc()` like so...

```
#include <assert.h>
#include <string.h>
#include <sys/types.h>
#include <unistd.h>


void* malloc(size_t size) {
  if(size == 0){
    return NULL;
  }

  void* p = sbrk(0);
  void* request = sbrk(size);
  if (request == (void *) -1) {
    return NULL; // sbrk failed.
  } else {
    return p;
  }
}
```
The program will correctly increase the heap size with the call to `sbrk()` and will return the address of the start of the chunk. If you're confused with the `sbrk()` function call, here's what the linux man page says...

> "**sbrk()** increments the program's data space by _increment_ bytes. Calling **sbrk()** with an increment of 0 can be used to find the current location of the program break."
>
> "On success, **sbrk()** returns the previous program break. (If the break was increased, then this value is a pointer to the start of the newly allocated memory). On error, (void *) -1 is returned,"

We have a working version of `malloc()`, but what about an implementation for `free()`? C requires the programmer to deallocate memory once it's done being used or else we could run into some nasty problems in the future, so having a functioning `free()` is necessary for us to create. The prototype is

`void free(void* ptr);`

When free is passed a pointer that was previously returned from malloc, it's supposed to free the space. But given a pointer to something allocated by our malloc, we have no idea what size block is associated with it. This begs the question

+ **If `free()` only receives a pointer, how does it know how big the memory chunk to release is?**

## Free List and Headers <a name = "headers"></a>
To answer this question, we'll start with a very simple implementation of `malloc()` which utilizes the syscall `sbrk()`.




## Implementation for Malloc
We want to store things on the heap, but that requires bookkeeping, which also takes up memory on the heap. Kind of like who came first, the chicken or the egg. 

What can be done is the use of **headers**.
For every allocation done with `malloc()`, the size of that memory will be reserved. In order to keep track of how big the chunk is, a `struct header` is placed before EVERY allocation, and it has an int field called `memory_chunk_size`. That means we're spending 4 bytes before every allocation to store bookkeeping info about that allocation.

After that allocation, I can check whether the space succeeding it is allocated or not. If it is allocated, then I don't have to do anything. Otherwise, if it's free, then we need to add a `struct freelist
