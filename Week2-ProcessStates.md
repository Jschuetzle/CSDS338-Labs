# What we're doing today
+ [Processes](#process)
+ [Process States](#process-states)
+ [Waiting Processes](#waiting)
+ [Background vs. Foreground](#bgfg)
+ [Useful Options and Commands](#options)

## Processes <a name = "process"></a>
Loui has probably mentioned what processes are in lecture. If you didn't pick up on it, processes are basically just programs in the operating system that want to run on the CPU. The CPU is the main hub of execution on your computer...it does more than just calculations. One of the many jobs an OS has is to manage what **processes** get to run on the CPU at certain times. So ultimately, think of processes as a unit of execution on the OS, where only one process can utilize a CPU *core* at at time (cores are what really matter). 

Many of you are probably familiar with the term *thread*, and I don't want you to confuse a process with a thread. Threads are also a unit of execution, but they are the *smallest unit of execution*. So by logic,
a processes can be composed of multiple threads, which very often is the case. There's a lot more details involved, but **we will many focuses on processes** for this class. Here's a quick 
[2 min video](https://www.youtube.com/watch?v=Dhf-DYO1K78) if you're interested in learning some more specifics.

![threads in a process block](/images/threads_in_process.png)

### Observing Processes
One of the most frequent tasks that are asked of you on homeworks is to use the command `top`. Running this command will output a table with information about all the current processes that exist on the system currently. Here's what that looks like on my chromebook which runs Debian Linux.

![top output](/images/top.png)

By default, the table updates every 3 seconds and you're able to scroll up and down to see all the processes if there are many. I think the columns that you should be able to understand at this point/after this lab are PID, USER, S, and COMMAND. There are a lot more columns that are available to display...you can change that by pressing 'f' and using your common sense/problem solving abilities from there. IMPORTANT:
in order to exit the top output, press `q` or `Ctrl+C`. We'll see in a second why `Ctrl+C` is important.

## Process States <a name = "process-states"></a>
The columns I really want to go over today are the S and USER columns. To start off, we'll talk about S, which refers to the **current state** of the process. 

Just because the processes are listed on the top output, it doesn't mean they are all running simultaneously. Like I said before, only one process can run on a CPU core at a time. Ok. So if we figure out the number of CPUs on the system, then we can determine the maximum # of processes out of this list that are actually running. If I type the command `lscpu`, then I can get some important info about the CPUs on my system.

![lscpu](/images/lscpu.png)

The output says my chromebook has 2 CPUs. Does that mean 2 CPUs each with 1 core? Or 1 CPU with 2 cores? I did a basic google search with the CPU name (Intel Celeron CPU N3350) and found that it really is 1 CPU with 2 cores (which seems more practical for a chromebook).

![cpu-search](/images/cpu_google_search.png)

We know that there must be a maximum of two processes running from this top output...but how do we find out which ones they are? That's where the current state comes into play. There are many different states that a process can be in, but the simple way to think about is that either

+ the process is on the CPU (a running state)
+ OR, the process is NOT on the CPU (a not-running state)

![process states diagram](/images/process_states.png)

In this diagram, all the non-running states are 
- new
- ready
- waiting
- terminated

There are more non-running states than this, but this list simplifies those ideas quite well. Hopefully this diagram reminds you of state diagrams from CSDS 281...it is just like that. For now, don't worry about the arrows that connect each of the states...we will learn more about that as we dive into scheduling in the next few weeks. BUT, I do want you to have some understanding about each of the states.

### Explanation of State Diagram
The **new** and **terminated** states should be pretty self explanatory. A process is in the new state when it is created (you'll learn in lecture what it means for a process to 'be created'), and is in a terminated state once it's completely done with it's task. Also trivial is the **running** state...this is just when a process is executing on the CPU. 

The states **ready** and **waiting** might seem slightly similar. If something is ready, but not running, isn't it technically waiting? I'd say yes. *But something that is waiting might not be ready to run*, and that's the key distinction. 

**When a process is ready** (runnable), **the process has all the resources it needs to run, but the CPU just isn't available**. The operating system deals with these 'ready processes' by placing them in a queue. When the CPU become available, the operating system will examine all the processes in that ready queue and will allow one of them to take the CPU. 

![simple ready queue diagram](/images/ready-queue.png)

**When a process is waiting** (sleeping), **the resources the process needs are not available**. Regardless of whether the CPU is open or not, a sleeping process relinquishes any possiblility of running on the CPU while it stays sleeping. When the resources become available again, a signal will be sent to the CPU, and that previously sleeping process will now become ready or will run. An example of a process that sleeps a lot is a process that reads input from the keyboard.

There are different types of sleeping states, but I won't talk about that now. [Here's a link](https://access.redhat.com/sites/default/files/attachments/processstates_20120831.pdf) if you'd like to read more about it.

## Waiting Processes <a name = "waiting"></a>
Waiting processes may seem slightly unimportant at first, but they actually save the processor a lot of time. A common example of processes having to wait is related to device controllers.

![diagram of CPU communication lines with devices](/images/device-controllers.png)

Each of these device controllers has its own registers and buffers, similar to the ones that are used on a CPU, but only for use within those device controllers. Instructions on the CPU don't perform behaviors/actions that happen on these device controllers, rather the CPU instructions tell the device controllers to get something done. We'll dive more into device speeds during the memory units, but device drivers are comparatively much much slower than the speed at which the CPU operates (except for caches, everything on the OS is comparatively slow).

![table of speeds for CPU and other parts of OS](/images/controller-speed.png)

If we were to accept these speeds, then here's what the execution flow might look like...

![cpu and controller activity diagram](/images/cpu-complete-wait.png)

I'll talk about this diagram in class, but essentially we are wasting what seems to be infinite time. In order for the operating system to be efficient, we essentially want to be operating at CPU speed (nanosecond scale) as much as possible. So how do we maneuver the issue of device controller speeds? This is done by **context-switching** and **interrupts**.

If a process requests for resources, it usually won't be able to continue it's execution until it receives the resources. In this situation, we can take the process off the CPU, i.e. **changing it's state from running to waiting**. This is an example of a context switch, when one process gets moved off the CPU, presumably for another process to run. Once the device controller obtains the resources, it will send an interrupt to the CPU, basically like a text message you get in the middle of class. 

![](/images/interrupt-diagram.png)

Once that interrupt is processed, we can put the previously waiting process into a ready state in order to execute once again on the CPU. You can see from the above diagram that we're saving a lot of CPU cycles with this new method of execution.

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
After compiling and running the program. We can use `top` to view what state the program is in, however the command isn't being interpreted and run immediately. You can try over and over, but nothing seems to happen.

&nbsp;

![top not working](/images/foreground.png)

&nbsp;

So what's going on here? This brings up the concept of the **foreground**. In your terminal, the foreground is like the spotlight. Just like how a spotlight can be quite focused, **only one process is allowed to occupy the foreground at a time**, kind of like how only one process can occupy a CPU core at a time (foreground is software based and CPU core is hardware based though). 

The foreground contains an active process that is *visible* to the user. While that foreground process is running, the shell (Bash) **isn't necessarily ignoring the commands we type in**, it just doesn't execute them right away. It will actually wait until the foreground is terminated until it will execute any of the subsequent commands typed in.

&nbsp;

![Bash shell waiting until hello world is done until it runs my ls](/images/shell_waiting_foreground.png.png)

&nbsp;

This seems super problematic though! How would I run a program and view it on `top` simultaneously? Doesn't that render `top` sort of useless? Well, there's two workarounds. The first would be to open a second terminal, that way you technically have two foregrounds to operate with. 

The second option is to send the process to the **background**. Running a command in the background 
means that the command is running in a way that allows the shell to continue executing other commands without waiting for the background process to finish. You can send a process to the background in two ways.  The first way is by adding an ampersand to the end of the command like so

&nbsp;

![Sending hello world to background and typing ls successfully](/images/background-ampersand.png)

&nbsp;

An alternative is to press `Ctrl+z` in order to pause the execution of the process. This will open up the foreground for you to type the command `bg` which then sends this process to execute in the background.
[Here's a stackoverflow post](https://stackoverflow.com/questions/19074956/what-happens-when-you-hit-ctrlz-on-a-process) I really liked that talked more in depth about pausing a process.

&nbsp;

![Pausing hello world and using bg](/images/background_pause.png)

&nbsp;

There will be moments this semester where the homework will ask you to run multiple instances of a program, or to investigate `top` while a program is running, so I think it's REALLY important to know how to send a program to the background. ALSO, the foreground and background ARE NOT PROCESS STATES! They are just mechanisms, or contexts, that the operating system manages.

## Useful Options and Commands <a name = "options"></a>
So far, every time we open `top`, it may seem daunting since there are so many processes on the screeen--many of which seem random and belong to *root* (kernel space). These processes aren't super important to us...yes, they are probably important to the system, but they are not important to us as users. 

I want to go over some useful options that you can use to make looking at `top` output a lot easier. An **option** is just an added command that follows the normal command you were intending to run. For example, if I wanted some more descriptive information of the files in the current directory, instead of typing just `ls`, I could type

```
ls -l
```
or
```
ls -a
```

The "-(insert char)" is the option. They act as parameters that you could pass to a command/program, kind of like how you can pass parameters to functions in any programming language. This implies that each command we run has some default execution version, and that version contains no options. 

If you aren't very familiar with a command and want to see some of the options it has available, you can type `man <cmd-name>` and the manual will output to the screen. Usually, this output is quite long, so I'd recommend you pipe the output to `less`, and then scroll down until you find the information you're looking for.

```
man top | less
```

One quick note, you can also stack options onto each other, e.g. `ls -tla`, which is equivalent to `ls -t -l -a`. This works as long as the options you stack don't require values, e.g. I wouldn't be able to group together the following options

```
top -d 4 -u jss270
```

#### Changing Delay
The default time delay for `top` is 3 seconds...meaning every 3 seconds the output will update. Usually, the homeworks will ask you to take a screengrab of top during your program's execution, and trying to take a screenshot with something like Snipping Tool can be a pain. So, this delay can be changed easily with the `-d` option. You can either specify the delay before you run the `top` command or change it interactively.

```
top -d 10  //updates info every 10 seconds
```

&nbsp;

![changing delay interactively](/images/top_delay.png)

&nbsp;

#### Simplifying the Processes on Top
Oftentimes you're just trying to view to behavior of a handful of processes rather than all the processes on the system. Removing the processes you don't need
can be really useful for you AND the TAs (who have to look and interpret your top output). Here are some useful options, some of which I think you should use everytime you need a top screengrab for homework. 

&nbsp;

| ![nonidle processes](/images/nonidle_option.png) | 
|:--:| 
| *Filtering out all processes that are idle* |

&nbsp;

| ![Processes by PID](/images/pid_option.png) | 
|:--:| 
| *Selecting processes with a specific PID* |

&nbsp;

| ![user's processes](/images/user_option.png) | 
|:--:| 
| *Displaying all processes pertaining to a specific user* |
