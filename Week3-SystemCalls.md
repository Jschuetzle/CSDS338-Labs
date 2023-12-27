# What we're doing today
+ [Process Memory Segments](#pms)
+ [Execution Stack](#stack)
+ [User Space vs Kernel Space](#user-vs-kernel)
+ [Process States](#process-states)
+ [Background vs. Foreground](#bgfg)
+ [Useful Options and Commands](#options)

## Process Memory Segments <a name = "pms"></a>
Today we're going to consider the simple program below

```
int foo(int a){
  int b,c;
  b = a + 5;
  c = b + a;
  return c;
}

void bar(void){
  int local;
  foo(10);
  //after
}
```
Hopefully you understand that each of the variables and functions in the following program are stored in memory...But what exact does *memory* mean? What does it look like? What all is stored in *memory* for 
this program? This brings up the idea of a process's **memory layout**. You might have already gone over this in class, but I want to stress that understanding the memory layout of a process/program is 
really important if you eventually want to end up doing low level programming with any OS/hardware (especially for memory constrained environments). 

![Process memory layout](/images/process-mem-layout.png)

Each program is going to be given a chunk of memory that is organized different sections like the image above. I wrote a little bit about each of the sections below...don't worry about remembering all of them,
BUT you should remember the location and direction of growth for the **heap** and the **stack**. I won't read over these in lab, but you're welcome to.

#### Text/Code
This section contains the text of the compiled program and also the binary instructions that the CPU will end up executing. It's read-only (think about why that is) and is placed in the low memory addresses
so that it won't be overriden by the growth of the heap/stack.

#### Initialized Data
This section contains your global and static variables...global variables being outside of any function (including main). Any global constants you define will probably be stored here, but constants in C can
be tricky (look up how #define works and where it's stored in memory, also watch [this video](https://www.youtube.com/watch?v=8a3HyL1VN0Q) about `const int` if you're interested).

#### Uninitialized Data
These are very similar to the initialized data section, however these variables have not been defined yet, e.g. `static int max;`.

### Heap
Any dynamically allocated memory that the program has will be located here. Since this value is dynamic, it can grow upwards or shrink downwards during the program (hence the direction of the arrows in the diagram).
We will learn what exactly is meant by dynamic allocation once we get more into allocation algorithms, but this is anything associated with *malloc, realloc, calloc,* and *free*. This is the area where memory
leaks happen, i.e. substantial amount of memory is allocated but not returned to the OS, and the total amount of memory (which is finite) runs out.  

### Stack
This section contains any locally scoped memory in the program. Since all functions have some scope to them, a stack structure works very well for signaling what variables are and aren't available at certain 
parts of the program...return addresses also fit nicely into the stack structure. Basically, any type of variables that are initialized/instantiated inside a function *without* the calls we talked about above
will be stored in the stack. Note that the stack grows DOWNWARD and shrinks upward, and the given back of memory doesn't have to be handled by the user like it is in the heap.


## Execution Stack Example <a name = "stack"></a>
Let's look back at the program that was brought up earlier. 
```
int foo(int a){
  int b,c;
  b = a + 5;
  c = b + a;
  return c;
}

void bar(void){
  int local;
  foo(10);
  //after
}
```
We know there aren't any global variables, and there aren't any calls to malloc, calloc, etc, so that must mean that during execution everything will be stored on the stack. But how exactly is it stored on the
stack? Here's a simple picture of what the stack might look like as the program executes

![general overview of stack that's highlighted](/images/static-stack.png)

If we use `gcc` (like `cc` but it has more capabilities) with the correct options, we can actually see what our C program looks like as assembly instructions. Ignore all the `.cfi` calls, and focus on just
the highlighted parts.

![assembly instructions for main](/images/stack-main.png)

The first few lines of code make sure that the base pointer (%rbp) lines up with the top of the new stack frame. It does this by pushing the stack down,
then storing the address of the previous base pointer in the this new area (storing base pointer info about the calling function). Then it updates the address of the base pointer to match that of the stack pointer
(they are the same since nothing from `main()` has been loaded yet). Finally, the stack is moved down by 16 bytes to anticipate any new memory that `main()` might need right away.  

The important part here is understanding the circled section of code. When we define the variable `int local = 10`, we're essential asking for 4 bytes of space--since ints are 32-bit in C--and since there is
some free space between the base pointer and stack pointer of the current frame, we will take the first 4 bytes that are lower than the base pointer. Next, INTERESTINGLY, the argument, 10, to the `foo()` function
is actually placed in a register (%edi) rather than the stack, which is different than what we predicted earlier. Like the `subq $16, %rsp` command, this is one of the many optimizations that the compiler will
make. Finally we make a call to the foo function which will (1) invisibly push the current address into stack memory, so we can return once foo completes and (2) invisibly perform an unconditional jump to the 
`foo` label.

Here is some of the assembly for the `foo` function. It is quite complicated because of the additions...please don't waste your time trying to understand it because the addition is intuitively simple. 
I've highlighted some parts that I think are worth looking at

![assembly instructions for foo](/images/stack-foo.png)

The first highlighted section, again, stores the previous stack frame's base pointer and levels the stack pointer and base pointer. All the space in between the two highlighted sections has to do with the
additions. The final section stores the result into register `%eax`, which I'm assuming represents the variable `c`. Again, this an is optimization because the values is getting stored in a register, rather
than storing it on the stack, which requires us to update the stack pointer. Finally, we move the stack pointer up by using `pop` and then we `ret` to the address that was previously stored. If you're more
interested in an explanation of some of the assembly instructions, [look here](https://www.cs.virginia.edu/~evans/cs216/guides/x86.html) and if you want to know more about stack frames, you can
[look here](https://rabbit.eng.miami.edu/class/519/frames.html).

## User Space vs Kernel Space <a name = "user-vs-kernel"></a>


