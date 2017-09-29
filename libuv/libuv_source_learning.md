# `Hello World`
The simplest example
```c
#include <stdio.h>
#include <stdlib.h>
#include <uv.h>

int main() {
    uv_loop_t *loop = malloc(sizeof(uv_loop_t));
    uv_loop_init(loop);

    printf("Now quitting.\n");
    uv_run(loop, UV_RUN_DEFAULT);

    uv_loop_close(loop);
    free(loop);
    return 0;
}
```

We know that:
 - The loop struct the the key in whole program, and we need create (malloc) it by ourselves
 - use `uv_loop_init` to init it
 - TODO: uv_run?

# `uv_loop_t`
```c
typedef struct uv_loop_s uv_loop_t;

struct uv_loop_s {
  /* User data - use this for whatever. */
  void* data;
  /* Loop reference counting. */
  unsigned int active_handles;
  void* handle_queue[2];
  void* active_reqs[2];
  /* Internal flag to signal loop stop. */
  unsigned int stop_flag;
  UV_LOOP_PRIVATE_FIELDS
};

```

Most of fields are hidden in the macro `UV_LOOP_PRIVATE_FIELDS`. 
This macro contains different fields in different type of OSes:
```
uv-unix.h	192 #define UV_LOOP_PRIVATE_FIELDS

uv-win.h	312 #define UV_LOOP_PRIVATE_FIELDS
```
We only focus on unix platform here.

```c
#define UV_LOOP_PRIVATE_FIELDS                                                \
  unsigned long flags;                                                        \
  int backend_fd;                                                             \
  void* pending_queue[2];                                                     \
  void* watcher_queue[2];                                                     \
  uv__io_t** watchers;                                                        \
  unsigned int nwatchers;                                                     \
  unsigned int nfds;                                                          \
  void* wq[2];                                                                \
  uv_mutex_t wq_mutex;                                                        \
  uv_async_t wq_async;                                                        \
  uv_rwlock_t cloexec_lock;                                                   \
  uv_handle_t* closing_handles;                                               \
  void* process_handles[2];                                                   \
  void* prepare_handles[2];                                                   \
  void* check_handles[2];                                                     \
  void* idle_handles[2];                                                      \
  void* async_handles[2];                                                     \
  void (*async_unused)(void);  /* TODO(bnoordhuis) Remove in libuv v2. */     \
  uv__io_t async_io_watcher;                                                  \
  int async_wfd;                                                              \
  struct {                                                                    \
    void* min;                                                                \
    unsigned int nelts;                                                       \
  } timer_heap;                                                               \
  uint64_t timer_counter;                                                     \
  uint64_t time;                                                              \
  int signal_pipefd[2];                                                       \
  uv__io_t signal_io_watcher;                                                 \
  uv_signal_t child_watcher;                                                  \
  int emfile_fd;                                                              \
  UV_PLATFORM_LOOP_FIELDS                                                     \
```

And you may notice that there is another hidden fields macro at the bottom: `UV_PLATFORM_LOOP_FIELDS`. As its name:
```
H A D	uv-posix.h	25 #define UV_PLATFORM_LOOP_FIELDS \ macro 
H A D	uv-aix.h	25 #define UV_PLATFORM_LOOP_FIELDS \ macro 
H A D	uv-linux.h	25 #define UV_PLATFORM_LOOP_FIELDS \ macro 
H A D	uv-os390.h	27 #define UV_PLATFORM_LOOP_FIELDS \ macro 
H A D	uv-sunos.h	32 #define UV_PLATFORM_LOOP_FIELDS \ macro 
H A D	uv-darwin.h	37 #define UV_PLATFORM_LOOP_FIELDS \ macro 
H A D	uv-unix.h	106 # define UV_PLATFORM_LOOP_FIELDS /* empty */ macro 
```

We only focus on linux platform here :P
```c
#define UV_PLATFORM_LOOP_FIELDS                                               \
  uv__io_t inotify_read_watcher;                                              \
  void* inotify_watchers;                                                     \
  int inotify_fd;                                                             \
```

# init `uv_loop_t` : `uv_loop_init`
`uv_loop_init` function is a quiet "typical" init function: initializing all fields of a loop struct.

Some details:
 - 