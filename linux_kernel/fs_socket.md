# File system & socket

## File system

### File Descriptor

A number, exposed to user space, represents a "pointer" or "reference" to one readable/writable "file object" in kernel.

### File Object

It is a abstract of "something" which can act like a normal "file".

The operations a file object should have:
```c
struct file_operations {
        struct module *owner;
        loff_t (*llseek) (struct file *, loff_t, int);
        ssize_t (*read) (struct file *, char *, size_t, loff_t *);
        ssize_t (*aio_read) (struct kiocb *, char *, size_t, loff_t);
        ssize_t (*write) (struct file *, const char *, size_t, loff_t *);
        ssize_t (*aio_write) (struct kiocb *, const char *, size_t, loff_t);
        int (*readdir) (struct file *, void *, filldir_t);
        unsigned int (*poll) (struct file *, struct poll_table_struct *);
        int (*ioctl) (struct inode *, struct file *, unsigned int, unsigned long);
        int (*mmap) (struct file *, struct vm_area_struct *);
        int (*open) (struct inode *, struct file *);
        int (*flush) (struct file *);
        int (*release) (struct inode *, struct file *);
        int (*fsync) (struct file *, struct dentry *, int);
        int (*aio_fsync) (struct kiocb *, int);
        int (*fasync) (int, struct file *, int);
        int (*lock) (struct file *, int, struct file_lock *);
        ssize_t (*readv) (struct file *, const struct iovec *,
                          unsigned long, loff_t *);
        ssize_t (*writev) (struct file *, const struct iovec *,
                           unsigned long, loff_t *);
        ssize_t (*sendfile) (struct file *, loff_t *, size_t,
                             read_actor_t, void *);
        ssize_t (*sendpage) (struct file *, struct page *, int,
                             size_t, loff_t *, int);
        unsigned long (*get_unmapped_area) (struct file *, unsigned long,
                                            unsigned long, unsigned long,
                                            unsigned long);
        int (*check_flags) (int flags);
        int (*dir_notify) (struct file *filp, unsigned long arg);
        int (*flock) (struct file *filp, int cmd, struct file_lock *fl);
};
```

## socket

### How socket co-work with file system

Alloc & init a file object when we are creating a socket in user apace.

```c
// `syscall3(socket ...) : sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));`  
            |  
            V  
// `static int sock_map_fd(struct socket *sock, int flags) : sock_alloc_file(sock, flags, NULL);`  
            |  
            V  
struct file *sock_alloc_file(struct socket *sock, int flags, const char *dname)
{
	struct qstr name = { .name = "" };
	struct path path;
	struct file *file;

	if (dname) {
		name.name = dname;
		name.len = strlen(name.name);
	} else if (sock->sk) {
		name.name = sock->sk->sk_prot_creator->name;
		name.len = strlen(name.name);
	}
	path.dentry = d_alloc_pseudo(sock_mnt->mnt_sb, &name);
	if (unlikely(!path.dentry))
		return ERR_PTR(-ENOMEM);
	path.mnt = mntget(sock_mnt);

	d_instantiate(path.dentry, SOCK_INODE(sock));

	file = alloc_file(&path, FMODE_READ | FMODE_WRITE,
		  &socket_file_ops);  // <------------ Create the file object for this socket, and set socket_file_ops to the file object's f_op field
	if (unlikely(IS_ERR(file))) {
		/* drop dentry, keep inode */
		ihold(path.dentry->d_inode);
		path_put(&path);
		return file;
	}

	sock->file = file;
	file->f_flags = O_RDWR | (flags & O_NONBLOCK);
	file->private_data = sock;
	return file;
}
EXPORT_SYMBOL(sock_alloc_file);
``` 

### socket file ops

And make this file object's file_operations point ot socket's version, e.g. socket_file_ops
```c
static const struct file_operations socket_file_ops = {
	.owner =	THIS_MODULE,
	.llseek =	no_llseek,
	.aio_read =	sock_aio_read,
	.aio_write =	sock_aio_write,
	.poll =		sock_poll,
	.unlocked_ioctl = sock_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = compat_sock_ioctl,
#endif
	.mmap =		sock_mmap,
	.release =	sock_close,
	.fasync =	sock_fasync,
	.sendpage =	sock_sendpage,
	.splice_write = generic_splice_sendpage,
	.splice_read =	sock_splice_read,
};
```

> Details about sock_poll are in event_poll/socket_poll.md