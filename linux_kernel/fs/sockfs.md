# sockfs

## `vfs`

`vfs` provides a abstract layer of many kinds of things (including `socket`). 
User space app's read/write operation will be handled by `vfs` as a file, then the `vfs` forward those operations to `socket` implementation.
To achieve this, the `socket` must be "involved" into the `vfs` system as a filesystem(its name is "sockfs") before we can use it.

## sockfs init
```c
static int __init sock_init(void)
{
	int err;
	/*
	 *      Initialize the network sysctl infrastructure.
	 */
	err = net_sysctl_init();
	if (err)
		goto out;

	/*
	 *      Initialize skbuff SLAB cache
	 */
	skb_init();

	/*
	 *      Initialize the protocols module.
	 */

	init_inodecache();

	err = register_filesystem(&sock_fs_type);
	if (err)
		goto out_fs;
	sock_mnt = kern_mount(&sock_fs_type);
	if (IS_ERR(sock_mnt)) {
		err = PTR_ERR(sock_mnt);
		goto out_mount;
	}

	/* The real protocol initialization is performed in later initcalls.
	 */

#ifdef CONFIG_NETFILTER
	err = netfilter_init();
	if (err)
		goto out;
#endif

	ptp_classifier_init();

out:
	return err;

out_mount:
	unregister_filesystem(&sock_fs_type);
out_fs:
	goto out;
}

core_initcall(sock_init);	/* early initcall */
```


## Operations

```c
// how to define a file system
// The basic info of a file system: its name, how to create & destroy
static struct file_system_type sock_fs_type = {
	.name =		"sockfs",  // name of this file system
	.mount =	sockfs_mount,  // how to mount this fs into vfs
	.kill_sb =	kill_anon_super,  // how to destroy a fs
};
```