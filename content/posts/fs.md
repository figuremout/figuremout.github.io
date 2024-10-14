---
title: "从文件描述符看开去：文件系统数据结构关联关系"
tags:
- linux
- kernel
date: 2022-10-10
draft: true
cover:
    image: /images/fs.drawio.svg
    caption: "Linux File System Data Structure"
    hiddenInSingle: false
---
# VFS inode & 磁盘 inode
VFS inode 是`struct inode`，它是VFS对各类不同的文件系统的共性做的抽象，它被包含在`struct ext2_inode_info`中，该结构体中还包含一些文件系统的特性。
磁盘 inode 是`struct ext2_inode`，和磁盘上的 inode 数据结构相同。

分配 inode 的时候，其实分配的是 `struct ext2_inode_info`，其中`vfs inode`字段是一个`struct inode`，然后对外给出去`struct inode`的地址即可。
VFS 层拿 `struct inode`的地址使用，底下文件系统强转类型后，取外层的`struct ext2_inode_info`地址使用。
```c
// fs/ext2/super.c
static struct inode *ext2_alloc_inode(struct super_block *sb)
{
	struct ext2_inode_info *ei;
	// 给ext2_inode_info分配内存
	ei = kmem_cache_alloc(ext2_inode_cachep, GFP_KERNEL);

	// 初始化ext2_inode_info
	
	// 返回ext2_inode_info内部的vfs_inode
	return &ei->vfs_inode;
}
```

通过`struct inode`取到外层的`struct ext2_inode_info`是通过`fs/ext2/ext2.h`的`EXT2_I`函数。
```c
// fs/ext2/ext2.h
static inline struct ext2_inode_info *EXT2_I(struct inode *inode)
{
	return container_of(inode, struct ext2_inode_info, vfs_inode);
}

// include/linux/kernel.h
/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:        the pointer to the member.
 * @type:       the type of the container struct this is embedded in.
 * @member:     the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member) ({                      \
        const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
        (type *)( (char *)__mptr - offsetof(type,member) );})
```
求`struct ext2_inode_info`的起始地址和其成员变量`vfs_inode`之间的差值得到一个 offset，反过来再用一个`struct inode`的起始地址减去这个 offset 就得到该`struct inode`所属的`struct ext2_inode_info`的起始地址。

## 参考
- [存储基础 — 文件描述符 fd 究竟是什么？](https://zhuanlan.zhihu.com/p/364617329)

# files_struct 中打开文件的静态+动态数组
`files_struct`有两个地方管理打开文件：

1. `struct file * fd_array[NR_OPEN_DEFAULT]`是一个静态数组，随着`files_struct`结构体分配出来的，`NR_OPEN_DEFAULT`在32位机上为32，在64位机上为64
2. `struct fdtable`也是个数组管理结构，只不过这个是一个动态数组，数组边界是用字段描述的

大部分进程只会打开少量的文件，所以静态数组就够了，这样就不用另外分配内存。如果超过了静态数组的阈值，那么就动态扩展。

## 静态数组
files_struct 的静态初始化：
```c
// fs/file.c
struct files_struct init_files = {
        .count          = ATOMIC_INIT(1),
        .fdt            = &init_files.fdtab,
        .fdtab          = {
                .max_fds        = NR_OPEN_DEFAULT,
                .fd             = &init_files.fd_array[0],
                .close_on_exec  = init_files.close_on_exec_init,
                .open_fds       = init_files.open_fds_init,
                .full_fds_bits  = init_files.full_fds_bits_init,
        },
        .file_lock      = __SPIN_LOCK_UNLOCKED(init_files.file_lock),
        .resize_wait    = __WAIT_QUEUE_HEAD_INITIALIZER(init_files.resize_wait),
};
```
可以看出此时`files_struct->fdt`指向自己的`files_struct->fdtab`，`files_struct->fdtab->fd`又指向`files_struct->fd_array`

## 动态数组
`struct files_struct`扩充使用内核源码中的`expand_files`来实现，expand_files会调用expand_fdtable：
```c
// fs/file.c
static int expand_fdtable(struct files_struct *files, unsigned int nr)
        __releases(files->file_lock)
        __acquires(files->file_lock)
{
        struct fdtable *new_fdt, *cur_fdt;

        spin_unlock(&files->file_lock);
        new_fdt = alloc_fdtable(nr); // 分配一个fdtable

        /* make sure all __fd_install() have seen resize_in_progress
         * or have finished their rcu_read_lock_sched() section.
         */
        if (atomic_read(&files->count) > 1)
                synchronize_sched();

        spin_lock(&files->file_lock);
        if (!new_fdt)
                return -ENOMEM;
        /*
         * extremely unlikely race - sysctl_nr_open decreased between the check in
         * caller and alloc_fdtable().  Cheaper to catch it here...
         */
        if (unlikely(new_fdt->max_fds <= nr)) {
                __free_fdtable(new_fdt);
                return -EMFILE;
        }
        cur_fdt = files_fdtable(files);
        BUG_ON(nr < cur_fdt->max_fds);
        copy_fdtable(new_fdt, cur_fdt); // 复制其中三个变量：fd, open_fds, close_on_exec
        rcu_assign_pointer(files->fdt, new_fdt); // 将新分配的fdtable赋值给files的fdt
        if (cur_fdt != &files->fdtab)
                call_rcu(&cur_fdt->rcu, free_fdtable_rcu);
        /* coupled with smp_rmb() in __fd_install() */
        smp_wmb();
        return 1;
}
```
当进程打开的文件数超过`NR_OPEN_DEFAULT`时，就要在对上面的已初始化的`struct files_struct`进行扩充。
当进行`struct files_struct`扩充时，会分配一个新的`struct fdtable`，另外还分配了满足扩充要求的`fd_array`。
分配并初始化新的`struct fdtable`变量后，原先指向`fdtab`的`struct files_struct`指针成员`fdt`，会调整为指向新分配的`struct fdtable`变量。这时，`struct files_struct`实例变量中就包含两个`struct fdtable`存储区：一个是其自身的，一个新分配的，用`fdt`指向。

## 参考
- [存储基础 — 文件描述符 fd 究竟是什么？](https://zhuanlan.zhihu.com/p/364617329)
- [struct files_struct和struct fdtable](https://blog.csdn.net/metersun/article/details/80513702)


# Open File Table 打开文件表
linux中进程级的open file table就是`struct files_struct`（内核源码中注释`Open file table structure`）

linux中系统打开文件表就是`struct file`的双向链表，用来记录系统所有已打开文件的信息。进程每打开一个文件就建立一个`struct file`，并把它加入到系统打开文件表中。
[[TODO]] 旧版本是这样，新版本呢？

# 页高速缓存
页高速缓存的核心数据结构是`address_space`对象，它是一个嵌入在页所有者的索引节点对象中的数据结构。
它管理一个文件在内存中缓存的所有页。

如果页高速缓存中的页的所有者是一个文件：
- `address_space`对象就嵌入在 VFS 索引节点对象的 i_data 字段中
- 索引节点的 i_mapping 字段总是指向索引节点的数据页所有者的`address_space`对象
- `address_space`对象的 host 字段指向其所有者的索引节点对象。

因此，如果页属于一个文件，则：
- 相应的`address_space`对象存放在 VFS 索引节点的 i_data 字段中
- 页的所有者就是文件的索引节点，因此索引节点的 i_mapping 字段指向自己的 i_data 字段
- `address_space`对象的 host 字段指向这个索引节点

`struct file`和`struct inode`中都包含有一个`struct address_space`的指针，分别为 f_mapping 和 i_mapping。
`struct file`是一个特定于进程的数据结构，而`struct inode`则是一个特定于文件的数据结构。每当进程打开一个文件时，都会将`file->f_mapping`设置到`inode->i_mapping`

## 参考
- 《深入理解 Linux 内核》

# 没有参考的参考
- https://zhuanlan.zhihu.com/p/107247475
- https://zhuanlan.zhihu.com/p/61123802
- https://www.zhihu.com/question/356860746

