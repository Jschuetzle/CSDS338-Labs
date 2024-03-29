# Welcome to OS!

## What we're doing today
+ [Connecting to EECSlab](#server-connect)
+ [Navigating the terminal](#terminal)
+ [Creating C programs](#text-editing)
+ [Compiling and Running Code](#compile-run)
+ [Student Questionnaire](#q)

## Connecting to the Server <a name = "server-connect"></a>
This class is mainly centered around the Linux Operating system, so there may be moments on homeworks where it isn't sufficient to use Mac or Windows (or at least it's more work). As a result, **eecslab** is just a Linux server for students to use for this class. In order to connect, open your laptop's terminal and type in 

```
ssh CASEID@eecslab-#.case.edu
```

The `ssh` command basically connects your computer to the server using the Secure Shell method. The second part of the command includes your username and the address to the server. The number following the word eecslab in the address just refers to which specific eeclab server you want to connect to. We have 4 servers (1-4), and if one of them has crashed, try to connect to one of the other servers. 

Once the server recognizes you, it will ask for your password, which is just your normal password associated with your caseid. IMPORTANT NOTE: for security reasons, the terminal doesn't show your password as you enter it, so don't be confused when no text is showing up.

## Terminal <a name = "terminal"></a>
After connecting to the server, you'll probably see a blinking line and some kind of address that is to the left of the cursor. 

&nbsp;

![Terminal Screen](/images/terminal_init.png)

&nbsp;

This is the bash shell. **Bash** is just an command-line (terminal) interpreter that runs on the server...basically, an application that interprets Linux commands that are entered through the keyboard. Bash is like any other process on the system, and it's job is to read your commands, perform necessary behaviors to get the command to run, and display the output of the command, if there is any. You can actually see bash running when we type the command `ps`, which displays all the active processes on the server.

&nbsp;

![Active Processes](/images/ps.png)

&nbsp;

The output of this can be confusing if you're not used to operating systems and the terminal, but I promise it's simple and we'll explain it a bit more after I introduce some more commands.

### Moving Around
So far, the terminal seems quite boring. It's not clear what resources are available and it might seem like an old computer that does nothing. We can clear that confusion up by moving around the server just like how you'd move around the file system on your computer. Probably the most useful and frequently used commands in all of Linux is `ls`. This command outputs all the contents of the directory (or folder) that you are currently in. 

&nbsp;

![ls on Root Directory](/images/root_ls.png)

&nbsp;

If the output from this command is empty, don't worry! That just means the directory is empty. For instance, any time you log onto the server for the first time as a new user, the initial directory you're located in is the `/home` directory, and it will be empty since you've never been on the server before (there actually is never an empty directory, but don't worry about that for now). You can see the full name of the current directory you're in with the command `pwd` (print working directory). 

&nbsp;

![Working Directory](/images/pwd.png)

&nbsp;

OK...you know how tell what directory you're in, and how to view the contents of that directory. But what else is there to do? 

Well, again just the file explorer on your computer, you can move around to different directories. The file system on the server follows a tree structure, so you can move yourself UP and DOWN the tree. Since our current working directory is empty, there are no child directories for us to open. Let's start by going up the tree instead. You can move to a new directory with the command `cd` (change
directory) followed by the name of the directory. In almost all operating systems, going up the tree encounters a special case, where the name of the directory is `..`. So, in order to go all the way to the top, enter the command 

```
cd ..
```

until you reach the root. Here's what that might look like

&nbsp;

![Move to root directory](/images/cd_toroot.png)

&nbsp;

Notice how I started in my home directory, and moved up all the way until the root. If I try to move up any further, notice how it will still keep me in the root directory. From here, we can `ls` to see the contents of the entire server. We won't need to be familiar with all of these directories, but some of them can be worth getting familiar with (like `/proc` or `/bin`). Let's move back to our home directory by doing either one of the following. 

1.
```
cd /home/caseid
```

2.
```
cd ~
```

The trailing tilde is just a shortcut that references the current user's home directory. Since this directory is empty to start off with, let's put something in it. This can be done with the command `mkdir` (make directory) followed by whatever you want to name it. Once you create the child directory, move into it. Here's what that sequence will look like...

&nbsp;

![directory creation](/images/make_directory.png)

&nbsp;

## Making a Hello World Program <a name = "text-editing"></a>
So far we've navigated the directories, but how do we create programs? The process of creating programs can be quite different depending on what operating system you use, but for this class we can use a text editor called **Vim** in order to create and edit a file. In order to open a new or already existing file, type the command

```
vim helloWorld.c
```

Just a heads up, this class isn't very coding heavy, and on many of the homeworks you are allowed to use whatever languages/tools you'd like. BUT, C is one the best langauges to communicate to the operating system with since it's so low level, so for almost all the demos I will be coding in C. Therefore, the `.c` file extension must be added onto the file name.

After typing the command, you see an empty window pop up. This is where you'll write the hello world program. BUT, you'll quickly notice that *almost* anything you type doesn't show up on the screen. This has to do with Vim.

&nbsp;

![vim logo](/images/vim.png)

&nbsp;

That's because we aren't in the correct Vim mode. Vim is different than most editors in that it contains different modes and **encourages the user to not use their mouse**. This makes the text editor super light weight and can speed up programming once you get used to it...but it's extremely annoying and frustrating as a beginner. *You are not required to edit your program in Vim* for this class. But at the minimum, you will either need to copy and paste your code into a file using Vim, or will have to use a simiilar editor called Nano. 

So, how are we going to write our hello world program if we can't type. Enabling typing is easy: just press `i`. This will put you into `INSERT` mode, which allows for typing. You be able to see the change in modes in the bottom left of the window. Once you're in `INSERT` mode, type the following program.

```
#include <stdio.h>

void main(){
  printf("Hello World\n");
}
```

## Compiling and Running Hello World <a name = "compile-run"></a>
Now that the program is written, how do we run the program? Well, the first thing you need to do is save the file (make a write). You can can exit INSERT mode by pressing the escape key. Then, type ":wq". These characters will show up at the bottom of the editor; the 'w' stands for write (save) and the 'q' stands for quit. Here's what that should look like

&nbsp;

![Vim exit screen](/images/vim_exit.png)

&nbsp;

So the program has been written, but how do we run it? Since C is a compiled language, rather than an interpreted language like Python, all we have to do is compile the hello world program and run the executable file it generates. The default C compiler on eecslab can be used like so

&nbsp;

![Hello world compile](/images/helloWorld_compile.png)

&nbsp;

Notice the extra file `a.out` that is created after compiling. This is an executable file that contains only machine code (binary) for the processor to execute. This is a large reason why C is such a fast programming language...all the instructions for the programs lifecycle have been determined before runtime. In order to run our program, just type the command below and check that the program correctly outputs `Hello World`.

&nbsp;

![Hello world output](/images/helloWorld_run.png)


## Student Questionnaire <a name="q"></a>
The level of experience that students have coming into this class can be vastly different. Some students already have a year or two of command-line experience and others have none (as was my case). The class will seem overwhelming at times and you may reach points where you have no idea what's going on, but I want to help you in those moments. 

I'd appreciate if you could fill out this quick form that just gives some info on your background experience...that way I can change the level of difficulty in labs if need be. Thanks, and best of luck for the semester!

![Form](https://forms.gle/copbnHrDHFBgQPNf7)
