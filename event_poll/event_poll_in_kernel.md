# Epoll Code Demo
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
        do_work(events[n].data.fd);
    }
}
```
