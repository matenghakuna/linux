#块IO层#

##块设备##

扇区是块设备的最小寻址单元，最常见的大小是512Byte,但其他大小的扇区也是常见的，比如CD-ROM的扇区都是2KB。
块是文件系统的抽象，它是文件系统的最小逻辑寻址单元(*只能基于块来访问文件系统*)。在Linux中，块的大小不能超过一个页面。

##缓冲区和缓冲区头##

缓冲区是磁盘块在内存中的表示。
每一个缓冲区都有一个对应的描述符，该描述符用`buffer_head`表示，称为缓冲区头，在`<linux/buffer_head.h>`中定义。缓冲区头的目的在于描述磁盘块和物理内存缓冲区之间的映射关系。

    /*
     * Historically, a buffer_head was used to map a single block
     * within a page, and of course as the unit of I/O through the
     * filesystem and block layers.  Nowadays the basic I/O unit
     * is the bio, and buffer_heads are used for extracting block
     * mappings (via a get_block_t call), for tracking state within
     * a page (via a page_mapping) and for wrapping bio submission
     * for backward compatibility reasons (e.g. submit_bh).
     */
    struct buffer_head {
    	unsigned long b_state;		/* buffer state bitmap (see above) */
    	struct buffer_head *b_this_page;/* circular list of page's buffers */
    	struct page *b_page;		/* the page this bh is mapped to */
    
    	sector_t b_blocknr;		/* start block number */
    	size_t b_size;			/* size of mapping */
    	char *b_data;			/* pointer to data within the page */
    
    	struct block_device *b_bdev;
    	bh_end_io_t *b_end_io;		/* I/O completion */
     	void *b_private;		/* reserved for b_end_io */
    	struct list_head b_assoc_buffers; /* associated with another mapping */
    	struct address_space *b_assoc_map;	/* mapping this buffer is
    						   associated with */
    	atomic_t b_count;		/* users using this buffer_head */
    };

##bio结构体##

bio代表现场正在执行的，以片断链表形式组织的块io操作，其中最重要的几个域是bi_io_vecs、bi_vcnt、bo_idx。

    /*
     * main unit of I/O for the block layer and lower layers (ie drivers and
     * stacking drivers)
     */
    struct bio {
    	sector_t		bi_sector;	/* device address in 512 byte
    						   sectors */
    	struct bio		*bi_next;	/* request queue link */
    	struct block_device	*bi_bdev;
    	unsigned long		bi_flags;	/* status, command, etc */
    	unsigned long		bi_rw;		/* bottom bits READ/WRITE,
    						 * top bits priority
    						 */
    
    	unsigned short		bi_vcnt;	/* how many bio_vec's */
    	unsigned short		bi_idx;		/* current index into bvl_vec */
    
    	/* Number of segments in this BIO after
    	 * physical address coalescing is performed.
    	 */
    	unsigned int		bi_phys_segments;
    
    	unsigned int		bi_size;	/* residual I/O count */
    
    	/*
    	 * To keep track of the max segment size, we account for the
    	 * sizes of the first and last mergeable segments in this bio.
    	 */
    	unsigned int		bi_seg_front_size;
    	unsigned int		bi_seg_back_size;
    
    	unsigned int		bi_max_vecs;	/* max bvl_vecs we can hold */
    
    	unsigned int		bi_comp_cpu;	/* completion CPU */
    
    	atomic_t		bi_cnt;		/* pin count */
    
    	struct bio_vec		*bi_io_vec;	/* the actual vec list */
    
    	bio_end_io_t		*bi_end_io;
    
    	void			*bi_private;
    #if defined(CONFIG_BLK_DEV_INTEGRITY)
    	struct bio_integrity_payload *bi_integrity;  /* data integrity */
    #endif
    
    	bio_destructor_t	*bi_destructor;	/* destructor */
    
    	/*
    	 * We can inline a number of vecs at the end of the bio, to avoid
    	 * double allocations for a small number of bio_vecs. This member
    	 * MUST obviously be kept at the very end of the bio.
    	 */
    	struct bio_vec		bi_inline_vecs[0];
    };

```c
    ------------
    |struct bio|
    ------------\
       |         \
       |bi_io_vec \ bi_idx
       |           \
    ----------------------------
    |bio_vec |bio_vec |bio_vec |  bio_vec结构体链表，总数为bio_vcnt
    ----------------------------
       |     
       |     
    ------
    |page|
    ------

    /*
     * was unsigned short, but we might as well be ready for > 64kB I/O pages
     */
    struct bio_vec {
    	struct page	*bv_page;
    	unsigned int	bv_len;
    	unsigned int	bv_offset;
    };
```

每个块I/O请求都通过一个bio来表示，每个请求包含[1,n]个块，这些块存储在bio_vec结构体数组中。这些结构体描述了每个片断在物理页中的实际位置。共有bi_vcnt个片断。当块I/O层开始执行请求，需要使用各个片断时，**bi_idx**域会不断更新，从而指向当前片断。**bi_idx更重要的作用在于分割bio结构体**，RAID驱动器可以把单个bio结构体分割到阵列中的各个硬盘上去。RAID设备驱动只需要拷贝这个bio结构体，再把bi_idx设置为每个独立硬盘操作时需要的位置就可以了。

##请求队列##
块设备将他们挂起的块I/O请求保存在请求队列中，由`request_queue`表示，包含一个双向请求链表及相关控制信息。通过内核中像文件系统这样高层的代码将请求加入到队列中。请求队列只要不为空，队列对应的块设备驱动程序就会从队列头取请求，然后送到对应的块设备。

请求队列中的每一项都是一个单独的请求， 由request表示。一个请求可能要操作多个连续的磁盘块，所以每个请求可以由多个bio组成。磁盘上的块必须连续，但在内存中这些块并不一定要连续。

##IO调度程序##
IO调度程序通过两种方法减少磁盘寻址时间：合并与排序。

###Linus电梯###
###最终期限I/O调度程序###
###预测I/O调度程序###
###完全公正的排序I/O调度程序###
###空操作的I/O调度程序###
