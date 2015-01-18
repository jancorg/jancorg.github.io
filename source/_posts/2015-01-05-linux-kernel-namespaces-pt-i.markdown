---
layout: post
title: "Linux Kernel Namespaces pt I"
date: 2015-01-05 20:03:42 +0100
comments: true
categories: [ devops, namespaces, kernel, linux ]
---



Namespaces are used to provide a process or a group of processes with the idea of being the only process or group of processes in the system.

They are a way to detach processes from a specific kernel layer assigning them to a new one. Or in other words, they are indirections layers for global resources.

Lets imagine this as an extension to classical `chroot()` syscall. When setting a new root calling chroot, kernel was isolating new branch from existing one, and thus creating a new namespace for the process.

> Namespaces now provide the basis for a complete lightweight virtualization system, in the form of containers.

Currently, linux support following namespaces

 NameSpace | from Kernel | Descrition 
 --------- | :---------: | ------------------------------------- 
 UTS       | 2.6.19      | domain and hostname 
 IPC       | 2.6.19      | queues, semaphores, and shared memmory 
 PID       | 2.6.19      | pid
 NS        | 2.4.19      | Filesystems 
 NET       | 2.4.24      | IP, routes, network devices... 
 USER      | 3.8         | Uid, Guid,... 

<br>
I let individual namespaces explanations as simple as this, or in other words, for another day.


## Namespaces API

In Linux kernel there are not distinction between process and threads implementions, threads are just light weight processes. Threads are also created by calling `clone()` but with different arguments (`CLONE_VM` mainly). From the kernel point of view, a process/thread is a task.

Namespaces can be nested. Limit for nesting namespaces is 32.

``` c nest_limit https://github.com/torvalds/linux/blob/87c31b39abcb6fb6bd7d111200c9627a594bf6a9/kernel/user_namespace.c
	if (parent_ns->level > 32)
		return -EUSERS;
```

### Structs
Next are most important structs parts used by namespaces API.

``` c task_struct https://github.com/torvalds/linux/blob/master/include/linux/sched.h 
struct task_struct {
[...]
/* process credentials */
	const struct cred __rcu *cred;	/* effective (overridable) subjective task
					 * credentials (COW) */
[...]
/* namespaces */
	struct nsproxy          *nsproxy;
```

``` c nsproxy https://github.com/torvalds/linux/blob/master/include/linux/nsproxy.h
struct nsproxy {
	atomic_t count;
	struct uts_namespace *uts_ns;
	struct ipc_namespace *ipc_ns;
	struct mnt_namespace *mnt_ns;
	struct pid_namespace *pid_ns_for_children;
	struct net           *net_ns;
};
```

``` c credentials https://github.com/torvalds/linux/blob/master/include/linux/cred.h
struct cred {
[...]
	struct user_namespace *user_ns; /* user_ns the caps and keyrings are relative to. */
[...]
```

``` c user_namespace https://github.com/torvalds/linux/blob/master/include/linux/user_namespace.h
struct user_namespace {
[...]
	struct user_namespace	*parent;
	struct ns_common	    ns;
[...]
};
```
<br>
### System calls

Namespaces can be created or modified by `clone()`, `unshare()` and `setns()` system calls. All of them are not POSIX system calls, so only available in linux.

`clone()` system call is more specific than `fork()` or `vfork()` system calls.  They are alike, because they are implemented by calling `do_fork()` function with different flags and args.

``` c fork_linux https://github.com/torvalds/linux/blob/master/kernel/fork.c
SYSCALL_DEFINE0(fork)
{
	return do_fork(SIGCHLD, 0, 0, NULL, NULL);
[...]
SYSCALL_DEFINE0(vfork)
{
	return do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, 0,
			0, NULL, NULL);
[...]
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp, int __user *, parent_tidptr, int __user *, child_tidptr, int, tls_val)
{
	return do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr);
``` 

`copy_process()` function in `do_fork()` calls `copy_namespaces()`. In case any namespace flags present, it just uses parent namespaces. (`fork()`, `vfork()`  behavior)

``` c 
if (likely(!(flags & (CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNET)))) {
		get_nsproxy(old_ns);
		return 0; }
	
static inline void get_nsproxy(struct nsproxy *ns)
	{
		atomic_inc(&ns->count);
	}
```
If any is present, it creates a new nsproxy struct and copies all namespaces.

``` c create_new_namespaces https://github.com/torvalds/linux/blob/603ba7e41bf5d405aba22294af5d075d8898176d/kernel/nsproxy.c
	new_nsp = create_nsproxy();
    new_nsp->mnt_ns = copy_mnt_ns(flags, tsk->nsproxy->mnt_ns, user_ns, new_fs);
	new_nsp->uts_ns = copy_utsname(flags, user_ns, tsk->nsproxy->uts_ns);
	new_nsp->ipc_ns = copy_ipcs(flags, user_ns, tsk->nsproxy->ipc_ns);
    new_nsp->pid_ns_for_children = copy_pid_ns(flags, user_ns, tsk->nsproxy->pid_ns_for_children);
	new_nsp->net_ns = copy_net_ns(flags, user_ns, tsk->nsproxy->net_ns);

	return new_nsp;
```

Child process is responsible for changing any namespace data.
``` c
sethostname(hostname, strlen(hostname);
```

`unshare()` system call allows a process to disassociate parts of its execution context that are currently being shared with other processes.

It allows to control shared execution context without creating a new process.

``` c 
SYSCALL_DEFINE1(unshare, unsigned long, unshare_flags)
{
	err = unshare_fs(unshare_flags, &new_fs);
	err = unshare_fd(unshare_flags, &new_fd);
	err = unshare_userns(unshare_flags, &new_cred);
	err = unshare_nsproxy_namespaces(unshare_flags, &new_nsproxy, new_cred, new_fs);
```

Flags like `CLONE_NEW*` flags have same effect as they have in `clone()`. All others ones that `unshare()` accepts has reverse effect. Since they were copied in above `create_new_namespaces` example.

When a task ends, all namespaces they belong to that does not have any other process attached are cleaned. This means, mounts unmounted, network interfaced destroyed, etc.

``` c put_nsproxy https://github.com/torvalds/linux/blob/master/include/linux/nsproxy.h
static inline void put_nsproxy(struct nsproxy *ns)
{
	if (atomic_dec_and_test(&ns->count)) {
		free_nsproxy(ns);
	}
}
```

Last API function to comment is `setns()` that simply allows a process to join an existing namespace.


``` c setns https://github.com/torvalds/linux/blob/603ba7e41bf5d405aba22294af5d075d8898176d/kernel/nsproxy.c
SYSCALL_DEFINE2(setns, int, fd, int, nstype)
{
	file = proc_ns_fget(fd);
	ns = get_proc_ns(file_inode(file));
	new_nsproxy = create_new_namespaces(0, tsk, current_user_ns(), tsk->fs);
	err = ns->ops->install(new_nsproxy, ns);

	switch_task_namespaces(tsk, new_nsproxy);
```

fd argument specifies the namespace to join. It is translated to a nscommon struct calling `get_proc_ns(file_inode(file))`. We will cover fds in next point.

``` c
define get_proc_ns(inode) ((struct ns_common *)(inode)->i_private)
```

## Namespaces in /proc

Per process namespaces can be found under `/proc/$pid/ns`. 

``` sh 
$ ls -l /proc/$$/ns
total 0
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan  5 21:12 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan  5 21:12 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan  5 21:12 net -> net:[4026531962]
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan  5 21:12 pid -> pid:[4026531836]
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan  5 21:12 user -> user:[4026531837]
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan  5 21:12 uts -> uts:[4026531838]
```

Each process namespace has an inode number, that corresponds with a namespace struct. If two tasks share same number, they belongs to same namespace. Inode for ns files in namespaces is not the same as `stat -c %i` shows. They are sym links.
For namespaces, like sockets or pipes inode number is shown in form `type:[inode]`.

``` sh
# for pid in 643 23681 32178 ; do readlink /proc/$pid/ns/mnt ; done
mnt:[4026531840]
mnt:[4026532430]
mnt:[4026532430]
# for pid in 643 23681 32178 ; do md5sum /proc/$pid/mounts ; done
8dddf7d919672a56849bb487840b94e0  /proc/643/mounts
70159c37e8c8f16c61ceaa047ad9528a  /proc/23681/mounts
70159c37e8c8f16c61ceaa047ad9528a  /proc/32178/mounts
# unshare -m /bin/bash 
# readlink /proc/$$/ns/mnt
mnt:[4026532493]
# for pid in  643 23681 32178; do stat -c %i /proc/$pid/ns/mnt ; done
1308868
1308731
1308686
```

Proc namespaces files are implemented through `proc_ns_operations` struct.

``` c proc_ns_operations https://github.com/torvalds/linux/blob/master/include/linux/proc_ns.h
struct proc_ns_operations {
	const char *name;
	int type;
	struct ns_common *(*get)(struct task_struct *task);
	void (*put)(struct ns_common *ns);
	int (*install)(struct nsproxy *nsproxy, struct ns_common *ns);
};

extern const struct proc_ns_operations netns_operations;
extern const struct proc_ns_operations utsns_operations;
extern const struct proc_ns_operations ipcns_operations;
extern const struct proc_ns_operations pidns_operations;
extern const struct proc_ns_operations userns_operations;
extern const struct proc_ns_operations mntns_operations;
```

<br>
There are availabe two commands that correspond to each system call (like most of the shell commands that are just called like any system call). `unshare` for `unshare()` and `nsenter` for `setns()`.
