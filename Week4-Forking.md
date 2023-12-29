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

 In that last bullet the word *requests* is really important. System calls are *special instructions* only callable while in kernel mode (bit mode 0), so user applications aren't allowed to call these instructions,
 rather they have to *request* that the instruction be executed. In one of the examples, we saw how information such as a syscall ID number can be passed to a register for future use in kernel space. This suggests
 that the operating system has a **table like structure** that stores all the possible system calls and their associated IDs. 

 You can actually look up all the syscalls that the OS has with the command `man syscalls`

 ![output of man syscalls](/images/man-syscalls.png)

 The list contains hundreds of system calls that are available for the operating system to use, and will even be relied upon in order to get back to user space to continue execution. 

 + **How are we supposed to know which system calls to request for though?**
 + **How do we figure out which system calls are being made?**

 ## Examining the `ls` Command <a name = "ls"></a>
 In order to better answer this question, we're going to do a deep dive into how the `ls` command executes. Now, we don't have the source code of the `ls` command, nor does it seem evident where we'd find it.
 Some extremely helpful diagnostic tools can give us more information though! The commands `ltrace` and `strace` will intercept any library calls and system calls, respectively, during the execution of the `ls`
 command. They can be used like so

```
ltrace ls
```

```
strace ls
```
If the operating system doesn't recognize those commands, you can install them with the package manager of whatever system you're using (for me, it's `apt install strace`). These commands won't necessarily output
the source code, but they will output all the dynamic library calls and system calls that occur during execution, and this will inform us on the behavior of `ls` while it's executing. I'm going to dive deeper
into the output in class as opposed to here, but I will list some sequences of instructions that I think are important here.

#### Ltrace output
| ![directory name getting copied to memory](/images/ltrace-copy-dirname.png) | 
|:--:| 
| Subdirectory name getting copied into heap |

| ![directory name getting written to stack](/images/ltrace-stack-writes.png) | 
|:--:| 
| Subdirectory names getting written into stack buffer |

| ![Contents of the stack buffer getting written to terminal](/images/ltrace-stack-writes.png) | 
|:--:| 
| Contents of the stack buffer getting written to terminal |




