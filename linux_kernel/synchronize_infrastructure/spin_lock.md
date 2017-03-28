# Spin Lock
## Impl
`spinlock_t` -> `struct raw_spinlock rlock` -> `arch_spinlock_t raw_lock`
-> arch specifically:
### Impl in different arch:
#### x86
```c
typedef struct arch_spinlock {
	union {
		__ticketpair_t head_tail;
		struct __raw_tickets {
			__ticket_t head, tail;
		} tickets;
	};
} arch_spinlock_t;
```
#### arm
```c
typedef struct {
	union {
		u32 slock;
		struct __raw_tickets {
#ifdef __ARMEB__
			u16 next;
			u16 owner;
#else
			u16 owner;
			u16 next;
#endif
		} tickets;
	};
} arch_spinlock_t;
```
#### arm64
```c
typedef struct {
#ifdef __AARCH64EB__
	u16 next;
	u16 owner;
#else
	u16 owner;
	u16 next;
#endif
} __aligned(4) arch_spinlock_t;
```
### What's "tickets"? Why do they look like this?
#### It's kind of ***MCS lock***
> Here is a good article:  
https://www.quora.com/How-does-an-MCS-lock-work  
and its Chinese translation:  
http://www.ituring.com.cn/article/42394  

**I pasted it below:**

Lets first start with the "why" and the we'll get to the "how." As they do in their paper, lets start with a spin-lock based on the Test-and-set instruction. It looks like this (adapted from wikipedia):

```c
function Lock(lock) { 
     while (test_and_set(lock) == 1); 
}

function Unlock(lock){
     lock=0;
}
```

The Lock function works by relying on test_and_set to do the heavy lifting. Test_and_set will atomically read the lock (which is either a 1 or 0) and if it finds a zero, will set the lock to be 1. If the lock is held (set to 1), then it will simply return 1. Because this is an atomic instruction, no other processor can begin a test_and_set on this address while another test_and_set is still executing, allowing us to ensure mutual exclusion.

This works fine...so whats the problem? Well, there are two main problems here: (1) the test_and_set instruction must be executed atomically by the hardware, this makes it a very heavy-weight instruction and creates lots of coherency traffic over the interconnect, (2) we are not guaranteeing FIFO ordering amongst the processors competing for the lock. So a processor could be waiting for a quite a long time if it is unlucky.

So lets forget about the test-and-set lock...its too expensive and can't guarantee FIFO ordering. A better choice would be the Ticket lock, which is somewhat similar to  Lamport's bakery algorithm. The intuition behind the ticket lock is that each thread selects a ticket (like in a bakery) and waits for their ticket to be called. The code is below (adapted from the MCS paper):

```c
ticket_lock{
      int now_serving;
      int next_ticket;
};

function Lock(ticket_lock lock){
      //get your ticket atomically
      int my_ticket = fetch_and_increment(lock.next_ticket);
      while(my_ticket != now_serving){}
}

function UnLock(ticket_lock lock){
      lock.now_serving++;
}
```

Our Lock function gets the ticket using a single atomic operation (fetch_and_increment) which returns the old value and increments the next_ticket. By using an atomic instruction, we are guaranteed that two processors will never get the same ticket.

So what problems did we solve here? First, we only perform a single atomic operation during lock. This causes far less cache coherency traffic then the busy-waiting atomic operations of the test_and_set lock. Second, we now have FIFO ordering for lock acquisition. If our processor has been waiting for a while to get the lock, no johnny-come-lately can arrive and get it before us. 

BUT, there is still a problem. Each processor spins by reading the same variable, the now_serving member of the ticket_lock. Why is this a problem? Well, think of it from a cache coherency perspective. Each processor is spinning, constantly reading the now_serving variable. Finally, the processor holding the lock releases and increments now_serving. This invalidates the cache line for all other processors, causing each of them to go across the interconnect and retrieve the new value. If each request for the new cache line is serviced in serial by the directory holding the cache line, then the time to retrieve the new value is linear in the number of waiting processors.

We can improve scalability if the lock acquisition time is O(1) instead of O(n) (where n is the number of waiting processors). Thats where the MCS lock comes in. Each processor will spin on a local variable, not a variable shared with other processors. Here's the code:

```c
mcs_node{
      mcs_node next;
      int is_locked;
}

mcs_lock{ 
      mcs_node queue;
}

function Lock(mcs_lock lock, mcs_node my_node){
      my_node.next = NULL;
      mcs_node predecessor = fetch_and_store(lock.queue, my_node);
      //if its null, we now own the lock and can leave, else....
      if (predecessor != NULL){
          my_node.is_locked = true;
          //when the predecessor unlocks, it will give us the lock
          predecessor.next = my_node; 
          while(my_node.is_locked){}
}

function UnLock(mcs_lock lock, mcs_node my_node){
      //is there anyone to give the lock to?
      if (my_node.next == NULL){
            //need to make sure there wasn't a race here
            if (compare_and_swap(lock.queue, my_node, NULL)){
                 return;
            }
            else{
                 //someone has executed fetch_and_store but not set our next field
                 while(my_node.next==NULL){}
           } 
      }
     //if we made it here, there is someone waiting, so lets wake them up
     my_node.next.is_locked=false;
}
```

There is more code here, but its not too tricky. The basic intuition is that each processor is represented by a node in a queue. When we lock, if someone else holds the lock then we need to register ourselves by adding our node to the queue. We then busy wait on our node's is_locked field. When we unlock, if there is a processor waiting, we need to set that field to false to wake it up. 

More specifically, in Lock( ) we use fetch_and_store to atomically place our node at the end of the queue. Whatever was in lock.queue (the old tail of the queue) is now our "predecessor" and will get the lock directly before us. If lock.queue was NULL then no one owned the lock and we simply continue. If we need to wait, we then set our predecessor's next field so that it knows that we are next in line. Finally, we busy-wait for our predecessor to wake us up.

In UnLock( ), we first check if anyone is waiting for us to finish. The trick here is that even though mynode.next is NULL, another processor may have just performed the fetch_and_store in Lock( ) but has yet to set our next field. We use compare_and_swap to atomically make sure that our node is still then last in the queue. If it is, we are assured that no one is waiting and we can leave. Otherwise, we need to busy-wait on the other processor to set our next field. Finally, if someone is waiting, we wake them up by setting their is_locked field to false. 

So we've solved a few of the problems we mentioned before. We have minimal atomic instructions, and we've created a more scalable lock by busy-waiting on local variables that reside in their own cache line (mynode.is_locked).