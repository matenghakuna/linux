#Linux Virtual File System#

##VFS四个主要对象##
- 超级块对象，代表一个具体的已安装文件系统
- 索引节点对象，代表一个具体的文件
- 目录项对象，代表一个目录项，是路径的一个组成部分
- 文件对象，代表由进程打开的文件

**VFS将目录作为一个文件来处理，所以不存在目录对象。目录项代表的是路径中的一个组成部分，它可能包括一个普通文件。换句话说，目录项不同于目录，但目录却是另一种形式的文件**

###超级块对象###
超级块对象用于存储特定文件系统的信息，通常对应于存放在磁盘特定扇区中的文件系统超级块或文件系统控制块。对于并非基于磁盘的文件系统，它们会在使用现场创建超级块并将其保存到内存中。

创建、管理和撤销超级块对象的代码位于文件fs/super.c中。超级块对象通过`alloc_super()`函数创建并初始化。在文件系统安装时，文件系统会调用该函数以便从磁盘读取文件系统超级块，并将其内容填充到内存中的超级块对象中。

    struct super_block {
    	struct list_head	s_list;		/* Keep this first */
    	dev_t			s_dev;		/* search index; _not_ kdev_t */
    	unsigned long		s_blocksize;
    	unsigned char		s_blocksize_bits;
    	unsigned char		s_dirt;
    	loff_t			s_maxbytes;	/* Max file size */
    	struct file_system_type	*s_type;
    	const struct super_operations	*s_op;
    	const struct dquot_operations	*dq_op;
    	const struct quotactl_ops	*s_qcop;
    	const struct export_operations *s_export_op;
    	unsigned long		s_flags;
    	unsigned long		s_magic;
    	struct dentry		*s_root;
    	struct rw_semaphore	s_umount;
    	struct mutex		s_lock;
    	int			s_count;
    	int			s_need_sync;
    	atomic_t		s_active;
    #ifdef CONFIG_SECURITY
    	void    *s_security;
    #endif
    	struct xattr_handler	**s_xattr;
    
    	struct list_head	s_inodes;	/* all inodes */
    	struct hlist_head	s_anon;		/* anonymous dentries for (nfs) exporting */
    	struct list_head	s_files;
    	/* s_dentry_lru and s_nr_dentry_unused are protected by dcache_lock */
    	struct list_head	s_dentry_lru;	/* unused dentry lru */
    	int			s_nr_dentry_unused;	/* # of dentry on lru */
    
    	struct block_device	*s_bdev;
    	struct backing_dev_info *s_bdi;
    	struct mtd_info		*s_mtd;
    	struct list_head	s_instances;
    	struct quota_info	s_dquot;	/* Diskquota specific options */
    
    	int			s_frozen;
    	wait_queue_head_t	s_wait_unfrozen;
    
    	char s_id[32];				/* Informational name */
    
    	void 			*s_fs_info;	/* Filesystem private info */
    	fmode_t			s_mode;
    
    	/*
    	 * The next field is for VFS *only*. No filesystems have any business
    	 * even looking at it. You had been warned.
    	 */
    	struct mutex s_vfs_rename_mutex;	/* Kludge */
    
    	/* Granularity of c/m/atime in ns.
    	   Cannot be worse than a second */
    	u32		   s_time_gran;
    
    	/*
    	 * Filesystem subtype.  If non-empty the filesystem type field
    	 * in /proc/mounts will be "type.subtype"
    	 */
    	char *s_subtype;

    	/*
    	 * Saved mount options for lazy filesystems using
    	 * generic_show_options()
    	 */
    	char *s_options;
    };

####超级块操作####
超级块对象中最重要的一个域是s_op，它指向超级块的操作函数表。超级块操作函数表由`super_operations`结构体表示，定义在<linux/fs.h>中。

    struct super_operations {
       	struct inode *(*alloc_inode)(struct super_block *sb);
    	void (*destroy_inode)(struct inode *);
    
       	void (*dirty_inode) (struct inode *);
    	int (*write_inode) (struct inode *, int);
    	void (*drop_inode) (struct inode *);
    	void (*delete_inode) (struct inode *);
    	void (*put_super) (struct super_block *);
    	void (*write_super) (struct super_block *);
    	int (*sync_fs)(struct super_block *sb, int wait);
    	int (*freeze_fs) (struct super_block *);
    	int (*unfreeze_fs) (struct super_block *);
    	int (*statfs) (struct dentry *, struct kstatfs *);
    	int (*remount_fs) (struct super_block *, int *, char *);
    	void (*clear_inode) (struct inode *);
    	void (*umount_begin) (struct super_block *);
    
    	int (*show_options)(struct seq_file *, struct vfsmount *);
    	int (*show_stats)(struct seq_file *, struct vfsmount *);
    #ifdef CONFIG_QUOTA
    	ssize_t (*quota_read)(struct super_block *, int, char *, size_t, loff_t);
    	ssize_t (*quota_write)(struct super_block *, int, const char *, size_t, loff_t);
    #endif
    	int (*bdev_try_to_free_page)(struct super_block*, struct page*, gfp_t);
    };

###索引节点对象###
索引节点对象包含了内核在操作文件或目录时需要的全部信息。索引节点由inode结构体表示，它定义在文件<linux/fs.h>中。

    struct inode {
    	struct hlist_node	i_hash;
    	struct list_head	i_list;		/* backing dev IO list */
    	struct list_head	i_sb_list;
    	struct list_head	i_dentry;
    	unsigned long		i_ino;
    	atomic_t		i_count;
    	unsigned int		i_nlink;
    	uid_t			i_uid;
    	gid_t			i_gid;
    	dev_t			i_rdev;
    	u64			i_version;
    	loff_t			i_size;
    #ifdef __NEED_I_SIZE_ORDERED
    	seqcount_t		i_size_seqcount;
    #endif
    	struct timespec		i_atime;
    	struct timespec		i_mtime;
    	struct timespec		i_ctime;
    	blkcnt_t		i_blocks;
    	unsigned int		i_blkbits;
    	unsigned short          i_bytes;
    	umode_t			i_mode;
    	spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
    	struct mutex		i_mutex;
    	struct rw_semaphore	i_alloc_sem;
    	const struct inode_operations	*i_op;
    	const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
    	struct super_block	*i_sb;
    	struct file_lock	*i_flock;
    	struct address_space	*i_mapping;
    	struct address_space	i_data;
    #ifdef CONFIG_QUOTA
    	struct dquot		*i_dquot[MAXQUOTAS];
    #endif
    	struct list_head	i_devices;
    	union {
    		struct pipe_inode_info	*i_pipe;
    		struct block_device	*i_bdev;
    		struct cdev		*i_cdev;
    	};
    
    	__u32			i_generation;
    
    #ifdef CONFIG_FSNOTIFY
    	__u32			i_fsnotify_mask; /* all events this inode cares about */
    	struct hlist_head	i_fsnotify_mark_entries; /* fsnotify mark entries */
    #endif
    
    #ifdef CONFIG_INOTIFY
    	struct list_head	inotify_watches; /* watches on this inode */
    	struct mutex		inotify_mutex;	/* protects the watches list */
    #endif
    
    	unsigned long		i_state;
    	unsigned long		dirtied_when;	/* jiffies of first dirtying */
    
    	unsigned int		i_flags;
    
    	atomic_t		i_writecount;
    #ifdef CONFIG_SECURITY
    	void			*i_security;
    #endif
    #ifdef CONFIG_FS_POSIX_ACL
    	struct posix_acl	*i_acl;
    	struct posix_acl	*i_default_acl;
    #endif
    	void			*i_private; /* fs or device private pointer */
    };

**一个索引节点代表文件系统中的一个文件，它也可以是设备或管道这样的特殊文件。索引节点仅当文件被访问时，才在内存中创建。**

####索引节点操作####
    struct inode_operations {
    	int (*create) (struct inode *,struct dentry *,int, struct nameidata *);
    	struct dentry * (*lookup) (struct inode *,struct dentry *, struct nameidata *);
    	int (*link) (struct dentry *,struct inode *,struct dentry *);
    	int (*unlink) (struct inode *,struct dentry *);
    	int (*symlink) (struct inode *,struct dentry *,const char *);
    	int (*mkdir) (struct inode *,struct dentry *,int);
    	int (*rmdir) (struct inode *,struct dentry *);
    	int (*mknod) (struct inode *,struct dentry *,int,dev_t);
    	int (*rename) (struct inode *, struct dentry *,
    			struct inode *, struct dentry *);
    	int (*readlink) (struct dentry *, char __user *,int);
    	void * (*follow_link) (struct dentry *, struct nameidata *);
    	void (*put_link) (struct dentry *, struct nameidata *, void *);
    	void (*truncate) (struct inode *);
    	int (*permission) (struct inode *, int);
    	int (*check_acl)(struct inode *, int);
    	int (*setattr) (struct dentry *, struct iattr *);
    	int (*getattr) (struct vfsmount *mnt, struct dentry *, struct kstat *);
    	int (*setxattr) (struct dentry *, const char *,const void *,size_t,int);
    	ssize_t (*getxattr) (struct dentry *, const char *, void *, size_t);
    	ssize_t (*listxattr) (struct dentry *, char *, size_t);
    	int (*removexattr) (struct dentry *, const char *);
    	void (*truncate_range)(struct inode *, loff_t, loff_t);
    	long (*fallocate)(struct inode *inode, int mode, loff_t offset,
    			  loff_t len);
    	int (*fiemap)(struct inode *, struct fiemap_extent_info *, u64 start,
    		      u64 len);
    };

###目录项对象###

VFS把目录当作文件对待，所以在路径/bin/vi中，bin和vi都属于文件--bin是特殊的目录文件而vi是一个普通文件，路径中的每个组成部分都有一个索引节点对象表示。虽然他们可以统一由索引节点表示，但是VFS经常需要执行目录相关的操作，比如路径名查找等。为了方便查找操作，VFS引入了目录项的概念。每个dentry代表路径中的一个特定部分，在路径中（包括普通文件），每一个部分都是目录项对象。VFS在执行目录操作时，会现场创建目录项对象。

目录项对象由dentry结构体表示，定义在文件<linux/dcache.h>中。

    struct dentry {
    	atomic_t d_count;
    	unsigned int d_flags;		/* protected by d_lock */
    	spinlock_t d_lock;		/* per dentry lock */
    	int d_mounted;
    	struct inode *d_inode;		/* Where the name belongs to - NULL is
    					 * negative */
    	/*
    	 * The next three fields are touched by __d_lookup.  Place them here
    	 * so they all fit in a cache line.
    	 */
    	struct hlist_node d_hash;	/* lookup hash list */
    	struct dentry *d_parent;	/* parent directory */
    	struct qstr d_name;
    
    	struct list_head d_lru;		/* LRU list */
    	/*
    	 * d_child and d_rcu can share memory
    	 */
    	union {
    		struct list_head d_child;	/* child of parent list */
    	 	struct rcu_head d_rcu;
    	} d_u;
    	struct list_head d_subdirs;	/* our children */
    	struct list_head d_alias;	/* inode alias list */
    	unsigned long d_time;		/* used by d_revalidate */
    	const struct dentry_operations *d_op;
    	struct super_block *d_sb;	/* The root of the dentry tree */
    	void *d_fsdata;			/* fs-specific data */
    
    	unsigned char d_iname[DNAME_INLINE_LEN_MIN];	/* small names */
    };

与super_block,inode不同，dentry没有对应的磁盘数据结构，VFS根据字符串形式的路径名现场创建它。

####目录项状态####

目录项有三种状态：

- 被使用

dentry->d_inode指向一个有效的索引节点，dentry->d_count大于0。
表示正在被VFS使用。

- 未使用

dentry->d_inode指向一个有效的索引节点，dentry->d_count等于0。
表示被缓存。

- 负状态

dentry->d_inode等于NULL，表示索引节点被删除，或路径不再正确，但目录项仍然保留，以便快速解析以后的路径查询。

####目录项缓存####

遍历路径十分耗时，因此内核将目录项对象缓存在dcache中，分为三个部分：

- 被使用的目录项**链表**

通过inode->identry连接相关的inode,因为一个给定的inode可能有多个链接，所以就可能对应多个目录项对象。

- 最近被使用的**双向链表**

包含未被使用和负状态的目录项对象。

- **散列表和相应的散列函数**

由数组dentry_hashtable表示，其中每一个元素都是一个指向具有相同键值的目录项对象。散列值由散列函数`d_hash()`计算,是内核提供给文件系统的唯一一个hash函数。查找散列表要通过`d_lookup()`函数。

####目录项操作####

    struct dentry_operations {
    	int (*d_revalidate)(struct dentry *, struct nameidata *);
    	int (*d_hash) (struct dentry *, struct qstr *);
    	int (*d_compare) (struct dentry *, struct qstr *, struct qstr *);
    	int (*d_delete)(struct dentry *);
    	void (*d_release)(struct dentry *);
    	void (*d_iput)(struct dentry *, struct inode *);
    	char *(*d_dname)(struct dentry *, char *, int);
    };

###文件对象###

表示进程已打开的文件。文件对象 n : 目录项对象 1 : 索引对象 1

    struct file {
    	/*
    	 * fu_list becomes invalid after file_free is called and queued via
    	 * fu_rcuhead for RCU freeing
    	 */
    	union {
    		struct list_head	fu_list;
    		struct rcu_head 	fu_rcuhead;
    	} f_u;
    	struct path		f_path;
    #define f_dentry	f_path.dentry
    #define f_vfsmnt	f_path.mnt
    	const struct file_operations	*f_op;
    	spinlock_t		f_lock;  /* f_ep_links, f_flags, no IRQ */
    	atomic_long_t		f_count;
    	unsigned int 		f_flags;
    	fmode_t			f_mode;
    	loff_t			f_pos;
    	struct fown_struct	f_owner;
    	const struct cred	*f_cred;
    	struct file_ra_state	f_ra;
    
    	u64			f_version;
    #ifdef CONFIG_SECURITY
    	void			*f_security;
    #endif
    	/* needed for tty driver, and maybe others */
    	void			*private_data;
    
    #ifdef CONFIG_EPOLL
    	/* Used by fs/eventpoll.c to link all the hooks to this file */
    	struct list_head	f_ep_links;
    #endif /* #ifdef CONFIG_EPOLL */
    	struct address_space	*f_mapping;
    #ifdef CONFIG_DEBUG_WRITECOUNT
    	unsigned long f_mnt_write_state;
    #endif
    };

类似于目录项对象，文件对象没有对应的磁盘数据。文件对象通过f_dentry指向相关的目录项对象，目录项对象会指向相关的索引节点，索引节点会记录文件是否是脏的。

####文件操作####

    /*
     * NOTE:
     * read, write, poll, fsync, readv, writev, unlocked_ioctl and compat_ioctl
     * can be called without the big kernel lock held in all filesystems.
     */
    struct file_operations {
    	struct module *owner;
    	loff_t (*llseek) (struct file *, loff_t, int);
    	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    	ssize_t (*aio_read) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
    	ssize_t (*aio_write) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
    	int (*readdir) (struct file *, void *, filldir_t);
    	unsigned int (*poll) (struct file *, struct poll_table_struct *);
    	int (*ioctl) (struct inode *, struct file *, unsigned int, unsigned long);
    	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
    	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
    	int (*mmap) (struct file *, struct vm_area_struct *);
    	int (*open) (struct inode *, struct file *);
    	int (*flush) (struct file *, fl_owner_t id);
    	int (*release) (struct inode *, struct file *);
    	int (*fsync) (struct file *, struct dentry *, int datasync);
    	int (*aio_fsync) (struct kiocb *, int datasync);
    	int (*fasync) (int, struct file *, int);
    	int (*lock) (struct file *, int, struct file_lock *);
    	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
    	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
    	int (*check_flags)(int);
    	int (*flock) (struct file *, int, struct file_lock *);
    	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
    	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
    	int (*setlease)(struct file *, long, struct file_lock **);
    };


###和文件系统相关的数据结构###

    struct file_system_type {
    	const char *name;
    	int fs_flags;
    	int (*get_sb) (struct file_system_type *, int,
    		       const char *, void *, struct vfsmount *);
    	void (*kill_sb) (struct super_block *);
    	struct module *owner;
    	struct file_system_type * next;
    	struct list_head fs_supers;
    
    	struct lock_class_key s_lock_key;
    	struct lock_class_key s_umount_key;
    
    	struct lock_class_key i_lock_key;
    	struct lock_class_key i_mutex_key;
    	struct lock_class_key i_mutex_dir_key;
    	struct lock_class_key i_alloc_sem_key;
    };


`get_sb()`函数从磁盘读取超级块，并在文件系统被安装时，在内存中组装超级块对象。描述文件系统，不管有多少个实例安装到系统中，还是根本就没有安装到系统中，都只有一个file_system_type结构。

当文件系统被安装时，将有一个vfsmount结构体在安装点被创建，该结构体用来代表文件系统的实例。

    struct vfsmount {
    	struct list_head mnt_hash;
    	struct vfsmount *mnt_parent;	/* fs we are mounted on */
    	struct dentry *mnt_mountpoint;	/* dentry of mountpoint */
    	struct dentry *mnt_root;	/* root of the mounted tree */
    	struct super_block *mnt_sb;	/* pointer to superblock */
    	struct list_head mnt_mounts;	/* list of children, anchored here */
    	struct list_head mnt_child;	/* and going through their mnt_child */
    	int mnt_flags;
    	/* 4 bytes hole on 64bits arches */
    	const char *mnt_devname;	/* Name of device e.g. /dev/dsk/hda1 */
    	struct list_head mnt_list;
    	struct list_head mnt_expire;	/* link in fs-specific expiry list */
    	struct list_head mnt_share;	/* circular list of shared mounts */
    	struct list_head mnt_slave_list;/* list of slave mounts */
    	struct list_head mnt_slave;	/* slave list entry */
    	struct vfsmount *mnt_master;	/* slave is on master->mnt_slave_list */
    	struct mnt_namespace *mnt_ns;	/* containing namespace */
    	int mnt_id;			/* mount identifier */
    	int mnt_group_id;		/* peer group identifier */
    	/*
    	 * We put mnt_count & mnt_expiry_mark at the end of struct vfsmount
    	 * to let these frequently modified fields in a separate cache line
    	 * (so that reads of mnt_flags wont ping-pong on SMP machines)
    	 */
    	atomic_t mnt_count;
    	int mnt_expiry_mark;		/* true if marked for expiry */
    	int mnt_pinned;
    	int mnt_ghosts;
    #ifdef CONFIG_SMP
    	int *mnt_writers;
    #else
    	int mnt_writers;
    #endif
    };


###和进程相关的数据结构###

files_struct结构体由进程描述符中的files域指向。所有与单个进程相关的文件信息都包含在其中。

    /*
     * Open file table structure
     */
    struct files_struct {
      /*
       * read mostly part
       */
    	atomic_t count;
    	struct fdtable *fdt;
    	struct fdtable fdtab;
      /*
       * written part on a separate cache line in SMP
       */
    	spinlock_t file_lock ____cacheline_aligned_in_smp;
    	int next_fd;
    	struct embedded_fd_set close_on_exec_init;
    	struct embedded_fd_set open_fds_init;
    	struct file * fd_array[NR_OPEN_DEFAULT];
    };

fs_struct由进程描述符的fs域指向，它包含文件系统和进程相关的信息。

    struct fs_struct {
    	int users;
    	rwlock_t lock;
    	int umask;
    	int in_exec;
    	struct path root, pwd;
    };

mnt_namespace由进程描述符中的mnt_namespace域指向。使得进程在系统中看到唯一的文件系统层次结构。

    struct mnt_namespace {
    	atomic_t		count;
    	struct vfsmount *	root;
    	struct list_head	list;
    	wait_queue_head_t poll;
    	int event;
    };

list域是已安装的文件系统的双向链表，它包含的元素组成了全体命名空间。

对多数进程来说，它们的描述符都指向唯一的files_struct和fs_struct结构体。但是，对于使用克隆标志:CLONE_FILES或CLONE_FS创建的进程，会共享这两个结构体。所以多个进程描述符可能指向同一个files_struct或fs_struct。

mnt_namespace结构体的使用方法却和前两种结构完全不同。默认情况下，所有进程共享同样的命名空间，也就是它们都从相同的挂载表中看到同一个文件系统层次结构。只有在进行cloen()操作时使用CLONE_NEWS标志，才会给进程一个唯一的命名空间结构体的拷贝。大多数进程不提供这个标志，所有进程都继承其父进程的命名空间。因此大多数系统上只有一个命名空间。
