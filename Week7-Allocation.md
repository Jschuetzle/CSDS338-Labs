# What we're doing today
+ [What is `malloc()`?](#malloc-intro)
+ [Headers](#headers)
+ [Free List](#freelist)
+ [Coding](#coding)


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

## Headers <a name = "headers"></a>
A common trick to work around this is to store metadata about a memory region in some space that we squirrel away just below the pointer that is returned. 

![headers for allocated space and free space](/images/malloc-headers.png)

Say the top of the heap is currently at 0x1000 and we ask for 0x400 bytes. Our current malloc will request 0x400 bytes from sbrk and return a pointer to 0x1000. If we instead save, say, 0x10 bytes to store information about the block, our malloc would request 0x410 bytes from sbrk and return a pointer to 0x1010, hiding our 0x10 byte block of meta-information from the code that's calling malloc.

So now, when we call `free(void* ptr)` on an address, all that needs to be done is to examine the header behind the malloc'd address in order to figure out how large the chunk is. You would have to ensure that each header is the same size, and you can do so by using a struct. 

```
 typedef struct header {
  int size;
} header;
```

**How do we actually free the memory though?**

## Free List <a name = "freelist"></a>
We theoretically could just pass in a negative value to `sbrk()` in order to free up memory. However, this would only make sense if the chunk of memory being freed is currently at the end of the heap. If the chunk was in between two other allocations, that would cut off any further chunks and render them useless because they would be past the program break. This is super problematic.

![chunk of memory becoming useless after a naive free()](/images/naive-free.png)

We could technically try to copy all of the memory that was above the newly freed hole, but there's no way to notify all of the pointers on the stack (pointing to the heap) that their addresses need to be adjusted. The copying and syscalls required to achieve this would also incur A LOT of overhead. 

The only solution is to mark that the block has been freed without returning it to the OS, so that future calls to malloc can use re-use the block. This is actually quite smart because now not every call to `malloc()` would require a syscall (additional overhead). Instead, `malloc()` can execute entirely in user space with the _cached memory_ in the heap.

In order to do this we'll need be able to access the metadata for each free block, just like how we did for each allocated block. For simplicity, we'll create only one struct header and use it for both allocated and free blocks...using two different struct headers can lead to differently sized headers, which can be confusing. For now, we added two attributes onto our original struct.

```
typedef struct header {
  int size;
  struct header* next;   //pointer to next header in free list
  int free;    //signals if header is preceding a free block
} free_header;
```
To make it easy for `malloc()` to identify which blocks are free, we can have all the free metadata structs be apart of a linked list, more commonly known as the **free list**.

![](/images/free-list.png)

Now, when `malloc()` is choosing a chunk of space for an allocation, it can first look through the free list and choose what it thinks is the best chunk for the job. In our case, we'll be using a _first fit_ policy, i.e. the first viable free chunk we find, we will use.

## Coding our own Malloc <a name = "code"></a>
Before coding, let's write down the methodology for both malloc and free...

#### Malloc
1. check preconditions for size argument
2. create a pointer for the final header
3. traverse the free list
  4a. if free header found,
    - set header as allocated
      
  4b. if not found,
    - request for more space
 
5. return address after header

#### Free
1. check preconditions for pointer argument
2. obtain pointer to header
3. set header as free

Here's the code I will write in class...I will also go over the helper functions and if there's time we'll test it out a bit.

```
typedef struct header {
        size_t size;
        struct header* next;
        int free;
} header;

#define HEADER_SIZE sizeof(header)

void* heap = NULL;   //points to BASE of heap

void* malloc(size_t size);
void* realloc(void* ptr, size_t size);
void* calloc(size_t nelem, size_t elsize);
header* find_free_block(header** last, size_t size);
header* request_space(header* last, size_t size);
void free(void* ptr);

void* malloc(size_t size){
        //check preconditions
        if(size <= 0){
                return NULL;
        }

        header* block;  //will become ptr to header 

        //special case where this is the FIRST malloc being called on heap
        if(!heap){
                block = request_space(NULL, size);
                if(!block) {
                        return NULL;    //request_space() failed
                }
                heap = block;
        } else {
                header* last = heap;    //only used when we can't find a free chunk

                block = find_free_block(&last, size);

                //no viable free blocks found
                if(!block){
                        block = request_space(last, size);
                        if(!block){
                                return NULL;    //request_space() failed
                        }
                } else {
                        block->free = 0;
                }
        }

        return block+1;    //return address after the header
}

header* find_free_block(header** last, size_t size){
        //start looking from base of heap...it's guaranteed to be header
        header* current = heap;

        //need current to be nonnull...keep traversing if we can't find viable chunk
        while(current && !(current->free && current->size >= size)){
                *last = current;
                current = current->next;
        }
        return current;
}

header* request_space(header* last, size_t size){
        //will store header for new space
        header* block;
        block = sbrk(0);

        //increase heap space
        void* request = sbrk(size + HEADER_SIZE);
        if(request == (void *) -1){
                return NULL;   //sbrk failed
        }

        //update the linkedlist
        if(last){
                last->next = block;
        }

        //initialize new header accordingly
        block->size = size;
        block->next = NULL;
        block->free = 0;
        return block;
}

void free(void* ptr){
        //check preconditions
        if(!ptr){
                return;
        }


        header* block_ptr = (header *)ptr - 1;
        block_ptr->free = 1;
}
```
