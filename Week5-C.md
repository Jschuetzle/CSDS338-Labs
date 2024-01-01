# What we're doing today
+ [C vs. Java](#c)
+ [Pointers and Addresses](#ptr)
+ [Structs](#struct)
+ [Parameter Passing](#parameter)
+ [Memory Allocation](#malloc)

## C vs Java <a name = "c"></a>
I know the background experience for students in this class can be diverse...some students have worked in multiple languages and some only one language. However, I know everyone here has some level of experience with
Java, and I want to introduce C in terms of how it's different than Java. 

### Types/Architecture
The primitive types between Java and C are similar, but further reduced in C. The main types are **char, int, float,** and **double**. If you wanted to use something like a boolean type in C, you'd have to use the
`#include` directive, which is similar to import statements in Java. Since there is no builtin support for booleans, the expressions inside if statements are really just evaluating to numbers such as 0 or 1.

!['true' statement in C evaluating to 1](/images/c-statements.png)

The biggest difference though is that **C is not object-oriented**. Although C has structs, which are similar to objects, there doesn't exist an idea of classes/objects that contain data members and methods in C. 
We're really only using basic functionality like if statements, loops, and functions in most of our programs. 

### Memory
Another big difference in C is that it is a _lower level language_ than Java. There are not as many abstractions in C, and you will often find yourself having to think about the memory of the program more than you
would in Java. The last primitive type that you need to be familiar with are **pointers**. Pointers are just memory addresses, often in hexadecimal, that represent a location in memory **where each address 
stores 1 byte of information**. So, the two things you're often dealing with in C are _addresses_ and the _values at each address_. This is a lot different than Java, where you don't have to deal with raw 
memory addresses at all. For example, Java abstracts memory allocation by using the `new` keyword whenever a new object is made, and we'll see later how this is dealt with in C.







