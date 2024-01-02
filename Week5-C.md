# What we're doing today
+ [C vs. Java](#c)
+ [Pointers and Addresses](#ptr)
+ [Structs](#struct)
+ [Memory Allocation](#malloc)
+ [Parameter Passing](#parameter)

## C vs Java <a name = "c"></a>
I know the background experience for students in this class can be diverse...some students have worked in multiple languages and some only one language. However, I know everyone here has some level of experience with
Java, and I want to introduce C in terms of how it's different than Java. 

### Types/Architecture
The primitive types between Java and C are similar, but further reduced in C. The main types are **char, int, float,** and **double**. If you wanted to use something like a boolean type in C, you'd have to use the
`#include` directive, which is similar to import statements in Java. Since there is no builtin support for booleans, the expressions inside if statements are really just evaluating to numbers such as 0 or 1.

!['true' statement in C evaluating to 1](/images/c-statements.png)

The biggest difference though is that **C is not object-oriented**. Although C has structs, which are similar to objects, there doesn't exist an idea of classes/objects that contain data members AND methods. 
We're really only using basic functionality like if statements, loops, and functions in most of our programs. 

### Memory
Another big difference in C is that it is a _lower level language_ than Java. There are not as many abstractions in C, and you will often find yourself having to think about the memory of the program more than you
would in Java. The last primitive type that you need to be familiar with are **pointers**. Pointers are just memory addresses, often in hexadecimal, that represent a location in memory **where each address 
stores 1 byte of information**. So, the two things you're often dealing with in C are _addresses_ and the _values at each address_. 

This is a lot different than Java, where you don't have to deal with raw memory addresses at all. For example, Java abstracts memory allocation by using the `new` keyword whenever a new object is made, and we'll see later how this is dealt with in C.


## Pointers* &Addresses <a name = "ptr"></a>
Just like how I can create an int, I can also create a _pointer to an int_.

```
int main(){
  int a = 5;
  int* ptr = &a;
  //or
  int* ptr = (int *) malloc(10*sizeof(int));
}
```

The only difference is that when I create a pointer, there must be an address for me to point to. Technically, I could just assign some arbitrary memory address to the pointer like I would for the int.
```
int a = 5
int* ptr = 0x1247a0e09f;  //this is technically allowed
printf("%d", *(ptr));
```
But this is usually TERRIBLE practice and will most likely lead to a segmentation fault. After running the code above, I got hit with one of these bad boys...

```
Segmentation fault (core dumped)
```

A segmentation fault basically means the program tries to access an address in memory that it has no business accessing. We won't talk much right now about about who/what gets to decide what addresses are accessible, but just know that we want to initialize pointers from one of the following

1. already existing addresses
2. malloc

You probably saw the first piece of code and were confused with the `&` sign. Personally, I think it's really easy to remember what the ampersand sign does...ampersand starts with 'a' and address starts with 'a', so remember that '&' will always return the address of the variable preceding it. I'm going to talk about this code and the printed values in class.

```
#include <stdio.h>

int main(){
  int a = 5;
  int* ptr = &a;
  printf("%p", &a);
  printf("%p", a);
  printf("%p", &ptr);
  printf("%p", ptr);
}
```
So, the ampersand sign is the first new operator in C that isn't available in Java, and it always give us the address of some variable. The second new operator in C is `*`, and it also is related to pointers. If the `&` gives us the address of some variable, an `*` will give us _the value at an address_.  Let's say for some reason that we didn't have access to `int a` and only had the pointer...then how would we ever have access to the value 5 again? We can access the value in the following way...
```
#include <stdio.h>

int main(){
  int a = 5;
  int* ptr = &a;
  printf("%d", *ptr);  //correctly dereferenced
  printf("%d", ptr);   //incorrect...converts the address that ptr points to into an int 
  
}
```
![output of following program](/images/dereference.png)

## Structs <a name = "struct"></a>
The closest thing we have in C to classes/objects are structs.









