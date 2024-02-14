# What we're doing today
+ [Quick Syscall Review](#syscall)
+ [Examining `ls` Further](#ls)
+ [Bash as a Calling Process](#bash)
+ [Parent vs Child Execution Order](#parent-child)


## System Calls Review <a name = "syscall"></a>
Last week we reviewed a lot of material. We went over 

+ a process's memory layout, specifically the stack
+ kernel space and special permissions associated with kernel mode
+ how applications make requests for certain system calls

In that last bullet the word *requests* is really important. System calls are *special instructions* only executable while in kernel mode (bit mode 0), so user applications aren't allowed to execute these instructions, rather they have to *request* that the instruction be executed. I want to stress, **in your C programs, you can utilize system calls in your code**, however, they will be executed in kernel space, not user space.

In one of the examples, we saw how information such as a syscall ID number can be passed to a register for future use in kernel space. This suggests that the operating system has a **table like structure** that stores all the possible system calls and their associated IDs. You can actually look up all the syscalls that the OS has with the command `man syscalls`

![output of man syscalls](/images/man-syscalls.png)

The list contains hundreds of system calls that are available for the operating system to use, and will even be relied upon in order to get back to user space to continue execution. 


 ## Examining the `ls` Command <a name = "ls"></a>
In order to better understand system calls, we're going to do a deep dive into how the `ls` command executes. Now, we don't have the source code of the `ls` command, nor does it seem evident where we'd find it. Some extremely helpful diagnostic tools can give us more information though! The commands `ltrace` and `strace` will intercept any library calls and system calls, respectively, during the execution of the `ls` command. We already have talked about what system calls are, but what are library calls? 

**Library calls** are just calls to any functions that reside in shared libraries (.so file extension) in the `/lib` directory on the system. Think of this as using functions that are imported from some package in Java, e.g. the creation of an ArrayList would require a library call. Functions like `printf`, `malloc`, and `sleep` are all examples of libcalls, and their stack frames can reside in user space.

We can actually see these .so files in the memory map if we run any program from HW1! Here's what my memory map looked like after running a program that extends the stack through recursion and goes to sleep in the final function call...

![memory map with shared object files from glibc](/images/shared-object-files.png)

My program didn't make any references to `malloc` because I simply extended the stack through recursive calls, and I didn't use the `printf` function. The culprit here is my call to `sleep`. If you use the manual pages on terminal, i.e. `man sleep`, you'll see the name of the command and a number in parentheses right next to it. That number can tell us whether the function is a system call (2) or a library call (3). In the case of `sleep` there's a one, which is confusing, but if you look a couple lines down you'll see that it refers to `sleep(3)`, meaning that `sleep` is a library call.

We use `ltrace` and `strace` in the following ways...

```
ltrace ls
```

```
strace ls
```
If the operating system doesn't recognize those commands, you can install them with the package manager of whatever system you're using (for me, it's `apt install strace`). 

These commands won't necessarily output the source code, but they will output all the dynamic library calls and system calls that occur during execution, and this will inform us on the execution behavior of `ls`. I'm going to dive deeper into the output in class as opposed to here, but I will list some sequences of instructions that I think are important here.


### strace Output
The very first line of our `strace` output tells us a lot about the nature of the `ls` command. 

![replacing current program for ls binary](/images/execve.png)

It seems that `ls` is already a binary executable on the OS, and it's stored in the `/bin` directory (which stands for binary). You can even verify this by navigating to the `/bin` directory, and typing the following command.
```
ls -l | grep ls
```

If this command is extremely confusing, don't worry...I don't expect you to know everything going on. `grep` simply just picks out any line that contains the word I typed after...kind of like Ctrl+f in Chrome. On my output, you can see the third file that got listed is the executable binary for the `ls` command...

![ls binary](/images/binls.png)

When we type in `ls` and press enter, that doesn't mean that this binary is executed right away though! On the strace output, the `execve()` reveals that it's actually getting *loaded* into the memory of the calling process. We'll talk about what the calling process could be in a little bit...but understand that the `execve()` call is allowing for execution of a new program. Loui has hopefully talked a little bit about forking and about process creation...`exec()` is different than forking in the following way...

```
One sometimes sees execve() (and the related functions described
in exec(3)) described as "executing a new process" (or similar).
This is a highly misleading description: there is no new process;
many attributes of the calling process remain unchanged (in
particular, its PID).  All that execve() does is arrange for an
existing process (the calling process) to execute a new program.
```

**So instead of birthing a new process, `exec()` overwrites it's memory layout with that of a new program**, in this case, that new program is `/bin/ls`.

| ![directory names getting written to standard out](/images/strace-write.png) | 
|:--:| 
| Directory names getting written to terminal...INSTANTLY! |


### ltrace Output
I think the `ltrace` is less insightful than the `strace` output, and I won't spend that much time on it. But, here are some interesting actions going on under the hood. 

| ![directory name getting copied to memory](/images/ltrace-copy-dirname.png) | 
|:--:| 
| Subdirectory name getting copied into heap |

| ![directory name getting written to stack](/images/ltrace-stack-writes.png) | 
|:--:| 
| Subdirectory names getting written into stack buffer |


## Investigating the 'Calling' Process <a name = "bash"></a>
Remember from the strace, that the `exec()` call simply overwrites the memory of the current process with that of a new program--the new program being `/bin/ls`. But what exactly is the *current process* in this situation?

+ **Who is running before `/bin/ls` is loaded into memory?**

We can't simply get this information from `strace ls`, because `strace` will only tell us which syscalls are made _after_ `ls` process begins executing, instead of _before_. That's where `bash` comes into play. 
<!--
During the first week, I talked briefly about [Bash](Week1Intro.md#terminal) as an interpreter for the commands you type in the terminal. It's job is to read the command you typed and perform specific actions in order for the program to run successfully. After that is done, it must listen and wait for any future commands. This is great, **but how do you prove that Bash is calling `ls`?**.

We can do this by **using strace on bash!** After running this command

```
cat | strace bash
```

all the system calls that Bash executes are printed, BUT the final line seems unfinished...

![final line of strace for bash](/images/strace-bash.png)

This is odd. If you look at the documentation (`man read`), you'll see that the function should take 3 parameters. So why might it be getting cut off? Well, the single parameter that IS listed represents a file descriptor. We're not going to dive deep into what a file descriptor is, but it's super important that it's value is 0. The file descriptor with value 0 is the standard input for the terminal...meaning that **the strace output didn't finish because bash didn't finish...** bash is currently waiting for input from the user, which makes sense!

We can abuse this by typing in `ls` and running it, which leads to the following output...

![ls command getting read](/images/stracebash-ls.png)
-->
### Forking
Ok! So Bash must be the calling process? Not yet... Remember that `exec()` we talked about? If you scroll through the output of this strace, you'll see that there are *no invocations* of `exec()`. So we went through all this trouble and we still don't know who the calling process is. But the following syscall reveals what's really going on here...

![strace bash output clone call](/images/stracebash-clone.png)

If you look at the documentation of this function (`man clone`), you'll read that this function is very similar to `fork()`. For our purposes, they are basically the same. We know that `fork()` creates an entirely new child process that is basically a duplicate of the parent process, i.e. both memory spaces have the same content (the only difference being the value that is returned). 

What's important here is that the next line of output **prints the contents of the directory...This must mean that `ls` has been executed!** So, this clone had to have been the *calling process*, i.e. the process where `execve()` syscall was invoked and where `ls` executed. We can ensure that the child finished it's execution by looking at the return value of the `clone()` syscall, which is the PID of the child. 

![strace bash output clone call](/images/stracebash-wait.png)

From the screenshot, we see that the PID is `16745`, and just a few syscalls after the `clone()`, there are two calls to `wait()`. If you read the documentation for this command (`man wait`), you'll read that it's job is to wait for the state of the child to change (probably from running to sleeping or running to terminated). If it successfully does this, the return value is the PID of the child process...**which is the same PID as the return value from `clone()`**.

From all of this, we can determine the sequence of events to be
+ Bash interprets the text in the terminal (standard input)
+ Bash fetches the `ls` binary executable
+ Bash clones, creating a child process
+ Child process executes `ls` and terminates
+ Bash goes back to listening to terminal

![Life cycle of `ls`](/images/ls-lifecyle.png)

## Parent/Child Execution Order <a name = "parent-child"></a>
While I was messing around with the strace outputs, I noticed that they weren't always consistent. In the example I put in these notes, you saw that the directory contents were written to the terminal _directly after_ the call to `clone()`. BUT, sometimes they appeared to cut off the `wait()` syscall like so.

![Directory contents printed out later, during wait() syscall](/images/strace-parentfirst.png)

**This has to do with execution order**, specifically the execution order after a `fork()`. We know after a fork, a duplicate process is made that has an entirely new PID. This means the scheduler has to choose between either (1) the parent, (2) the child, or (3) some other runnable process, i.e. the scheduler treats the parent and child as different entities. 

In our previous example, directly after the call to `clone()`, the contents of the directory had been written to the terminal. Therefore, in that specific example, it was the child who ran first. However in this example, the `bash` process seems to keep running directly after the `clone()` call. It's not until `bash` **waits** that the directory contents are printed. Therefore, in this new example, it was the parent who ran first. 

This is a great example of how the order of execution between the parent and child processes is **indeterminant**. If you really want to ensure yourself of the order, instead of running `ls`, you can run `strace ls` in order to also see the system calls of the child process. 
