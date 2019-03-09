# 文件系统分析

## 文件系统初始化
在系统初始化时 `init进程` 会调用 `setup()` 函数，而 `setup()` 函数会调用内核函数 `sys_setup()` 函数。`sys_setup()` 函数定义在 `drivers/block/genhd.c` 文件中，源码如下：
```c
asmlinkage int sys_setup(void * BIOS)
{
    ...
    mount_root();
    return (0);
}
```
`sys_setup()` 函数会调用 `mount_root()` 函数来读取根目录文件系统，源码如下：
```c
struct file_system_type file_systems[] = {
#ifdef CONFIG_MINIX_FS
	{minix_read_super,	"minix",	1},
#endif
#ifdef CONFIG_EXT_FS
	{ext_read_super,	"ext",		1},
#endif
#ifdef CONFIG_EXT2_FS
	{ext2_read_super,	"ext2",		1},
#endif
#ifdef CONFIG_XIA_FS
	{xiafs_read_super,	"xiafs",	1},
#endif
#ifdef CONFIG_MSDOS_FS
	{msdos_read_super,	"msdos",	1},
#endif
#ifdef CONFIG_PROC_FS
	{proc_read_super,	"proc",		0},
#endif
#ifdef CONFIG_NFS_FS
	{nfs_read_super,	"nfs",		0},
#endif
#ifdef CONFIG_ISO9660_FS
	{isofs_read_super,	"iso9660",	1},
#endif
#ifdef CONFIG_SYSV_FS
	{sysv_read_super,	"xenix",	1},
	{sysv_read_super,	"sysv",		1},
	{sysv_read_super,	"coherent",	1},
#endif
#ifdef CONFIG_HPFS_FS
	{hpfs_read_super,	"hpfs",		1},
#endif
	{NULL,			NULL,		0}
};

struct file_system_type *get_fs_type(char *name)
{
    int a;
    
    if (!name)
        return &file_systems[0];
    for(a = 0 ; file_systems[a].read_super ; a++)
        if (!strcmp(name,file_systems[a].name))
            return(&file_systems[a]);
    return NULL;
}

static struct super_block * read_super(dev_t dev,
        char *name,int flags, void *data, int silent)
{
    struct super_block * s;
    struct file_system_type *type;
    ...
    if (!(type = get_fs_type(name))) {
        return NULL;
    }
    ...
    s->s_dev = dev;
    s->s_flags = flags;
    if (!type->read_super(s,data, silent)) {
        s->s_dev = 0;
        return NULL;
    }
    s->s_dev = dev;
    s->s_covered = NULL;
    s->s_rd_only = 0;
    s->s_dirt = 0;
    return s;
}

void mount_root(void)
{
	struct file_system_type * fs_type;
	struct super_block * sb;
	struct inode * inode;
	...
	for (fs_type = file_systems; fs_type->read_super; fs_type++) {
		if (!fs_type->requires_dev)
			continue;
		sb = read_super(ROOT_DEV,fs_type->name,root_mountflags,NULL,1);
		if (sb) {
			inode = sb->s_mounted;
			inode->i_count += 3 ;	/* NOTE! it is logically used 4 times, not 1 */
			sb->s_covered = inode;
			sb->s_flags = root_mountflags;
			current->pwd = inode;
			current->root = inode;
			return;
		}
	}
	panic("VFS: Unable to mount root");
}
```
`mount_root()` 函数首先会遍历系统支持的所有文件系统，然后调用他们的 `read_super()` （譬如minix文件系统会调用minix_read_super()）方法来尝试读取 `根目录` 的文件系统超级块，如果 `read_super()` 方法返回一个超级块对象，表示读取成功，根目录使用此文件系统格式化的。然后把当前进程的 `pwd` (当前工作目录) 和 `root` (根目录) 字段设置为根目录的 `inode` 节点。