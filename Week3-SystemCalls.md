# What we're doing today
+ [Process Memory Segments](#pms)
+ [Execution Stack](#stack)
+ [User Space vs Kernel Space](#user-vs-kernel)
+ [System Calls](#system-calls)

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
  int local = 10;
  foo(10);
  local = 15;
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

#### Heap
Any dynamically allocated memory that the program has will be located here. Since this value is dynamic, it can grow upwards or shrink downwards during the program (hence the direction of the arrows in the diagram).
We will learn what exactly is meant by dynamic allocation once we get more into allocation algorithms, but this is anything associated with *malloc, realloc, calloc,* and *free*. This is the area where memory
leaks happen, i.e. substantial amount of memory is allocated but not returned to the OS, and the total amount of memory (which is finite) runs out.  

#### Stack
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
then storing the address of the previous base pointer in the this new area (storing base pointer info about the calling function). Then it updates the address of the base pointer to match that of the stack pointer (they are the same since nothing from `main()` has been loaded yet). Finally, the stack is moved down by 16 bytes to anticipate any new memory that `main()` might need right away.  

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
So far we've talked about all the sections of the program's memory layout, BUT these are just the sections in **user space**. Let's look at an actual process map to determine how large that chunk is compared with what the total chunk that the process is allowed to use. Type `top -u caseid` and copy/remember the PID of some random process on the output. We can look at that process's memory layout by typing

```
cat /proc/<pid>/maps | less
```
This now shows us the enitre memory layout of the process, with the large empty areas being condensed (DIFFICULT QUESTION TO PONDER: does these areas actually exist on RAM?). Notice that the addresses increase
as you scroll down, so if we want to see the very top address of user space, we can run `tac /proc/<pid>/maps | less` which just revereses the contents by line. Here's the output I got on my chromebook.

![top of user space in process map](/images/pmap.png)

In class, Loui discusses that the ratio of user space : kernel space is 3:1. THIS IS TRUE for 32-bit x86 Linux machines, by default. In reality, this ratio is different from system to system, and sometimes is
actually adjustable by a kernel parameter if you have permission to act as root. FOR MY OUTPUT, you can see the top of the stack ends at `7fff53c9e000`, which is about *half* the max of `ffffffffffff`. So for my
system, the real ratio is about 1:1 for user space to kernel space, which is true of many 64-bit x86 Linux machines. Here's a [stack exchange post](https://unix.stackexchange.com/questions/509607/how-a-64-bit-process-virtual-address-space-is-divided-in-linux) that seems to agree if you're more interested.

But what exactly is the *kernel space* and why is it necessary? 

Imagine I asked one of you to create an application like Powerpoint. Of course, the application would need to communicate with the display drives in order display the presentation. The application would need to
create a process for the current tab that was open, and maybe also have some architecture that utilizes multiple threads for different sections of the application. Finally, once the user is done working for the
day, the application would have to communicate to hard disk in order save all the progress. But, should a user trust that some random college kid did ALL OF THIS correctly? No way.

The operating system acts as protection for applications...otherwise, an application has to trust that other applications on the system will never make a impactful mistake. This is a topic that is extremely important for things like AWS, with servers containing multiple residents (multiple applications). If the faulty behavior of one company's application causes another completely unrelated company's application to crash, then this is very problematic. So, in theory, the operating system should be protected from the ill effects/malicious behavior of the applications in order to do it's job of keeping all applications safe.

So how is the kernel protected from the user? The first protective mechanism is done through the hardware with **dual-mode**. 

![system call with dual mode](/images/dual-mode.png)

The processor will contain a single bit (possibly in a protected register) that can be changed to denote whether it's in user mode or kernel mode. When the processor is on user mode, processes DO NOT have access
to all the memory on the system. This area of protected memory is called the **kernel space**. It stores memory that is necessary for the operating system to function...i.e. it is the memory of the operating system. Hopefully, you now have an understanding of the difference between user-level applications and the kernel.

## System Calls <a name = "system-calls"></a>
OK. User-level application don't have that much power then. BUT, how do we ever switch into kernel mode? It would seem silly to give user-level applications access to the mode bit...or else they would just
switch to bit to kernel mode and then access kernel space, which shouldn't be allowed. _So how does switching into kernel mode work?_

The solution are special processor instructions called **system calls** and **returning from system calls**. A system call is just a special processor instruction that allows an application to perform some behavior while switching the mode bit to 0, and starts executing at a fixed address that the operating system has set up solely for system calls (kernel space). This allows the kernel to control where we start executing so that it can define logic for doing that in safe way. Eventually, this leads to another instruction that the kernel can execute that returns back up to user level (mode bit 1) and continues from where execution left off. 

![dual mode diagram but with system calls and return from system calls highlighted](/images/dual-mode-highlight.png)

User-level applications don't have access to certain parts of memory, but they also don't have access to certain instructions. For example, you wouldn't want to give an application the opportunity to shut
down the operating system, otherwise it would basically hault the execution of all other programs. Or, e.g. you want to write some value to the terminal or some file on disk. **But how exactly is this system call made? Is a new stack frame created for the system call? Or something else?**

#### System Call Mechanics
We're gonna look at how calling something as simple as `printf()` ultimately leads to a system call.

![syscall mechanics for printf](/images/syscall-mech.png)

The purpose of something like `printf` is to print out some string to the terminal. First, that will invoke lib c, which is just some library that is combined and *linked* with your code (from  `#define <stdio.h>`) in order to produce a specific binary. In their logic, they end up calling the `write()` system call which writes standard out (the terminal). Before the system call is made, the user-program has to place information into either (1) specific registers or (2) the stack in order for the kernel to utilize those values. In this example, the program might have to load the ID of the syscall (ID=4) into a register, load the file descriptor (standard output with ID=1) into a register, and the string to be printed into a register. 

Then a call to **sysenter** or **int** (stands for *interupt*, ik...confusing) is made, which switches the mode bit to 0, switches to the kernel stack, and starts executing a kernel-defined system call handler. This will look up the syscall with ID=4, it sees that the function is `write()`, and then it will invoke the function. The syscall function will return to the user space by calling **sysexit** or **iret** which will return the number of bytes that were printed and then start executing where we left off. 

If you want to understand what's going on ini the diagram below, I recommend you watch [this video](https://www.youtube.com/watch?v=GoPTe_eIQwI&list=PLVW70f0xtTUxHXRtZhGEJAiBDFx-ofc_G) 51:47-56:05...the guy who makes these vids is awesome!

![complete syscall diagram for xv6 write](/images/syscall-xv6.png)

## Ideas I want you to be familiar with after today
+ Process Memory Layout
+ What information is stored in the Stack
+ Stack Frames (base pointer & stack pointer)
+ Purpose of the kernel
+ Dual-Mode Execution (user mode vs. kernel mode)
+ User space vs kernel space on process maps
+ What a system call is


