# What we're doing today
+ [Quick Syscall Review](#syscall)
+ [Examining `ls` Further](#ls)
+ [Forking](#fork)
+ [User Space vs Kernel Space](#user-vs-kernel)
+ [System Calls](#system-calls)

## System Calls Review <a name = "syscall"></a>
Last week we reviewed a lot of material. We went over 

+ a process's memory layout, specifically the stack
+ why the stack didn't occupy the highest possible memory addresses because of kernel space
+ how applications make requests for certain system calls to be made

In that last bullet the word *requests* is really important. System calls are *special instructions* only executable while in kernel mode (bit mode 0), so user applications aren't allowed to execute these instructions, rather they have to *request* that the instruction be executed. I want to stress, **in your C programs, you can utilize system calls in your code**, however, they will be executed in kernel space, not user space.

In one of the examples, we saw how information such as a syscall ID number can be passed to a register for future use in kernel space. This suggests that the operating system has a **table like structure** that stores all the possible system calls and their associated IDs. You can actually look up all the syscalls that the OS has with the command `man syscalls`

![output of man syscalls](/images/man-syscalls.png)

The list contains hundreds of system calls that are available for the operating system to use, and will even be relied upon in order to get back to user space to continue execution. 

 + **How do we figure out which system calls are being made?**

 ## Examining the `ls` Command <a name = "ls"></a>
In order to better answer this question, we're going to do a deep dive into how the `ls` command executes. Now, we don't have the source code of the `ls` command, nor does it seem evident where we'd find it. Some extremely helpful diagnostic tools can give us more information though! The commands `ltrace` and `strace` will intercept any library calls and system calls, respectively, during the execution of the `ls` command. We already have talked about what system calls are, but what are library calls? 

**Library calls** are just calls to any functions that reside in shared libraries (.so file extension) that reside in the `/lib` directory on the system. Think of this as using functions that are imported from some package in Java, e.g. the creation of an ArrayList would require a library call. Functions like `printf`, `malloc`, and `sleep` are examples of libcalls, and they can be executed in the normal user space stack.

We use `ltrace` and `strace` in the following ways...

```
ltrace ls
```

```
strace ls
```
If the operating system doesn't recognize those commands, you can install them with the package manager of whatever system you're using (for me, it's `apt install strace`). These commands won't necessarily output the source code, but they will output all the dynamic library calls and system calls that occur during execution, and this will inform us on the execution behavior of `ls`. I'm going to dive deeper into the output in class as opposed to here, but I will list some sequences of instructions that I think are important here.

#### strace Output
The very first line of our `strace` output tells us a lot about the nature of the `ls` command. 

![replacing current program for ls binary](/images/execve.png)

It seems that `ls` is already a binary executable on the OS, and it's stored in the `/bin` directory (which stands for binary). When we type in `ls` and press enter, that doesn't mean that this binary is executed right away though! The `execve()` reveals that it's actually getting *loaded* into the memory of the calling process. We'll talk about what the calling process could be in a little bit...but understand that the `execve()` call is allowing for execution of a new program. Loui has hopefully talked a little bit about forking and about process creation...`exec()` is different than forking in the following way...

```
One sometimes sees execve() (and the related functions described
in exec(3)) described as "executing a new process" (or similar).
This is a highly misleading description: there is no new process;
many attributes of the calling process remain unchanged (in
particular, its PID).  All that execve() does is arrange for an
existing process (the calling process) to execute a new program.
```

**So instead of birthing a new process, `exec()` overwrites it's user space memory with that of a new program**, in this case, that new program is `/bin/ls`.

| ![directory names getting written to standard out](/images/strace-write.png) | 
|:--:| 
| Directory names getting written to terminal...INSTANTLY! |


#### ltrace Output
| ![directory name getting copied to memory](/images/ltrace-copy-dirname.png) | 
|:--:| 
| Subdirectory name getting copied into heap |

| ![directory name getting written to stack](/images/ltrace-stack-writes.png) | 
|:--:| 
| Subdirectory names getting written into stack buffer |

| ![Contents of the stack buffer getting written to terminal](/images/ltrace-stack-writes.png) | 
|:--:| 
| Contents of the stack buffer getting written to terminal |






