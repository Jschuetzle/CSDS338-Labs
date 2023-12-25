# What we're doing today
+ [Processes](#process)
+ [Observing Processes](#top)
+ [Process States](#process-states)
+ [Background vs. Foreground](#bgfg)
+ [User Space vs Kernel Space](#spaces)

## Processes <a name = "process"></a>
Loui has probably mentioned what processes are in lecture. If you didn't pick up on it, processes are basically just programs that are running on the CPU. Remember, the CPU (and GPU) are really the only place on 
the computer where stuff gets done. One of the many jobs an OS has is to manage what **processes** get to run on the CPU at certain times. So ultimately, think of processes as a unit of execution on the OS,
where only one process can utilize a CPU *core* at at time (cores are what really matter). 

Many of you are probably familiar with the term *thread*, and I don't want you to confuse a process with a thread. Threads are also a unit of execution, but they are the *smallest unit of execution*. So by logic,
a processes can be composed of multiple threads, which very often is the case. There's a lot more details involved, but **we will many focuses on processes** for this class. Here's a quick 
[2 min video](https://www.youtube.com/watch?v=Dhf-DYO1K78) if you're interested in learning some more specifics.

![threads in a process block](/images/threads_in_process.png)

## Observing Processes
One of the most frequent (and easy IMO) things that are asked of you on homeworks is to use the command `top`. Running this command will output a table with information about all the current processes on that
exist on the system currently. Here's what that looks like on my chromebook which runs Debian Linux.

![top output](/images/top.png)

By default, the table updates every 3 seconds and you're able to scroll up and down to see all the processes if there are many. I think the columns that you should be able to understand at this point/after this lab 
are PID, USER, S, and COMMAND. There are a lot more columns that are available to display...you can change that by pressing 'f' and using your common sense/problem solving abilities from there. IMPORTANT:
in order to exit the top output, press `q` or `Ctrl+C'. We'll see in a second why `Ctrl+C` is important.

ADD A LITTLE EXPLANATION OF OPTIONS HERE IF YOU THINK THERE'S TIME TO DO SO.

## Process States <a name = "process-states"></a>
The columns I really want to go over today are the S and USER columns. To start off, we'll talk about S, which refers to the **current state** of the process. Just because all of these process are listed on the
top output, it doesn't mean they are all running simultaneously. Like I said before, only one process can run on a CPU core at a time. Ok. So if we figure out the number of CPUs on my system, then we can
determine the maximum # of processes out of this list that are actually running. If I type the command `lscpu`, then I can get some important info about the CPUs on my system.

![lscpu](/images/lscpu.png)

The output says my chromebook has 2 CPUs. Does that mean 2 CPUs each with 1 core? Or 1 CPU with 2 cores? I did a basic google search with the CPU name (Intel Celeron CPU N3350) and found that it really is
1 CPU with 2 cores (which seems more practical for a chromebook).

![cpu-search](/images/cpu_google_search.png)

We know that there must be a maximum of two processes running from this top output...but how do we find out which ones they are? That's where the current state comes into play. There are many different states that
a process can be in, but the simple way to think about is that either

+ the process is on the CPU (a running state)
+ OR, the process is NOT on the CPU (a not-running state)

![process states diagram](/images/process_states.png)

In this diagram, all the non-running states are 
- new
- ready
- waiting
- terminated

There are more non-running states than this, but this list simplifies those ideas quite well. Hopefully this diagram reminds you of state diagrams from CSDS 281...it is just like that, but each state is too 
complicated to be represented by a bunch of bits. For now, don't worry much if you don't understand the arrows that connect each of the states...we will learn more about that as we dive into scheduling in
the next few weeks. BUT, I do want you to have some understand about each of the states.

### Explanation of State Diagram
The **new** and **terminated** states should be pretty self explanatory. A process is in the new state when it is created (you'll learn in lecture what it means for a process to 'be created'), and is in a 
terminated state once it's completely done with it's task. Also trivial is the **running** state...this is just when a process is executing on the CPU. The states **ready** and **waiting** might seem slightly 
similar. If something is ready, but not running, isn't it technically waiting? I'd say yes. *But something that is waiting might not be ready to run*, and that's the key distinction. 

**When a process is ready** (runnable), **the process has all the resources it needs to run, but the CPU just isn't available**. The operating system deals with these 'ready processes' by placing them 
in a queue. When the CPU become available, the operating system will examine all the processes in that ready queue and will allow one of them to take the CPU. **when a process is waiting** (sleeping), **the
resources the process needs are not available**. Regardless of whether the CPU is open or not, a sleeping process relinquishes any possiblility of running on the CPU while it stays sleeping. When the resources
become available again, a signal will be sent to the CPU, and that previously sleeping process will now become ready or will run. An example of a process that sleeps a lot is a process that reads input from the
keyboard.

There are different types of sleeping states, but I won't talk about that now. [Here's a link](https://access.redhat.com/sites/default/files/attachments/processstates_20120831.pdf) if you'd like to read more 
about it.

## Background vs. Foreground <a name = "bgfg"></a>
Let's take our Hello World program and put it into a sleep state. We can do that explicitly by calling `sleep(100)` in our program after we print to the console. Note that we need to include a new header file
(like a Java import) in order for the sleep to properly work. 

```
#include <stdio.h>
#include <unistd.h>

void main(){
  printf("Hello World\n");
  sleep(100);
}
```
After compiling and running the program. We can use `top` to view what state the program is in, however the our command isn't being interpreted and run immediately. You can try over and over, but nothing 
seems to happen.

![top not working](/images/foreground.png)

So what's going on here? This brings up the concept of the **foreground**. In your terminal, the foreground is like the spotlight. Just like how a spotlight can be quite focused, **only one process is allowed to
occupy the foreground at a time**, kind of like how only one process can occupy a CPU core at a time (foreground is software based and CPU core is hardware based though). The foreground contains an active process
that is *visible* to the user. While that foreground process is running, the shell (Bash) **isn't necessarily ignoring the commands we type in**, it just doesn't execute them. It will actually wait until the
foreground is terminated until it will execute any of the subsequent commands I typed in. 

![Bash shell waiting until hello world is done until it runs my ls](/images/shell_waiting_foreground.png.png)

This seems super problematic though! How would I run a program and view it on `top` simultaneously? Doesn't that render `top` sort of useless? Well, there's two workarounds. The first and more annoying workaround
would be to open a second terminal, that way you technically have two foregrounds to operate with. The second option is to send the process to the **background**. Running a command in the background 
means that the command is running in a way that allows the shell to continue executing other commands without waiting for the background process to finish. You can send a process to the background in two ways. 
The first way is by adding an ampersand to the end of the command like so

![Sending hello world to background and typing ls successfully](/images/background-ampersand.png)

An alternative is to press `Ctrl+z` in order to pause the execution of the process. This will open up the foreground for you to type the command `bg` which then sends this process to execute in the background.
[Here's a stackoverflow post](https://stackoverflow.com/questions/19074956/what-happens-when-you-hit-ctrlz-on-a-process) I really liked that talked more in depth about pausing a process.

![Pausing hello world and using bg](/images/background_pause.png)

There will be moments this semester where the homework will ask you to run multiple instances of a program, or to investigate `top` while a program is running, so I think it's REALLY important to know how to 
send a program to the background.

## User Space vs Kernel Space <a name = "spaces"></a>
