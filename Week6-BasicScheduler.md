# What we're doing today
+ [Scheduler Design for FIFO](#design)
+ [Creating a Queue](#queue)
+ [Set Up for Simulation](#setup)
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

For simplicity, and pointer practice, we're going to create a linked-list based queue. This will require a `struct queue` and a `struct node` to encapsulate whatever we choose to store in the queue.

```
typedef struct queue{
        node* head;
        node* tail;
} queue;

typedef struct node {
        proc* process;
        struct node* next;
} node;
```

Since constant time operations are desired for enqueueing and dequeuing, the `struct queue` stores a pointer to the head and a pointer to the tail. We can use the `struct node` as a container for storing a process. Conceptually, it wouldn't make sense to have a data member in the `struct proc` called next, because the process isn't guaranteed to be in a queue, and we really only want to store information about the current process in `struct proc`.

If you are not familiar with `typedef`, don't worry. Essentially, it adds a new name for an existing type, that way we don't have to type `struct struct_name` everytime--making the code more readable. Note though, it _doesn't_ create a new type, just an alias to an already existing type.

### Queue Initialization
This is not necessary, but we can create something similar to a Java constructor that will return an empty queue, except we don't have to return anything here since the argument is a pointer.

```
void init_queue(queue* q){
        q->head = NULL;
        q->tail = NULL;
}
```

### Enqueue
Enqueueing is the same as insertion into the end of a linked list. The 3 basic things that must be done to enqueue:
+ create a new node
+ point the current tail toward the new node
+ set the new node as the tail

This piece of code gets those essentials done along with taking care of the case where we `enqueue()` into an empty queue.

```
bool enqueue(queue* q, pcb* process){
        //creation of new node
        node* newnode = malloc(sizeof(node));
        if(newnode == NULL){
                return false;   //return false if malloc fails
        }
        newnode->process = process;
        newnode->next = NULL;

        //make current tail point to this new node...check first if there was a tail to begin with
        if(q->tail != NULL){
                q->tail->next = newnode;
        }
        q->tail = newnode;

        //case where the node is being enqueued into an empty list
        if(q->head == NULL){
                q->head = newnode;
        }
        return true;
}
```

### Dequeue
Dequeue is the same as deleting the first node in a linked list. The essential parts are:
+ Store the contents of the current head
+ Move the head forward
+ Delete the previous head and return its stored contents

Here's the following code...

```
pcb* dequeue(queue* q){
        if(q->head == NULL){
                return NULL;
        }

        //store the contents of current head...for free and return
        node* temp = q->head;
        pcb* result = temp->process;

        //move the head forward
        q->head = q->head->next;
        if(q->head == NULL){
                q->tail = NULL;  //special case where queue contains no elements after dequeue
        }

        free(temp);
        return result;
}
```

We need to test that the queue works, but we don't have our `struct proc` defined yet. For simplicity, define `struct proc` with one int member and create a couple instances. Enqueue all processes, and then use a while loop to dequeue all processes while printing out the returned structs data member values, and make sure these values match with what was enqueued.


## Set Up <a name = "setup"></a>
Now that we've created a working queue, we're going to focus on what should be contained in `struct proc` and how we will 'load a process on the CPU'. I say that in air quotes because the CPU is simply a pointer to a struct process and switching simply changes the value of the pointer. **Normally, performing a context switch is much much more complicated.** More [here](https://www.youtube.com/watch?v=iDJ4RuaJEOQ&list=PLVW70f0xtTUxHXRtZhGEJAiBDFx-ofc_G&index=6) if you're interested (47:00-1;02:00)

![diagram of specifics of a context switch](/images/context-switch.png)

Trivially, each process will need a PID and some amount of execution time. It would also be good for future analysis to include when the process is created and when it finishes execution (**total turnover time** and **waiting time**). 

```
typedef struct process_control_block {
        int pid;
        int arrival;
        int burst;
        int finish;
} proc;
```
Now, we have all the components to create the scheduling simulation. First we must create a queue, the 'CPU', and some other variables/counters. Those will include a clock for time, a pid generator, and a counter to keep track of the \# of processes that have finished.

```
int main() {
        queue q;
        init_queue(&q);

        proc* cpu;

        int time = 0;
        int pid = 0;
        int completed = 0;
}
```

We can initialize some `proc`s so that they are present in the queue at the beginning of the simulation, but that means they must have some execution time already defined. This brings up the issue of how we'll assign execution time. In different types of scheduling schemes, a **time slice** is often assigned to a process, and once that time slice is used up, the process is preempted. OUr basic implementation though will allow each process to run until termination, therefore we must define the amount of time a process takes to execute until termination.

This _burst time_ can be assigned randomly once a process enters a queue. I created a function that generates a random burst time...for simplicity, it will be between 1 and 5 clock units.

```
#include <stdlib.h>
#define BURST_LOWER 1
#define BURST_UPPER 5

int rand_burst() {
        return (rand() % (BURST_UPPER - BURST_LOWER + 1)) + BURST_LOWER;
}
```

And now, the processes will be initialized and enqueued

```
int main() {
        //... previous code

        proc p1 = { .pid = pid++, .arrival = 0, .burst = rand_burst() };
        proc p2 = { .pid = pid++, .arrival = 0, .burst = rand_burst() };

        enqueue(&q, &p1);
        enqueue(&q, &p2);
}
```

## Coding the Scheduler <a name = "sched"></a>
There's many ways to run the simulation. You could run the simulation until the queue is empty (just make sure the rate of incoming procs is less than the rate of termination procs). You could run until a certain number of processes have completed. You could run until a certain timestamp. For simplicity, I have chosen the timestamp method.

I will paste the code here, with comments, and talk more thoroughly about it in class.

```
#define MAX_TIME 50
int total_turnover = 0;

while(time < MAX_TIME){
        cpu = dequeue(&q);

        //process has completed execution once this loop is done
        for(int i=0; i<(cpu->burst); i++){
                time++;

                //for every tick, I want there to be a chance that a new process goes in the queue
                //the average time for a new process to spawn needs to be greater than average time for process to finish,
                //or else queue will never stop

                int r = (rand() % 100) + 1;
                if(r < 35){
                        //create new pcb, and enqueue
                        pcb* temp = malloc(sizeof(pcb));
                        temp->pid = pid++;
                        temp->arrival = time;
                        temp->burst = generate_burst_time();

                        enqueue(&q, temp);
                }
        }

        completed++;
        cpu->finish = time;
        total_turnover += (time - cpu->arrival);

        printf("Terminated process\n");
        printf("PID: %d    Arrival: %d    Burst: %d    Finish: %d\n", cpu->pid, cpu->arrival, cpu->burst, cpu->finish);
        printf("Current Time: %d    Processes Executed: %d    Total Turnover: %d\n\n\n", time, completed, total_turnover);    
}

printf("Average Turnover Time: %f\n", (double)total_turnover / completed); 
```

This code actually has a segmentation fault...we will figure it out in class. If you'd like, try to debug it on your own! (Hint: it has to do with an empty queue)
