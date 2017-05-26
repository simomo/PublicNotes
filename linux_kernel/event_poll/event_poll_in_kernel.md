# Linux IO Models

| Mode         | Blocking                             | Non-blocking            |   |   |
|--------------|--------------------------------------|-------------------------|---|---|
| Synchronous  | read/write                           | read/write (O_NONBLOCK) |   |   |
| Asynchronous | I/O multiplexing (select/poll/epoll) | AIO                     |   |   |
|              |                                      |                         |   |   |


# Epoll Code Demo

## The code
```c
#define MAX_EVENTS 10
struct epoll_event ev, events[MAX_EVENTS];
int efd = epoll_create(1024);
...
epoll_ctl(efd,EPOLL_CTL_ADD,listenfd,&ev);
...
while(1) {
    int n =  epoll_wait(efd,events,MAX_EVENTS,-1);
    for(i = 0;i < n;i++){
        if(events[n].data.fd == listenfd) {
            conn = accept(listenfd,(struct sockaddr *)addr,&addrlen);
            setnonblocking(conn);
            ev.events = EPOLLIN| EPOLLET;
            ev.data.fd = conn;
            epoll_ctl(efd,EPOLL_CTL_ADD,conn,&ev);
        }
        else{
            do_work(events[n].data.fd);
        }
    }
}
```

## Explanation

### ***Create*** event poll fd

Allocate and init a event poll structure for following steps.

### ***Add*** the listening fd into event poll's "watching list"

This demo code implements a tcp server, we want to use epoll mechanism to handle
new incoming connections.

### Block on ***waiting*** for new incoming connections or incoming data

The code will block until a new connection arriving.

### ***Add*** the new incoming connection into event poll's "watching list"

We get a new incoming connection, then give it to epoll to handle incoming data from this connection.

### Three key functions

I emphasize three verb in above titles, they represent three functions: `epoll_create`, `epoll_ctl`, `epoll_wait`.

`epoll_create` will call syscall `epoll_create` or `epoll_create1`.

# Kernel level

## Data Structures

### `struct eventpoll`

The main data structure of the eventpool. Each call of epoll_create will create one this structure.

This structure will be stored inside the "private_data" field of the file structure.
```c
/*
 * This structure is stored inside the "private_data" member of the file
 * structure and represents the main data structure for the eventpoll
 * interface.
 */
struct eventpoll {
    /* Protect the access to this structure */
    spinlock_t lock;

    /*
     * This mutex is used to ensure that files are not removed
     * while epoll is using them. This is held during the event
     * collection loop, the file cleanup path, the epoll file exit
     * code and the ctl operations.
     */
    struct mutex mtx;

    /* Wait queue used by sys_epoll_wait() */
    wait_queue_head_t wq;

    /* Wait queue used by file->poll() */
    wait_queue_head_t poll_wait;

    /* List of ready file descriptors */
    struct list_head rdllist;

    /* RB tree root used to store monitored fd structs */
    struct rb_root rbr;

    /*
     * This is a single linked list that chains all the "struct epitem" that
     * happened while transferring ready events to userspace w/out
     * holding ->lock.
     */
    struct epitem *ovflist;

    /* wakeup_source used when ep_scan_ready_list is running */
    struct wakeup_source *ws;

    /* The user that created the eventpoll descriptor */
    struct user_struct *user;

    struct file *file;

    /* used to optimize loop detection check */
    int visited;
    struct list_head visited_list_link;
};
```
## `epoll_create` & `epoll_create1` syscall
### `epoll_create`
`epoll_create` syscall accept a parameter, size of ??????, but this parameter is useless actually, `epoll_create` will ignore this parameter, and call `epoll_create1`.

### `epoll_create1`
The real implementation of epoll_create, it will do:
#### allocate memory for a `struct eventpoll`
 ```c
 struct eventpoll *ep = NULL;
/*
 * Create the internal data structure ("struct eventpoll").
 */
error = ep_alloc(&ep);
if (error < 0)
    return error;
 ```
> C language tips:
> The codes above show us how to alloc memory for a pointer, pointing to a complex struct.

ep_alloc will:
 - init spin lock, `lock`
 - init mutex lock, `mtx`
 - init wait queue head, `wq` and `poll_wait`
 - init list head, `rdllist`
 - set `user` to current user
 - etc.

> TODO:
> [ ] sched/waitqueue.md
> [ ] synchronize_infrastructure/mutex.md
> [ ] synchronize_infrastructure/spin_lock.md

## `epoll_wait` syscall
```c
/*
 * Implement the event wait interface for the eventpoll file. It is the kernel
 * part of the user space epoll_wait(2).
 */
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
        int, maxevents, int, timeout)
{
    int error;
    struct fd f;
    struct eventpoll *ep;

    // Some param checks  - simomo
    .....
    .....

    /*
     * At this point it is safe to assume that the "private_data" contains
     * our own data structure.
     */
    ep = f.file->private_data;

    // Start do the real thing  - simomo
    /* Time to fish for events ... */
    error = ep_poll(ep, events, maxevents, timeout);
}
```

## `ep_poll` impl

```c
/**
 * ep_poll - Retrieves ready events, and delivers them to the caller supplied
 *           event buffer.
 *
 * @ep: Pointer to the eventpoll context.
 * @events: Pointer to the userspace buffer where the ready events should be
 *          stored.
 * @maxevents: Size (in terms of number of events) of the caller event buffer.
 * @timeout: Maximum timeout for the ready events fetch operation, in
 *           milliseconds. If the @timeout is zero, the function will not block,
 *           while if the @timeout is less than zero, the function will block
 *           until at least one event has been retrieved (or an error
 *           occurred).
 *
 * Returns: Returns the number of ready events which have been fetched, or an
 *          error code, in case of error.
 */
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
           int maxevents, long timeout)
{
    int res = 0, eavail, timed_out = 0;
    unsigned long flags;
    long slack = 0;
    wait_queue_t wait;
    ktime_t expires, *to = NULL;

    if (timeout > 0) {  //I: User sets the timeout value, and this function will block
        struct timespec end_time = ep_set_mstimeout(timeout);

        slack = select_estimate_accuracy(&end_time);
        to = &expires;
        *to = timespec_to_ktime(end_time);
    } else if (timeout == 0) {  //I: User don't want this function blocking, this function should return immediately
        /*
         * Avoid the unnecessary trip to the wait queue loop, if the
         * caller specified a non blocking operation.
         */
        timed_out = 1;
        spin_lock_irqsave(&ep->lock, flags);
        goto check_events;
    }

fetch_events:
    spin_lock_irqsave(&ep->lock, flags);

    if (!ep_events_available(ep)) {
        /*
         * We don't have any available event to return to the caller.
         * We need to sleep here, and we will be wake up by
         * ep_poll_callback() when events will become available.
         */
        init_waitqueue_entry(&wait, current);
        __add_wait_queue_exclusive(&ep->wq, &wait);

        for (;;) {
            /*
             * We don't want to sleep if the ep_poll_callback() sends us
             * a wakeup in between. That's why we set the task state
             * to TASK_INTERRUPTIBLE before doing the checks.
             */
            set_current_state(TASK_INTERRUPTIBLE);
            if (ep_events_available(ep) || timed_out)
                break;
            if (signal_pending(current)) {
                res = -EINTR;
                break;
            }

            spin_unlock_irqrestore(&ep->lock, flags);
            if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
                timed_out = 1;

            spin_lock_irqsave(&ep->lock, flags);
        }
        __remove_wait_queue(&ep->wq, &wait);

        set_current_state(TASK_RUNNING);
    }
check_events:
    /* Is it worth to try to dig for events ? */
    eavail = ep_events_available(ep);

    spin_unlock_irqrestore(&ep->lock, flags);

    /*
     * Try to transfer events to user space. In case we get 0 events and
     * there's still timeout left over, we go trying again in search of
     * more luck.
     */
    if (!res && eavail &&
        !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
        goto fetch_events;

    return res;
```



