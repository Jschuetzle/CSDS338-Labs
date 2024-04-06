# What we're doing today
+ [Process vs Thread](#pvst)
+ [Context Switching w/ Threads](#cs-threads)
+ [Page Tables](#table)
+ [Improving Page Tables](#tlb-and-hpts)


## Process Vs. Threads <a name = "pvst"></a>
For the entire semester this class has focused around the idea of **processes**. We think of processes as an entity for execution on the operating system: each gets an address space and gets execution time depending on how the scheduler treats it. However, you've been slightly lied to.

When talking about execution, or _flow of control_, we really are referring to **threads**. Most of you may already be somewhat familiar with the concept of a thread, but might be confused on the specifics of its implemetation. In simple words, threads represent a sequential execution of a program's code. Therefore when you introduce multiple threads, you achieve multiple sequential executions of the program's code. These executions could be covering the same lines of code or completely different lines. They could be executing simulaneously (in parallel) or concurrently (taking turns). This is a more conceptual explanation however...

An actual thread is really just an <a name="thread_context"></a>**execution context**. What exactly does that mean? Well, it's just a set of registers (register state) that signal where execution is happening. Here's a [great explanation](https://stackoverflow.com/questions/5201852/what-is-a-thread-really) I found online. Suppose you're reading a book, and you want to take a break right now, but you want to be able to come back and resume reading from the exact point where you stopped. One way to achieve that is by jotting down the page number, line number, and word number. So your execution context for reading a book is these 3 numbers.

If you have a roommate, and their using the same technique, they can take the book while you're not using it, and resume reading from where they stopped. Then you can take it back, and resume it from where you were. Threads work in the same way. In this example, a single book represents a process, and each person who currently is reading the book is a thread of that process. So, with a high level description, a process is just an address space with a collection of threads.

![process pointing to multiple threads](/images/process_and_threads.png)

However, a thread is slightly _more_ than just a register state. Each thread also gets it's own stack. So the memory layout of a multithreaded program really looks like this

![address space with two stacks, both above the heap](/images/multithread_stack.png)


## Context Switching and Race Conditions <a name = "cs-threads"></a>
Introducing the concept of threads should change the way we think about context switches. Previously, we understood that 'a process gets switched off the CPU' by saving the processes registers and copying over the new process registers. Now, it's essentially the same thing but just with threads.

The only caveat is that when we talked about processes getting switched, we had to change the virtual address space context in order to keep memory isolated between processes. Trivially, we won't need to do this if a context switch occurs between two threads of the same process. So what needs to change when this siutation occurs?

Well, every thread will have a [different stack and register](#thread_context) as discussed previously, so these things must change during a context switch. As a result, everything else in the address space doesn't have to switched out. This leads to threads sharing all the parts of the address space which are thread-dependent.

![table of what resources a thread shares vs owns](/images/thread_sharing.png)

More talk about types of threads...and then transition to race conditions due to sharing.
Information in the binary, the data segment, and the heap are shared since the information stored there isn't thread-dependent. Sharing information between threads might seem super beneficial, but in reality it opens the stage to **race conditions**.
