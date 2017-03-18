# Epoll Code Demo
## The code
```c
struct epoll_event ev, events[MAX_EVENTS];
int efd = epoll_create(1024);

epoll_ctl(efd, EPOLL_CTL_ADD, listenfd, &ev);

while(1) {
    int n = epoll_wait(efd, events, MAX_EVENTS, -1);

    for (events[n].data.fd == listendf) {
        conn = accept(listenfd, (struct sockaddr *) addr, &addrlen);
        setnonblocking(conn);
        ev.evnets = EPOLLIN | EPOLLET;
        ev.data.fd = conn;
        epoll_ctl(efd, EPOLL_CTL_ADD, conn, &ev);
    }
    else {
        do_work(events[n].data.fd)
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
## `epoll_create`
