# What we're doing today
+ [Virtual Memory](#vmem)


## Virtual Memory <a name = "vmem"></a>

The Virtual Memory Address space gives each process the illusion that it is the only entity utilizing memory, therefore each process believes it has access to all memory

Before, with bounds checking, two processes couldn't access the same addresses AND get their memory. With a VAS, you can have 2 processes that are both accessing addresses within what they think is their heap, and it will access their heaps memory, not the other processes heap's memory even though it's the same virtual address.

Virtual addresses are just the address that we use to do loads and stores. Proper management of VAS will ensure that faults will occur if we try to access memory that should be inaccessible for that moment
