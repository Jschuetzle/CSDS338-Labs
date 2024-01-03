# What we're doing today
+ [C vs. Java](#c)
+ [Pointers and Addresses](#ptr)
+ [Structs](#struct)
+ [Memory Allocation](#malloc)
+ [Practice Exercise](#practice)

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
The closest thing we have in C to classes/objects are structs. You can think of structs as a _consolidation of information_, similar to classes in Java, but only containing the data fields (variables). This allows you to represent a datatype that is more complicated than the primitive types like int or char. For example, if you wanted to create a datatype that modeled a student, you can utilize a struct.

```
struct Student {
  char name[30];
  int age;
  int* grades;
};

int main(){
  struct Student jackson;
}
```

You would write this definition of the struct in the global context of the program, like how you would with a function. Notice, when you define a variable to be a struct, you must include the `struct` keyword, because it isn't simply a `Student`, rather it is a struct Student`. Just like in Java, if we want to assign values to struct members, we can do so using the **dot operator**.

```
int main(){
  struct Student jackson;

  strcpy(jackson.name, "Jackson Schuetzle");
  jackson.age = 20;
  jackson.grades = (int *) malloc(4*sizeof(int));
  jackson.grades[0] = 95;
  jackson.grades[1] = 92;
  jackson.grades[2] = 86;
  jackson.grades[3] = 79;

  printf("Student being printed:\n");
  printf("Name: %s\n", jackson.name);
  printf("Age: %d\n", jackson.age);
  printf("Grades: ");
  for(int i=0; i<4; i++){
    printf("%d ", jackson.grades[i]);
  }

return 0;
}
```

![output of the above program](/images/basicstruct-output.png)

The special thing about structs is accessing data members through a struct pointer. Just like how we created pointers to primitive types, we can create a pointer to a struct in the same way. Here's an example with a more simple type of struct, named `Point`, where we change the values of the data members through means of the pointer. 

```
struct Point {
  int x;
  int y;
}

int main() {
  struct Point p1;
  p1.x = 3;
  p1.y = -2;
  printf("p1: (%d, %d)\n", p1.x, p1.y);

  struct Point* p2 = &p1;
  p2->x = -8;
  p2->y = 10;
  printf("p1: (%d, %d)\n", p1.x, p1.y);

  return 0;
}
```

![change the members of a struct through use of a pointer](/images/struct-pointer.png)

If you looked closely, you'll notice that we didn't use the dot operator on the struct pointer, instead we used `->` which is called the **arrow operator**. This is because using the dot operator on a pointer wouldn't make any sense. The value of the struct pointer `p2` is just some address, namely the address of struct `p1`. When the dot operator is used, the compiler will access and read the variable's data as if it were a collection of values. If we used the dot operator on a pointer, the compiler will recognize that the value it's accessing/reading _doesn't represent a struct_, and an error will be thrown. If I inserted `.` instead of `->` in the above code, I get the following message after trying to compile.

![incorrectly using dot operator on a struct pointer](/images/dotoperator-on-pointer.png)

So, in order to access the data members through the struct pointer `p2`, we must **dereference the struct pointer**. And that's essentially what the arrow operator does.

```
p2->x = 10   //equivalent to (*p2).x = 10
```

What's the point of using a struct pointer though? Frequently, we will initialize a struct dynamically by using `malloc()` or its related functions, which all have a return type of pointer. So doing something like `struct Point p1 = malloc(sizeof(struct Point));` isn't allowed because the types don't match.

## Malloc and etc. <a name = "malloc"></a>
I've used the function `malloc()` a few times in this lecture without explaining what it actually does, so I want to devote the end of the lesson to exactly that. Essentially, `malloc()` will search the heap for a segment of memory of a specified size. Right now, we're not going to worry about _how_ it finds/determines the segment, rather just know that after we find a segment, it will return the first address of the chunk as a pointer. **So basically, `malloc()` returns a pointer to a new memory segment**. 

```
#include <stdio.h>
#include <stdlib.h>

void main(){
  int* ptr = malloc(10 * sizeof(int));    //creating memory to fit 10 ints
  printf("Address of ptr: %p\n", &ptr);
  printf("Address that ptr points to: %p\n", ptr);
}
```

![output of the above code](/images/simple-malloc.png)


Notice how the return values are different...the pointer will be stored in the stack because it is a local variable, and the address that `malloc()` returns will be from the heap (more on that [here](./Week3-SystemCalls.md#pms) if you're confused). The function only takes one parameter, and that is the size, in bytes, of the dynamically allocated memory chunk. We need to be careful though, because the parameter is absolute. If we wanted to allocate space for five integers, then the code would look like

```
int* ptr = malloc(5 * sizeof(int));  //correct
int* ptr2 = malloc(5);    //incorrect
```

Technically, the first line is just making the call `malloc(20)` since the size of an int is 4 bytes (32 bit), but this is bad practice because it is less readable and more difficult to understand if there's a bug. Opposed to Java, which uses bracket notation for arrays (int []), you'll oftentimes see `malloc()` being used to create an array. This seems confusing at first, but it actually reduces the amount of memory your program takes up. **Why store the entire array in memory when you could instead store just the address of the first value?**

The final important thing to learn about memory allocation is memory deallocation. Java has a garbage collector, so when a dynamically allocated segment is not of use anymore, e.g. if a pointer is popped off the stack, the garbage collector will automatically open up the segment of memory on the heap that was previously being used. In C, we don't have the luxury of a garbage collector, so the programmer has to manually free up the memory. We can do this with the library call `free()` which takes a pointer and deallocates the memory that it points to. 

```
struct Point {
  int x;
  int y;
}

int main() {
  struct Point p1;
  p1.x = 3;
  p1.y = -2;
  printf("p1: (%d, %d)\n", p1.x, p1.y);

  struct Point* p2 = &p1;
  p2->x = -8;
  p2->y = 10;
  printf("p1: (%d, %d)\n", p1.x, p1.y);

  free(p2);    //CHANGE MADE HERE
  return 0;
}
```
For this class, it won't make any practical difference whether you free up memory. The programs were dealing with are often short lived and don't consume massive amounts of memory. However, it is best practice to use `free()`. If you don't, a **memory leak** can occur. This is when previously allocated memory isn't deallocated, therefore _heap spaced is wasted until there is no heap space left_, and the program crashes. In a professional setting, it could be fatal to the application if you didn't free up memory.

## Practice <a name = "practice></a>
+ Using malloc, create two arrays: one `int` array and one `char` array. Size and initialize the arrays so that the former contains digits 0-9 and the latter contains the lowercase alphabet a-z, both in order. Print the associated number-letter pairs to the console while using all the letters, i.e. once the end of the digits array reaches the end, loop back to the beginning.

`a0 b1 c2 ... j9 k0 ... z5` 

If you encounter a heap buffer overflow, then you went past the bounds of the end of the array.

+ Create an implementation of a Linked List by creating a `struct Node`. Initialize the linked list so that the length is at least 3, traverse the linked list, and output the traversal. (Hint: it's easier to create use struct pointers)





