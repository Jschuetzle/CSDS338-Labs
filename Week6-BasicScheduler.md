# What we're doing today
+ [Scheduler Design for FIFO](#design)
+ [Creating a Queue](#queue)
+ [Coding the Scheduler in C](#sched)

## Scheduler Design Discussion for FIFO <a name = "design"></a>
Today, we're going to design and code a scheduler. For simplicity, we're going to create a scheduler that follows the FIFO (First-in First-out) policy. Before we start coding, we're going to discuss all of the following ideas and see if/how we should implement them. The goal is to get a basic prototype, and once that is done we can more easily add complexity to our working version, instead of writing out a complex system right from the getgo.

+ Single vs Multi Processor?
+ \# of Ready Queues?
+ CPU-Bound and I/O Bound Processes?
+ Priority?
+ Is there a possibility for starvation?
+ Context Switching?

Regardless of whether we chose FIFO, the design would probably have to include some type of queue. For simplicity, it's a better idea to only do a **single processor with a single ready queue**, which is what we'll start with. However, it's very impractical to design a single processor system, as it's  much slower and basically obsolete in current operating system environments. If we do decide to design a multi-processor system, an essential question that must be asked is how many ready queues will I have?

![multiple ready queues with multiple processors](/images/multi-queues.png)

![single ready queue with multiple processors](/images/single-queue.png)

Differentiating between **CPU-bound and I/O-bound processes** isn't necessary if we want simplicity. BUT, incorporating them for a FIFO policy is quite important because it shows us the flaws. In theory, a FIFO scheduler will allow execution of a process until it completes, and we know from latency charts that I/O-bound processes will waste thousands/millions of CPU cycles if they are allowed to run until completion without getting kicked off. Implementing some form of CPU-burst and I/O burst into the scheduler would be good to show where our design fails, and how we could further improve it.

**Priority** isn't used in FIFO policy...the real "priority" is the arrival time of the process in the ready queue because the ordering of the processes is the ultimate determinant of when they will execute.

There is no chance of **starvation** if we use FIFO. Note, the difference between having to wait a REALLY REALLY long time and true starvation. Some processes inevitable might have to wait milliseconds to seconds for the I/O-bound processes to finish execution. BUT, we can guarantee that _eventually_ all processes will get to run and will complete termination. Therefore, starvation is impossible with FIFO.

Finally, **context switching** adds complexity, so this isn't our first focus. But, along with burst vs. I/O, it's important to implement because it gives a solution for the I/O bound processes that hog the CPU. **Would we implement preemption or yielding, or both?** Also, would we need to change/add to the queueing design?


## Coding a Queue in C <a name = "queue"></a>
Because we're utilizing FIFO, that is closely associated to using a double sided queue. However, C doesn't provide any standard libraries for data strucutres like Java does (Java Collections). There do exist data structure libraries, but they are far fewer and less popular, and using them would make us miss out on an opportunity to get experience coding in C, so the first task today is to code a queue.

![queue diagram](/images/queue.png)

For simplicity, and pointer practice, we're going to create a linked-list based queue.
