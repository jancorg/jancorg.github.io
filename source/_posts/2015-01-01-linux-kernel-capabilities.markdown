---
layout: post
title: "Linux Kernel Capabilities"
date: 2015-01-01 20:57:16 +0100
comments: true
categories: [ devops, linux, kernel, capabilities ] 
---

Capabilities were created as an alternative to classical two level privilege system: root and user. They split a root acoount into their privileges. This way, linux kernel allows a process to perform certain root tipical tasks without giving a process full root privileges.

A process (or better, a thread as the smaller unit a capability can be assigned to) can be granted with a given capability. Each capability is independent from each other.

For instance, a user process with just CAP_NET_BIND_SERVICE capability can open ports bellow 1024, however it can not kill any process or use chroot.

All linux kernel capabilities [list](http://linux.die.net/man/7/capabilities) can be fond on man pages.

Instead of checking effective UID of user, modern kernels checks for capabilities, so they allow the privileged operation if capability bit is set in the effective set.

``` c init_process_privileges https://github.com/torvalds/linux/blob/9a3c4145af32125c5ee39c0272662b47307a8323/kernel/cred.c
struct cred init_cred = {
	.usage                  = ATOMIC_INIT(4),
#ifdef CONFIG_DEBUG_CREDENTIALS
	.subscribers            = ATOMIC_INIT(2),
	.magic                  = CRED_MAGIC,
#endif
	.uid                    = GLOBAL_ROOT_UID,
	.gid                    = GLOBAL_ROOT_GID,
	.suid                   = GLOBAL_ROOT_UID,
	.sgid                   = GLOBAL_ROOT_GID,
	.euid                   = GLOBAL_ROOT_UID,
	.egid                   = GLOBAL_ROOT_GID,
	.fsuid                  = GLOBAL_ROOT_UID,
	.fsgid                  = GLOBAL_ROOT_GID,
	.securebits             = SECUREBITS_DEFAULT,
	.cap_inheritable        = CAP_EMPTY_SET,
	.cap_permitted          = CAP_FULL_SET,
	.cap_effective          = CAP_FULL_SET,
	.cap_bset               = CAP_FULL_SET,
	.user                   = INIT_USER,
	.user_ns                = &init_user_ns,
	.group_info             = &init_groups,
};
} kernel_cap_t;
```linux_capabilities

There are four sets of capabilities:
effective: capabilities that a process is allowed.
permitted: capabilities that a process is permited. This allows to enable, disable or drop capabilities.
inheritable: capabilities that a process can give to another process called, for instance, by calling exec() system call.
bounding set: Limit from capabilities can not be grown. They just can be dropped. 

Credentials, therefore, are mostly a set of uids/guis, management flags, capabilities, namaspaces and cgroups.

As formerly happened with UID, GID and mode, capabilities are also part of VFS. They are called File Capabilities. They are store in f_cred struct.

``` c file_struct https://github.com/torvalds/linux/blob/603ba7e41bf5d405aba22294af5d075d8898176d/include/linux/fs.h
struct file {
	union {
		struct llist_node	fu_llist;
		struct rcu_head 	fu_rcuhead;
	} f_u;
	struct path                     f_path;
	struct inode                    *f_inode;	/* cached value */
	const struct file_operations	*f_op;

	/*
	 * Protects f_ep_links, f_flags.
	 * Must not be taken from IRQ context.
	 */
	spinlock_t              f_lock;
	atomic_long_t           f_count;
	unsigned int            f_flags;
	fmode_t                 f_mode;
	struct mutex            f_pos_lock;
	loff_t                  f_pos;
	struct fown_struct      f_owner;
	const struct cred       *f_cred;
	struct file_ra_state    f_ra;

	u64                     f_version;
#ifdef CONFIG_SECURITY
	void                    *f_security;
#endif
	/* needed for tty driver, and maybe others */
	void                    *private_data;

#ifdef CONFIG_EPOLL
	/* Used by fs/eventpoll.c to link all the hooks to this file */
	struct list_head        f_ep_links;
	struct list_head        f_tfile_llink;
#endif /* #ifdef CONFIG_EPOLL */
	struct address_space    *f_mapping;
} __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */
```

This way, we can, for example ping some host on the internet using CAP_NET_RAW capability.

Following is an example of setting a capability in command line interface.

``` sh setting_beep_capabilities
# setcap cap_dac_override,cap_sys_tty_config+ep /usr/bin/beep
``` 


