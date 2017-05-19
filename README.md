# Project 3
# Introduction

This project involved the implementation of semaphores using a waiting queue  
which was used to demonstrate the synchronization of threads. Additionally,  
Thread Private Storage (TPS) was implemented to allow threads to have    
protected memory regions.  

# Phase 1: Semaphore Implementation

For this phase, several functions were implemented to allow a semaphore to  
perform its role in the synchronization of threads. Since each semaphore   
requires a waiting list and a count to keep track of the resources available   
for sharing between threads, we created a semaphore struct. This contained a   
queue to hold the threads in a FIFO order, as well as a count for the   
resources.  

## Semaphore Functions  
- __sem_create__ :  This function initialized the semaphore by dynamically   
allocating space using malloc, and initializing the count value taken in as a  
parameter to the function, as well as the queue by calling the queue_create()  
function.  If this queue was not initialized properly, or the count was a non  
positive integer, this suggests a failed creation of the semaphore and NULL   
is returned.  
- __sem_destroy__ : This function has the opposite goal as the sem_create, as  
it tries to free up the memory initialized by the semaphore using free().   
Before this can happen, we assure the semaphore we are trying to destroy   
exists, i.e. is not NULL, and that there are no threads currently in the   
queue.  
- __sem_down__ :  To model the taking of a resource by a thread, we decrement  
the count if there are resources available for taking (count > 0).  If this  
condition is not satisfied, we get the current thread by calling pthread_self  
and enqueue that to the semaphore's waiting list, where it is then blocked.  
The entire operation is enclosed in the enter and exit critical section  
functions to ensure the mutual exclusion property is held.  
- __sem_up__ : Opposite to the sem_down function, sem_up models the releasing  
of a resource, and thus an increment in the count if the queue is empty.   
Otherwise, we get the first blocked thread in the queue by dequeuing it and   
then unblocking it. Again, mutual exclusion during this operations is assured  
by calling enter_critical_section before the operations take place and  
calling exit_critical_section before exiting the sem_up function.  

# Phase 2: Implementation of Thread Private Storage  

For this phase, the goal was to implement the functions needed to create a  
TPS and provide for a private and protected memory page so that threads would   
not accidently modify information that other threads were using.  This part   
was divided into 3 phases.  

## Phase 2.1: Unprotected TPS with Naïve Cloning  
This part involved implementing the basic API of the TPS without concern for  
protection or copy on write cloning strategies.  
To begin, we created a tps struct that contained a pointer to the beginning  
of allocated memory, the thread whose storage would be of concern, and linked  
list features to keep track of the head of the TPS, as well as the next TPS.   
In addition, we keep track of the current tps with a global initialization.

## TPS functions  
- __tps_find__: This function takes in a thread id as a parameter, and   
iterates through the linked list until a match is found for that thread,   
returning this.    
- __tps_init__: Here, space is allocated for our global tps using malloc, and  
assigns the data of the struct, setting the thread by calling pthread_self   
and setting the head equal to our global current tps, and the next to NULL.  
The map is also initialized using the mmap function. Using the documentation,  
we assign its parameters. Given the size of a TPS area in the tps.h file, we   
set the size accordingly. Also since we know the TPS is concerned with   
protecting simultaneous reading and writing between threads, we choose to use  
those for the protect argument. Also we set the flags to private and  
anonymous. After this mapping is done, we assure it has not failed   
before returning.  
- __allocateTps__: This is a helper function for the initialization of a tps  
that will be used in tps_create and tps_clone.  
In this function, we first use our find_tps function to find the client   
thread whose tid is aquired using pthread_self(). If this TPS is NULL we   
allocate space in a similar manner done in tps_init function, with the   
addition of assigning this to be the next tps of our global current one in   
our linked list. We then set our current one to this temporary found one if   
the map has not already been allocated. This changed current tps then gets   
its map allocated using the mmap() function as done in tps_init.  
- __tps_create__: A call to allocateTps() is done to create the TPS area for   
the current thread, checking if memory allocation is successful.  
- __tps_destroy__: To destroy the tps, we first find the one associated with  
pthread_self and make sure it exists. We then deallocate the memory page   
using the unmap function munmap(), also found in the documentation. We use   
the tps's map and area as parameters and assure it successfully unmaps before  
returning.    
- __tps_read__: Here we again find_tps the client thread and check if the  
arguments are not trying to read more than the area available for our tps by  
comparing to our length + offset parameters.  We then copy the thread's map   
to the buffer parameter using memcpy() and return.     
- __tps_write__: This is implemented in the same way as tps_read, by also  
 assuring that its not trying to write out of bounds, except that memcpy()  
 copies content from the buffer to the thread's map.  
- __tps_clone__: In this phase, we again get the current tps   
using the find_tps and pthread_self() functions to be used as the new tps to   
clone in to. Its data is then initialized in the same way as in tps_create,   
by a call to our allocateTps() helper function. Then the tid from the   
parameter of the function is temporarily copied, whose map is then copied   
into our global current tps using memcpy().  

## Phase 2.2: Protected TPS &&  
##   Phase 2.3: Copy on Write Cloning
  These phases add protection to our previous TPS implementation by handling   
the possible segmentation faults that can occur without protection. The code  
provided in the project description for Phase 2.2 is completed.  
Phase 2.3 involves implementing a Copy-on-Write approach, delaying copy  
operations while shared memory pages are only read from by one of the threads  
that share it.  
## Changes from 2.1  
A page_struct is added which is used to hold the memory address space or map   
of the tps rather than the tps struct. The page also holds a shareCount to   
keep track of how many TPS's are currently sharing the same memory page. It   
is separated from our tps struct so multiple tps's can be able to point to  
the same page. In turn, our tps struct holds a member of the page.  
We then add the function __allocatePage()__ to initialize our global current   
tps's page, allocating memory using malloc and assigning the shareCount to 1,  
since that tps will initially be the only one using that memory page.  
Also, we have the initialization of the map here using the mmap() function    
rather than using it in our __allocateTps()__ helper function.  To account  
for this small change, we must also call allocatePage in our tps_create and    
tps_init  functions,  and assure that our allocation for the memory page was  
successful.        
- __segv_handler__: The skeleton code for this function was provided to us,  
and we just had to iterate through all of our TPS areas in our linked list to  
find if a match was found to the given * p_fault.  To do this, we set a   
temporary tps to be our the head of our current global tps. Using a while   
loop to see if this tps exists, we then checked if the map of its page was   
found which then causes match to be true (1) and breaks out of the loop.   
We then make our temporary tps the next one in our list and print out our  
error if the match was found.  
- __tps_init__: This function changes by adding the code given in the project  
description at the beginning to associate the function handler to the SIGSEGV  
and SIGBUS signals.  
- __givePermissions__: This is a helper function that will be called in our  
tps_read and tps_write functions, which adds protection during the copying of  
memory that is involved in these operations through use of memcpy(). Using  
the mprotect() function discovered in the assignment description, we activate  
accessibility for the reading and writing of memory for each of the   
parameters readFrom and writeTo using the protections PROT_READ and   
PROT_WRITE. During this period of accessible memory, memcpy() is called.   
Before exiting the function, the protections are then reset to none with   
the argument PROT_NONE.  
- __tps_write__: As noted above, this function gains protection through the  
call givePermissions. In addition, we call allocatePage() if our share count   
for our global current tps is more than 1: in other words if there are  
multiple TPS's sharing the same memory page, we create a new identical copy  
of that page to be privately used by the calling thread. This then decrements  
the shareCount by 1 since that thread now has its own copy and no longer has  
to share.  
- __tps_clone__: For phase 2.3, we maintain the initial implementation from  
2.1 with initializing a temporary copy of the tps to be cloned from, and a  
a temporary caller tps found using our find_tps and pthread_self() functions.  
We then call our allocateTps() function as before.  For this phase though, we  
then point our new tps page to the old tps page. Also, we increment the count  
for the shared resources of the old tps that had been cloned since there is   
now one more tps sharing that memory page.  

# Testing  
To test our implementations, we ran the files given such as sem_buffer.c  
sem_prime.c and sem_count.c to see if our semaphore in phase 1 worked  
properly.  Then for phase 2.1 we used the tps.c file to find that our tps  
also workedas expected.

# Sources  
* Piazza  
* <a href="https://www.gnu.org/software/libc/manual/html_mono/libc.html#Memory_002dmapped-I_002fO">Memory-mapped</a>  
* <a href="https://linux.die.net/man/2/mprotect">mprotect</a>  
* <a href="http://pubs.opengroup.org/onlinepubs/7908799/xsh/sigaction.html">sigaction</a>  
