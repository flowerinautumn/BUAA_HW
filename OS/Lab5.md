# Lab5

### 1. 思考题

#### Thinking 5.1

![image-20250517160601638](Lab5.assets/image-20250517160601638.png)

- 当外部设备产生中断信号或者更新数据时，此时Cache中之前旧的数据可能刚完成缓存，那么完成缓存的这一部分无法完成更新，则会发生错误的行为。
- 对于串口设备来说，读写频繁，信号多，在相同的时间内发生错误的概论远高于IDE磁盘。

#### Thinking 5.2

![image-20250518095858207](Lab5.assets/image-20250518095858207.png)

- 文件控制块结构体中，有这样的一个属性 `f_pad` ，大小为 `BY2FILE `即 256B ，则代表一个文件控制块的大小为 256B ，那么一个磁盘块 4KB 最多存储 4KB/256B=16 个文件控制块。
- 一个目录 = 一个磁盘块全部用来存目录项，又一个目录项32位=4B32位=4*B*；则一个磁盘块 4KB ，一个磁盘块中有 4KB/4B=1024 个目录项；又一个目录项指向一个存有文件内容的 4KB 磁盘块，一个磁盘块中有16个文件控制块，则一个目录下最多有1024∗16=16*K* 个文件。
- 一个文件控制块，描述一个文件；一个文件有 **直接指针 + 间接指针** 共1024个 ，每个指针指向一个磁盘块，存储着该文件的一部分文件数据。文件系统支持的单个文件最大，则表示1024个指针全部有效，一共指向了1024个磁盘块存着文件数据，又一个磁盘块 4KB，则单个文件最大 1024∗4KB=4MB。

#### Thinking 5.3

![image-20250518100615944](Lab5.assets/image-20250518100615944.png)

- 由于将 `DISKMAP ~ DISKMAP+DISKMAX` 这一段虚存地址空间作为**缓冲区**， `DISKMAX` = 0x400000000，因此最多处理 1GB。

#### Thinking 5.4

![image-20250518100919831](Lab5.assets/image-20250518100919831.png)

- `user/include/fd.h` 这两个宏用来找 `fd` 对应的**文件信息页面**和**文件缓存区地址**

  ```c
  #define INDEX2FD(i) (FDTABLE + (i)*BY2PG)
  #define INDEX2DATA(i) (FILEBASE + (i)*PDMAP)
  ```

#### Thinking 5.5

![image-20250518101230959](Lab5.assets/image-20250518101230959.png)

```c
#include "lib.h"

static char *msg = "This is the NEW message of the day!\n\n";
static char *diff_msg = "This is a different massage of the day!\n\n";

void umain()
{
        int r;
        int fdnum;
        char buf[512];
        int n;

        if ((r = open("/newmotd", O_RDWR)) < 0) {
            user_panic("open /newmotd: %d", r);
        }
        fdnum = r;
        writef("open is good\n");

        if ((n = read(fdnum, buf, 511)) < 0) {
            user_panic("read /newmotd: %d", r);
        }
        if (strcmp(buf, diff_msg) != 0) {
            user_panic("read returned wrong data");
        }
        writef("read is good\n");

        int id;

        if ((id = fork()) == 0) {
            if ((n = read(fdnum, buf, 511)) < 0) {
                user_panic("child read /newmotd: %d", r);
            }
            if (strcmp(buf, diff_msg) != 0) {
                user_panic("child read returned wrong data");
            }
            writef("child read is good && child_fd == %d\n",r);
            struct Fd *fdd;
            fd_lookup(r,&fdd);
            writef("child_fd's offset == %d\n",fdd->fd_offset);
        }
        else {
            if((n = read(fdnum, buf, 511)) < 0) {
                user_panic("father read /newmotd: %d", r);
            }
            if (strcmp(buf, diff_msg) != 0) {
                user_panic("father read returned wrong data");
            }
            writef("father read is good && father_fd == %d\n",r);
            struct Fd *fdd;
            fd_lookup(r,&fdd);
            writef("father_fd's offset == %d\n",fdd->fd_offset);
        }
}
```

- 运行结果：

```c
main.c: main is start ...
init.c: mips_init() is called
Physical memory: 65536K available, base = 65536K, extended = 0K
to memory 80401000 for struct page directory.
to memory 80431000 for struct Pages.
pmap.c:  mips vm init success
pageout:        @@@___0x7f3fe000___@@@  ins a page 
pageout:        @@@___0x40d000___@@@  ins a page 
FS is running
FS can do I/O
pageout:        @@@___0x7f3fe000___@@@  ins a page 
pageout:        @@@___0x407000___@@@  ins a page 
superblock is good
diskno: 0
diskno: 0
read_bitmap is good
diskno: 0
alloc_block is good
file_open is good
file_get_block is good
file_flush is good
file_truncate is good
diskno: 0
file rewrite is good
serve_open 00000400 ffff000 0x2
open is good
read is good
father read is good && father_fd == 0
father_fd's offset == 41
[00000400] destroying 00000400
[00000400] free env 00000400
i am killed ... 
child read is good && child_fd == 0
child_fd's offset == 41
[00001402] destroying 00001402
[00001402] free env 00001402
i am killed ... 
```

#### Thinking 5.6

![image-20250518101417220](Lab5.assets/image-20250518101417220.png)

- `struct Flie`：
  - `f_name[MAXNAMELEN]`：文件名，最大128字节
  - `f_size`：文件大小（字节）
  - `f_type`：文件类型（普通文件FTYPE_REG或目录FTYPE_DIR）
  - `f_direct[NDIRECT]`：直接指针数组，记录文件数据块在磁盘上的物理位置
  - `f_indirect`：间接指针，指向存储更多块指针的磁盘块
  - `f_dir`：指向所属目录的指针
  - `f_pad[]`：填充字段，确保结构体大小对齐磁盘块
- `struct Fd`：
  - `fd_dev_id`：设备ID，决定使用哪个驱动/服务函数
  - `fd_offset`：当前读写偏移量
  - `fd_omode`：打开模式（只读/只写/读写）
- `struct Filefd`：
  - `f_fd`：基本的文件描述符
  - `f_fileid`：文件ID，用于在打开文件表中查找
  - `f_file`：对应的文件控制块

#### Thinking 5.7

![image-20250518102310537](Lab5.assets/image-20250518102310537.png)

- `ENV_CREATE(user_env)` 和 `ENV_CREATE(fs_serv)` 都是异步消息，由 `init()` 发出创建消息后， `init()` 函数即可返回执行后续步骤，由 `fs` 和 `user` 线程执行自己的初始化工作。
- `fs` 线程初始化 `serv_init()` 和 `fs_init()` 完成后，进入 `serv()` 函数，被 `ipc_receive()` 阻塞为`ENV_NOT_RUNNABLE` ，直到收到 `user` 线程的 `ipc_send(fsreq)` 被唤醒。
- `user` 线程向 `fs` 线程 `ipc_send(fsreq)` 发送请求为同步消息，发送后自身进入阻塞`ENV_NOT_RUNNABLE` 等待被唤醒的fs线程服务结束时 `ipc_send(dst_va)` ,用户线程接收到数据后继续运行，此后 `fs` 线程进入阻塞，等待下次被用户唤醒。

### 2. 难点分析

#### 2.1 IDE磁盘驱动

##### 2.1.1 内存映射I/O

实现`kern/syscall_all.c`中的两个函数：

- `sys_write_dev`：将起始虚拟地址为`va`，长度为`len`字节的一段数据写到起始物理地址为`pa`，长度为`len`字节的物理空间上。

```c
int sys_write_dev(u_int va, u_int pa, u_int len) { // 该函数将用户空间虚拟地址va写入到设备内存，成功返回则0
	if (is_illegal_va_range(va, len) || is_illegal_dev_range(pa, len) || va % len != 0) { // 虚拟地址合法性检验+设备地址合法性检验+虚拟地址是否按len对齐
		return -E_INVAL;
	}
	if (len == 4) {
		iowrite32(*(uint32_t *)va, pa); // 4字节写入
	} else if (len == 2) {
		iowrite16(*(uint16_t *)va, pa); // 2字节写入
	} else if (len == 1) {
		iowrite8(*(uint8_t *)va, pa); // 1字节写入
	} else {
		return -E_INVAL;
	}
	return 0; // 操作成功返回0
}
```

- `sys_read_dev`：将起始物理地址为`pa`，长度为`len`字节的一段数据写到起始虚拟地址为`va`，长度为`len`字节的物理空间上。

```c
int sys_read_dev(u_int va, u_int pa, u_int len) { // 该函数实现从设备空间物理地址pa读取数据到用户空间
	if (is_illegal_va_range(va, len) || is_illegal_dev_range(pa, len) || va % len != 0) { // 虚拟地址合法性检验 + 设备地址合法性检验 + 地址对齐要求
		return -E_INVAL;
	}
	if (len == 4) {
		*(uint32_t *)va = ioread32(pa); // 4字节读取
	} else if (len == 2) {
		*(uint16_t *)va = ioread16(pa); // 2字节读取
	} else if (len == 1) {
		*(uint8_t *)va = ioread8(pa); // 1字节读取
	} else {
		return -E_INVAL;
	}
	return 0; // 操作成功返回0
}
```

##### 2.1.2 IDE 磁盘

- IDE磁盘操作通过PIIX4 I/O关键寄存器完成。

![image-20250519102926366](Lab5.assets/image-20250519102926366.png)

![image-20250519103111398](Lab5.assets/image-20250519103111398.png)

##### 2.1.3 驱动程序编写

系统调用`sys_write_dev`与`sys_read_dev`提供了通用的设备控制接口。而针对IDE磁盘这一特定设备，我们还需要为其编写用户态的控制接口，也即设备驱动程序。设备驱动程序为操作系统提供了标准化的设备控制接口，包括`ide_read`和`ide_write`函数。

实现`fs/ide.c`中的两个函数实现用户态下对磁盘的读写操作。

- `ide_read`：

  流程如下：

  - 使用  wait_ide_ready 等待 IDE 设备就绪。 
  - 向物理地址  MALTA_IDE_NSECT 处写入单字节 1，表示设置操作扇区数目为 1。 
  - 分四次设置扇区号，并在最后一次一同设置扇区寻址模式和磁盘编号。 
  - 向物理地址  MALTA_IDE_STATUS 处写入单字节值 MALTA_IDE_CMD_PIO_READ ，设置 IDE 设备为读状态。 使用  wait_ide_ready 等待 IDE 设备就绪。
  -  从物理地址 MALTA_IDE_DATA 处以四字节为单位读取数据。

```c
void ide_read(u_int diskno, u_int secno, void *dst, u_int nsecs) { // 实现从IDE磁盘指定扇区读取数据到内存中，参数分别为disk_number,sector_number,destination,number_of_seu_int diskno, u_int secno, void *dst, u_int nsecsu_int diskno, u_int secno, void *dst, u_int nsecsctors
	uint8_t temp; // 临时存储寄存器读写的数据
	u_int offset = 0, max = nsecs + secno; // 目标内存偏移量和终止扇区号
	panic_on(diskno >= 2); // 确保磁盘编号合法

	while (secno < max) { // 从secno开始，循环读取nsecs个扇区
		temp = wait_ide_ready(); // 等待IDE设备就绪
		temp = 1;
		panic_on(syscall_write_dev(&temp, MALTA_IDE_NSECT, 1)); // 设置操作扇区数目
		// 接下来将 28 位 LBA 地址分三次写入寄存器（LBAL、LBAM、LBAH）
		temp = secno & 0xff; // 设置操作扇区号的 [7:0] 位
		panic_on(syscall_write_dev(&temp, MALTA_IDE_LBAL, 1));

		temp = (secno >> 8) & 0xff; // 设置操作扇区号的 [15:8] 位
		panic_on(syscall_write_dev(&temp, MALTA_IDE_LBAM, 1));

		temp = (secno >> 16) & 0xff; // 设置操作扇区号的 [23:16] 位
		panic_on(syscall_write_dev(&temp, MALTA_IDE_LBAH, 1));

		temp = ((secno >> 24) & 0x0f) | MALTA_IDE_LBA | (diskno << 4); // 设置操作扇区号的 [27:24] 位，设置扇区寻址模式，设置磁盘编号
		panic_on(syscall_write_dev(&temp, MALTA_IDE_DEVICE, 1));

		temp = MALTA_IDE_CMD_PIO_READ; // 设置 IDE 设备为读状态
		panic_on(syscall_write_dev(&temp, MALTA_IDE_STATUS, 1));

		temp = wait_ide_ready(); // 等待 IDE 设备就绪

		for (int i = 0; i < SECT_SIZE / 4; i++) { // 读取扇区数据
			panic_on(syscall_read_dev(dst + offset + i * 4, MALTA_IDE_DATA, 4));
		}

		panic_on(syscall_read_dev(&temp, MALTA_IDE_STATUS, 1)); // 读取寄存器，确认操作是否成功

		offset += SECT_SIZE; // 进入下一个扇区
		secno += 1; // 进入下一个扇区
	}
}
```

- `ide_write`：

```c
void ide_write(u_int diskno, u_int secno, void *src, u_int nsecs) { // 实现向IDE磁盘写入数据，src是待写入磁盘的数据
	uint8_t temp; // 临时变量
	u_int offset = 0, max = nsecs + secno; // 偏移量与终止扇号
	panic_on(diskno >= 2); // 确保磁盘编号合法

	while (secno < max) { // 从sector_number开始循环写入
		temp = wait_ide_ready(); // 等待 IDE 设备就绪
		temp = 1;
		panic_on(syscall_write_dev(&temp, MALTA_IDE_NSECT, 1)); // 设置操作扇区数目

		temp = secno & 0xff; // 设置操作扇区号的 [7:0] 位
		panic_on(syscall_write_dev(&temp, MALTA_IDE_LBAL, 1));

		temp = (secno >> 8) & 0xff; // 设置操作扇区号的 [15:8] 位
		panic_on(syscall_write_dev(&temp, MALTA_IDE_LBAM, 1));

		temp = (secno >> 16) & 0xff; // 设置操作扇区号的 [23:16] 位
		panic_on(syscall_write_dev(&temp, MALTA_IDE_LBAH, 1));

		temp = ((secno >> 24) & 0x0f) | MALTA_IDE_LBA | (diskno << 4); // 设置操作扇区号的 [27:24] 位，设置扇区寻址模式，设置磁盘编号
		panic_on(syscall_write_dev(&temp, MALTA_IDE_DEVICE, 1));
        
		temp = MALTA_IDE_CMD_PIO_WRITE; // 设置 IDE 设备为写状态
		panic_on(syscall_write_dev(&temp, MALTA_IDE_STATUS, 1));

		temp = wait_ide_ready(); // 等待 IDE 设备就绪

		for (int i = 0; i < SECT_SIZE / 4; i++) { // 写入数据
			panic_on(syscall_write_dev(src + offset + i * 4, MALTA_IDE_DATA, 4));
		}

		panic_on(syscall_read_dev(&temp, MALTA_IDE_STATUS, 1));

		offset += SECT_SIZE; // 偏移量增加
		secno += 1; // 扇区号增加
	}
}
```

#### 2.2 文件系统结构

##### 2.2.1 磁盘文件系统布局

![image-20250519160232254](Lab5.assets/image-20250519160232254.png)

- 从图中可以看到，MOS 操作系统把磁盘最开始的一个磁盘块（4096字节）当作引导扇 区和分区表使用。接下来的一个磁盘块作为超级块（SuperBlock），用来描述文件系统的基本信 息，如Magic Number、磁盘大小以及根目录的位置。
- MOS操作系统中超级块的结构如下：

```c
struct Super {
 	u_int s_magic; // Magic number: FS_MAGIC
	u_int s_nblocks;  // Total number of blocks on disk
 	struct File s_root; // Root directory node
 };
```

- 其中每个域的意义如下：  
  - s_magic：魔数，为一个常量，用于标识该文件系统
  - s_nblocks：记录本文件系统有多少个磁盘块，在本文件系统中为1024。 
  - s_root：根目录，其 f_type 为 FTYPE_DIR，f_name 为“/”

- `fs/fs.c`文件中的`free_block`：释放磁盘块号为 `blockno` 的磁盘块，置为空闲，即清除

```c
void free_block(u_int blockno) {
	if (blockno == 0 || blockno >= super->s_nblocks) {
		return; // 检查块号有效性，不能为0也不能大于最大块数
	}
	bitmap[blockno / 32] |= 1 << (blockno & 0x1f);
    // [blockno / 32]计算块号对应的位图数组索引，(blockno & 0x1f)计算块号在整数中的位移
}
```

##### 2.2.2 文件系统详细结构

- 操作系统要想管理一类资源，就得有相应的数据结构。对于描述和管理文件来说，一般使用 文件控制块（File结构体）。其定义如下：

```c
 struct File {
 	char f_name[MAXNAMELEN]; // 文件名称
 	uint32_tf_size; // 文件大小
 	uint32_tf_type; // 文件类型
 	uint32_tf_direct[NDIRECT]; // 文件的直接指针
 	uint32_tf_indirect; // 文件的间接指针
 
 	structFile *f_dir; // 指向文件所属文件目录
 	char f_pad[FILE_STRUCT_SIZE-MAXNAMELEN-(3 + NDIRECT)* 4-sizeof(void *)]; // 为了让整数个文件结构体占用一个磁盘块，填充结构体中剩下的字节
 }__attribute__((aligned(4),packed));
```

- 对于普通的文件，其指向的磁盘块存储着文件内容，而对于目录文件来说，其指向的磁盘块存储着该目录下各个文件对应的文件控制块。当我们要查找某个文件时，首先从超级块中读取根 目录的文件控制块，然后沿着目标路径，挨个查看当前目录所包含的文件是否与下一级目标文件同名，如此便能查找到最终的目标文件。

- `tools/fsformat`文件中的`create_file`函数：函数实现了一个在目录中创建新文件的功能，它会查找目录中未使用的文件槽位，如果没有找到则分配新的块来存储文件。

  在 `dirf` 指向的文件控制块所代表的目录下分配一个文件控制块，并返回指向新分配的文件控制块的指针

```c
struct File *create_file(struct File *dirf) {
	int nblk = dirf->f_size / BLOCK_SIZE; // 计算目录使用的块数
	for (int i = 0; i < nblk; ++i) { // 遍历目录的每一个块
		int bno; // 
		if (i < NDIRECT) { // 如果小于直接指针数量
			bno = dirf->f_direct[i]; // 从直接指针数组中获取
		} else { // 如果大于直接指针数量
			bno = ((int *)(disk[dirf->f_indirect].data))[i]; // 从间接块中获取
		}

		struct File *blk = (struct File *)(disk[bno].data); // 将块号转换为内存中的File结构体指针

		for (struct File *f = blk; f < blk + FILE2BLK; ++f) { // 遍历块中的每一个File指针
			if (f->f_name[0] == '\0') { // 检查该文件是否被使用
				return f; // 返回未使用的文件指针
			}
		}
	}

	int bno = make_link_block(dirf, nblk); // 调用make_link_block函数为目录分配一个新块
	return (struct File *)(disk[bno].data); // 返回新块中的第一个File结构指针(新块中的所有槽位都未使用)
	return NULL;
}
```

##### 2.2.3 块缓存

- 使用块缓存的原因：块缓存指的是借助虚拟内存来实现磁盘块缓存的设计。由于对于硬盘I/O操作一次只能读 取少量数据，同时频繁读写磁盘会拖慢系统，故常用块缓存将数据暂存于内存，减少磁盘访问次 数，从而加快数据读取。下列函数来管理缓冲区内的内存。
- `fs/fs.c`文件中的`disk_addr`函数：计算指定磁盘块对应的虚拟地址。

```c
void *disk_addr(u_int blockno) {
	return (void *)(DISKMAP + blockno * BLOCK_SIZE);
}
```

- 当把一个磁盘块中的内容载入到内存中时，需要为之分配对应的物理内存；当结束使用这 一磁盘块时，需要释放对应的物理内存以回收操作系统资源。`fs/fs.c`中的map_block 函数和 unmap_block 函数实现了这一功能。

```c
int map_block(u_int blockno) {
	if (block_is_mapped(blockno)) { // 检查块是否已映射
		return 0; // 如果已经在缓存中则返回
	}
	return syscall_mem_alloc(0, disk_addr(blockno), PTE_D); // 调用syscall_mem_alloc系统调用分配内存页面
}
```

```c
void unmap_block(u_int blockno) {
	void *va;
	va = block_is_mapped(blockno); // 获取缓存页面的虚拟地址

	if (!block_is_free(blockno) && block_is_dirty(blockno)) { // 检查块是否正在使用，检查块是否被修改过
		write_block(blockno); // 若都满足，将修改写回磁盘
	}

	panic_on(syscall_mem_unmap(0, va)); // 调用 syscall_mem_unmap 系统调用解除内存映射

	user_assert(!block_is_mapped(blockno)); // 验证块确实已被解除映射
}
```

- read_block 函数和 write_block 函数用于读写磁盘块。read_block 函数将指定编号的磁 盘块读入到内存中，首先检查这块磁盘块是否已经在内存中，如果不在，先分配一页物理内存， 然后调用ide_read 函数来读取磁盘上的数据到对应的虚存地址处。 file_get_block 函数用于将某个指定的文件指向的磁盘块读入内存。其主要分为 2 个步骤：首先为即将读入内存的磁盘块分配物理内存，然后用read_block函数将磁盘内容以块为 单位读入内存中的相应位置。这两个步骤对应的函数都借助了系统调用来完成。 在完成块缓存部分之后我们就可以实现文件系统中的一些文件操作了。

- `fs/fs.c`文件中的`dir_lookup`函数：查找某个目录下是否存在指定的文件

  该函数的主要流程如下：

  -  遍历目录文件  dir 中的每一个磁盘数据块。（使 file_get_block 获取第 i 个数据块的数据。）
  - 遍历单个磁盘块中保存的所有文件控制块。 
  -  判断要查找的文件名 name 与当前文件控制块的  f_name 字段是否相同，若相同则返回该文件控制块。 若未找到则返回 -E_NOT_FOUND 。

```c
int dir_lookup(struct File *dir, char *name, struct File **file) {
	u_int nblock;

	nblock = dir->f_size / BLOCK_SIZE; // 计算目录占用块数

	for (int i = 0; i < nblock; i++) { // 遍历目录的每个块
		void *blk;

		try(file_get_block(dir, i, &blk)); // 读取目录的第i个块的数据
		struct File *files = (struct File *)blk; // 将块指针转化为文件指针

		for (struct File *f = files; f < files + FILE2BLK; ++f) { // 遍历块中的所有文件
			if (strcmp(name, f->f_name) == 0) { // 如国与目标文件名称相同
				*file = f; // 设置输出参数
				f->f_dir = dir; // 设置该文件的父目录指针为当前目录
				return 0; // 成功则返回0
			}
		}
	}

	return -E_NOT_FOUND;
}
```

- 这里我们给出了文件系统结构中部分函数可能的调用参考，认真理解 每个文件、函数的作用和之间的关系。

![image-20250520221010775](Lab5.assets/image-20250520221010775.png)

#### 2.3  文件系统的用户接口

##### 2.3.1 文件描述符

- fd 即FileDiscripter，是系统给用户提供的整数，用于其在描述符表(Descriptor Table) 中进行索引。我们在作为操作系统的使用者进行文件I/O 编程时，使用open 在描 述符表的指定位置存放被打开文件的信息；使用close将描述符表中指定位置的文件信息 释放；在write 和read 时修改描述符表指定位置的文件信息。这里的“指定位置”即文件描述符fd。

- 两类指针struct Fd *与struct Filefd *指针的定义如下：

  ```c
  struct Fd {
      u_int fd_dev_id;
      u_int fd_offset;
      u_int fd_omode;
  };
  
  struct Filefd {
      struct Fd f_fd;
      u_int f_fileid;
      struct File f_file;
  }
  ```

- `user/lib/file.c`文件中的`open`函数：尝试打开文件，若成功则该函数返回文件描述符的编号

  ```c
  int open(const char *path, int mode) {
  	int r; // 存储函数调用的返回结果
  	struct Fd *fd; // 指向文件描述符结构体的指针
  	r = fd_alloc(&fd); // 分配一个新的fd结构体
  	if (r) { // 分配失败则返回
  		return r;
  	}
  
  	r = fsipc_open(path, mode, fd); // 通过IPC请求文件系统打开文件
  	if (r) { // 失败则返回
  		return r;
  	}
  
  	char *va;
  	struct Filefd *ffd;
  	u_int size, fileid;
      
  	va = fd2data(fd); // 获取文件数据在用户空间的虚拟地址
  	ffd = (struct Filefd *)fd; // 将fd强制转换为Filefd结构
  	size = ffd->f_file.f_size; // 从ffd中提取文件大小
  	fileid = ffd->f_fileid; // 从ffd中提取文件ID
  
  	for (int i = 0; i < size; i += PTMAP) { // 分块映射文件内容到内存
  		r = fsipc_map(fileid, i, va + i); // 通过 IPC 请求文件系统将文件的第 i 字节开始的块映射到用户空间地址 va + i
  		if (r) {
  			return r;
  		}
  	}
  
  	return fd2num(fd); // 返回文件描述符（将 struct Fd* 转换为整数）
  }
  ```

- `user/lib/fd.c`文件中的`read`函数：从文件描述符 `fdnum` 读取数据到缓冲区 `buf`

  ```c
  int read(int fdnum, void *buf, u_int n) {
  	int r; // 存储返回值
  
  	struct Dev *dev; // 申明Dev指针
  	struct Fd *fd; // 申明fd指针
  
  	if ((r = fd_lookup(fdnum, &fd)) < 0) { // 调用fd_look通过文件描述符编号查找对应文件描述符指针
  		return r;
  	}
  
  	if ((r = dev_lookup(fd->fd_dev_id, &dev)) < 0) {
  		return r; // 通过文件描述符的设备ID查找对应设备结构指针
  	}
  
  	if ((fd->fd_omode & O_ACCMODE) == O_WRONLY) {
  		return -E_INVAL; // 确认文件是否只以只写模式打开
  	}
  
  	r = dev->dev_read(fd, buf, n, fd->fd_offset); // 调用设备特定的dev_read函数进行实际读取操作
  
  	if (r > 0) { // 如果成功则更新文件描述符的偏移量
  		fd->fd_offset += r;
  	}
  
  	return r;
  }
  ```


##### 2.2.3 文件系统服务

- `fs/serv.c`文件中的`serve_remove`函数：处理文件删除请求

```C
void serve_remove(u_int envid, struct Fsreq_remove *rq) {
	int r; // 存储返回值
	r = file_remove(rq->req_path); // 调用file_remove函数删除path指定文件

	ipc_send(envid, r, 0, 0); // 将删除操作结果发送回请求进程
}
```

- `user/lib/fsipc.c`文件中的`fsipc._remove`函数：

```c
int fsipc_remove(const char *path) {
	if (path[0] == '\0' || strlen(path) >= MAXPATHLEN) {
		return -E_BAD_PATH; // 检查路径是否为空字符串，长度是否超过最大长度
	}

	struct Fsreq_remove *req = (struct Fsreq_remove *)fsipcbuf; // 将全局缓冲区fsipcbuf强制转换为Fsreq_remove结构体指针

	strcpy((char *)req->req_path, path); // 将用户提供的路径字符串复制到请求结构的req_path字段

	return fsipc(FSREQ_REMOVE, req, 0, 0); // 调用fsipc函数发送文件删除请求
}
```

- `user/lib/file.c`文件中的 `remove`函数：

```C
int remove(const char *path) {
	return fsipc_remove(path); // 调用封装好的函数
}
```

### 3.实验体会

#### 3.1 lab5-1 Exam

##### 3.1.1 获取系统时间

![image-20250526100107534](Lab5.assets/image-20250526100107534.png)

```c
int time_read() {
    int time = 0;
    panic_on(syscall_read_dev(time, 0x15000000, 4));
    panic_on(syscall_read_dev(time, 0x15000010, 4));
    return time;
}
```

##### 3.1.2 简易磁盘阵列

![image-20250526101704097](Lab5.assets/image-20250526101704097.png)

```c
void raid0_write(u_int secno, void *src, u_int nsecs) {
    int i;
    for (i = secno; i < secno + nsecs; i++) {
        if (i % 2 == 0) {
            ide_write(1, i / 2, src + (i - secno) * SECT_SIZE, 1);
        } else {
            ide_write(2, i / 2, src + (i - secno) * SECT_SIZE, 1);
        }
    }
}
void raid0_write(u_int secno, void *src, u_int nsecs) {
	u_int offset = 0
	for (u_int i = secno; i < secno + nsecs; i++) {
		if (i % 2 == 0) {
			ide_write(1, i / 2, src + offset, 1);
		} else {
			ide_write(2, i / 2, src + offset, 1);
		}
		offset += SECT_SIZE;
	}	
}
```

```c
void raid0_read(u_int secno, void *dst, u_int nsecs) {
    int i;
    for (i = secno; i < secno + nsecs; i++) {
        if (i % 2 == 0) {
            ide_read(1, i / 2, dst + (i - secno) * (SECT_SIZE / 2), 1);
        } else {
            ide_read(2, i / 2, dst + (i - secno) * (SECT_SIZE / 2), 1);
        }
    }
}
```

##### 3.1.2  时间查询与计算

**题目背景**

在 Lab5 与 Lab2 中，我们分别实现了不同种类外设的读写操作，现在我们需要实现一个能够获取当前物理时间的函数，和一个要求进程必须等待指定时间后才能运行的函数。现在我们介绍如何获取**实时间**。

GXemul 中除了磁盘 `disk` 还存在另一类外设——实时时钟 `rtc` (Real-Time Clock)，其中记录了 Unix 时间戳，即从格林威治时间 1970 年 01 月 01 日 00 时 00 分 00 秒起至现在的总秒数。为了进一步精确 Unix 时间戳， `rtc` 中还记录了一个微秒字段。

当我们需要读取 `rtc` 中存放的时钟的值时，需要**先触发时钟更新**，才能获取到正确的数据，未更新时首次读取默认是 0（别问我怎么知道的）

在 GXemul 的手册中我们能查阅到 `rtc` 的起始地址为 `0x15000000`，且不同偏移量下的读写效果如下：

| 偏移     | 效果                                                         | 数据位宽 |
| -------- | ------------------------------------------------------------ | -------- |
| `0x0000` | 读/写：触发时钟更新                                          | 4 bytes  |
| `0x0010` | 读：从格林威治时间 1970 年 01 月 01 日 00 时 00 分 00 秒起至现在的总秒数 | 4 bytes  |
| `0x0020` | 读：为精确 Unix 时间戳而记录的微秒数，范围为 0 到 999999 微秒（1秒内） | 4 bytes  |

**题目要求:**

本题需要完成两个用户态中的函数，并要求在 `user/include/lib.h` 中声明，`user/lib/ipc.c` 中实现

1. `u_int get_time(u_int *us)`

从 `rtc` 中读取**总秒数**和**微秒数**，总秒数作为该函数的返回值，微秒数通过指针参数 `us` 返回。

1. `void usleep(u_int us)`

在任一用户态进程进入 `usleep` 函数后，必须停留 `us` 微秒后才能退出函数。这其中允许时间片的轮转，也就是说**在等待调度的过程中仍然计时**

- 请避免单位换算时可能发生的整数溢出
- `u_int` 为无符号整数，如果需要进行减法得到有符号整数，请在**运算前**使用 `(int)` 进行强制类型转换
- 课程组给出了的一种 `usleep` 的实现（填空即可）

answer:

```c
u_int get_time(u_int *us) {
    u_int temp = 0;
    panic_on(syscall_read_dev(&temp, 0x15000000, 4));
    panic_on(syscall_read_dev(&temp, 0x15000010, 4));
    panic_on(syscall_read_dev(us, 0x15000020, 4));
    return temp;
}

void usleep(u_int us) {
	// 读取进程进入 usleep 函数的时间
    u_int inTime = 0;
    u_int inUs = 0;
    inTime = get_time(&inUs);
	while (1) {
		// 读取当前时间
        u_int nowUs = 0;
        u_int nowTime = 0;
        nowTime = get_time(&nowUs);
        u_int dif = nowTime - inTime;
		if (dif * 1000000 + (nowUs - inUs) >= us/* 当前时间 >= 进入时间 + us 微秒*/) {
			return;
		} else {
			// 进程切换
            syscall_yield();
		}
	}
}
```

#### 3.2 lab5-1 Extra

```c
// 首先按照合适的逻辑规定好数据结构
u_int ssd_bitmap[32];
u_int ssd_cleanup[32];
u_int ssd_logic[32];
u_int ssd_physics[32];
```

```c
void ssd_init() {
    for (int ssdno = 0; ssdno < 32; ssdno++) {
        ssd_bitmap[ssdno] = 1;
        ssd_physics[ssdno] = (u_int)0xffffffff;
        ssd_logic[ssdno] = (u_int)oxffffffff;
    }
}
```

```c
int ssd_read(u_int logic_no, void *dst) {
    u_int physics_no = get_physics(logic_no);
    if (physics_no == (u_int)0xffffffff) { retrun -1; }
    ide_read(0, physics_no, dst, 1);
    return 0;
}
```

```c
void ssd_erase(u_int logic_no)  {
    u_int physics_no = get_physics(logic_no);
    if (physics_no == (u_int)0xffffffff) { return; }
    
    for (int ssdno = 0; ssdno < 32; ssdno++) {
        if (ssd_logic[ssdno] == logic_no) {
            erase(ssd_physics[ssdno]);
            ssd_logic[ssdno] = (u_int)0xffffffff;
            ssd_physics[ssdno] = (u_int)0xffffffff;
        }
    }
}
void erase(u_int physics_no) {
    char buf[512];
    ssd_cleanmap[physics_no]++;
    ssd_bitmap[physics_no] = 1;
    for (int i = 0; i < 512; i++) {
        buf[i] = 0;
    }
    ide_write(0, physics_no, buf, 1);
}
```

```c
u_int alloc_physics() {
    u_int target_no   = 1000;  // 低负荷下选出的可用块
    u_int erase_time  = 1000;  // 标记最少擦除次数
    u_int exchange_no = 1000;  // 高负荷下选出的替换块

    char buf[512];             // 用作转存时使用的中介

    /* Step 1: 根据选择逻辑获得擦除最少的可用物理块块号和擦除次数 */
    for (int ssdno = 0; ssdno < 32; ssdno++) {
        if (ssd_cleanmap[ssdno] < erase_time && ssd_bitmap[ssdno] == 1) {
            target_no = ssdno;
            erase_time = ssd_cleanmap[ssdno];
        }
    }

    /* Step 2: 低负荷状态可以直接结束函数，同时恭喜你拿到了 80 分 */
    if (erase_time < 5) { return target_no; }

    /* Step 3: 高负荷状态下，按照选择逻辑再次获得替换块*/
    erase_time = 1000;
    for (int ssdno = 0; ssdno < 32; ssdno++) {
        if (ssd_bitmap[ssdno] == 0 && ssd_cleanmap[ssdno] < erase_time) {
            exchange_no = ssdno;
            erase_time = ssd_cleanmap[ssdno];
        }
    }

    /* Step 4: 初始化转存的区域，不写也行 */
    for (int i = 0; i < 512; i++) { buf[i] = 0; }

    /* Step 5: 替换块擦除，做出提到的相应更新 */
    ssd_bitmap[target_no] = 0;                   // 1.
    for (int ssdno = 0; ssdno < 32; ssdno++) {   // 2.
        if (ssd_physics[ssdno] == exchange_no) {
            ssd_physics[ssdno] = target_no;
        }
    }
    ide_read(0, exchange_no, buf, 1);            // 3.
    ide_write(0, target_no, buf, 1);
    ssd_cleanmap[exchange_no]++;                 // 4.
   
    return exchange_no;
}
```

```c
void ssd_write(u_int logic_no, void *src) {
    u_int physics_no = (u_int)0xffffffff;
    
    /* Step 1: 先擦除原有物理块，销毁映射 */
    for (int ssdno = 0; ssdno < 32; ssdno++) {
        if (ssd_logic[ssdno] == logic_no) {
            physics_no = ssd_physics[ssdno];
            if (physics_no != (u_int)0xffffffff) {
                ssd_erase(logic_no);
            }
        }
    }
    
    /* Step 2: 根据分配逻辑申请一个新物理块，同时新建映射，标记使用情况 */
    u_int save_no = alloc_physics();
    ssd_bitmap[save_no] = 0;
    // debugf("alloc physics %d\n", save_no);
    for (int ssdno = 0; ssdno < 32; ssdno++) {
        if (ssd_logic[ssdno] == (u_int)0xffffffff) {
            ssd_logic[ssdno] = logic_no;
            ssd_physics[ssdno] = save_no;
            break;
        }
    }
    
    /* Step 3: 最后进行写入的操作 */
    ide_write(0, save_no, src, 1);
}
```

#### 3.3 lab5-2 Exam

##### 3.3.1 相对路径打开

- 首先我们需要整理用户进程调用文件系统进程函数的流程，以`open`函数为例：
  1. `user/file.c`文件中`open(const char *path, int mode)`为最顶层函数，它调用了`fsipc_open`函数获得文件描述符。
  2. `user/fsipc.c`文件中的`fsipc_open(const char *path, u_int mode, struct Fd *fd)`函数调用了调用函数`fsipc(FSREQ_OPEN,  req, fd, &perm)`函数。其中系统调用号在`user/include/fsreq.h`中。
  3. `fs/serv.c`文件中的`serve_open`函数调用了`fs/fs.c`中的`file_open`函数。
  4. `fs/fs.c`中的`file_open`函数主体是上面的`walk_path`函数

- 根据上面的步骤，我们来实现新的fsipc_*的全过程。
  1. `user/include/fsreq.h`增加文件服务进程的请求类型FSREQ_OPENAT和请求结构体struct Fsreq_openat；完成基础的数据结构准备。
  2. `user/lib/file.c`仿照open函数实现openat函数；供用户程序直接启动openat流程。
  3. `user/lib/fsipc.c`仿照 fsipc_open 实现 fsipc_openat；完成对 Fsreq_openat 各个字段的赋值；使其能发送 `openat` 的请求
  4. `fs/serv.c`修改serve函数使其能转发FSREQ__OPENAT请求；仿照serve_open实现serve_openat函数。
  5. `fs/fs.c`实现`walk_path_at`和`file_open_at`函数
  
  ```c
  // user/file.c
  
  int openat(int dirfd, const char *path, int mode) {
  	struct Fd *filefd;
  	int r;
  
  	/* Step 1: 获取相对目录在进程中的 fd，申请待打开的文件的 fd */
  	fd_lookup(dirfd, &dir);
  
  	struct Fd *fd;
  	r = fd_alloc(&fd);
  	if (r) {
  		return r;
  	}
  	
  	/* Step 2: 根据目录在文件服务进程中的 fileid，发送请求 */
  	int dir_fileid = ((struct Fliefd *)dir)->f_fileid;
  	r = fsipc_openat(dir_fileid, path, mode, fd);
  	if (r) {
  		return r;
  	}
  	
  	char *va;
  	struct Filefd *ffd;
  	u_int size, fileid;
  	
  	va = fd2data(fd);
  	ffd = (struct Filefd *)fd;
  	size = ffd->f_file.f_size;
  	fileid = ffd->f_fileid;
  
  	for (int i = 0; i < size; i += PTMAP) {
  
  		r = fsipc_map(fileid, i, va + i);
  		if (r) {
  			return r;
  		}
  	}
  
  	return fd2num(fd);
  }
  ```
  
  ```c
  // user/lib/fsipc.c
  
  int fsipc_open_at(u_int dir_fileid, const char *path, u_int omode, struct Fd *fd) {
  	u_int perm;
  	struct Fsreq_openat *req;
  
  	req = (struct Fsreq_open *)fsipcbuf;
  
  	// The path is too long.
  	if (strlen(path) >= MAXPATHLEN) {
  		return -E_BAD_PATH;
  	}
  
  	strcpy((char *)req->req_path, path);
  	req->req_omode = omode;
      req->dir_fileid = dir_fielid;
  	return fsipc(FSREQ_OPENAT, req, fd, &perm);
  }
  ```
  
  ```c
  // fs/serv.c
  
  void serve_openat(u_int envid, struct Fsreq_openat *rq) {
  	struct File *f;
  	struct Filefd *ff;
  	int r;
  	struct Open *o;
      struct Open *pOpen;
      
      if ((r = open_lookup(envid, rq->dir_fileid, &pOpen)) < 0) {
          ipc_send(envid, r, 0, 0);
          return;
      }
  
  	if ((r = open_alloc(&o)) < 0) {
  		ipc_send(envid, r, 0, 0);
  		return;
  	}
  
  	if ((rq->req_omode & O_CREAT) && (r = file_create(rq->req_path, &f)) < 0 &&
  	    r != -E_FILE_EXISTS) {
  		ipc_send(envid, r, 0, 0);
  		return;
  	}
  
  	if ((r = file_openat(dir, rq->req_path, &f)) < 0) {
  		ipc_send(envid, r, 0, 0);
  		return;
  	}
  
  	o->o_file = f;
  
  	if (rq->req_omode & O_TRUNC) {
  		if ((r = file_set_size(f, 0)) < 0) {
  			ipc_send(envid, r, 0, 0);
  		}
  	}
  
  	ff = (struct Filefd *)o->o_ff;
  	ff->f_file = *f;
  	ff->f_fileid = o->o_fileid;
  	o->o_mode = rq->req_omode;
  	ff->f_fd.fd_omode = o->o_mode;
  	ff->f_fd.fd_dev_id = devfile.dev_id;
  	ipc_send(envid, 0, o->o_ff, PTE_D | PTE_LIBRARY);
  }
  ```
  
  ```c
  // fs/fs.c
  
  int file_open(struct File *dir, char *path, struct File **file) {
  	return walk_path(dir, path, 0, file, 0);
  }
  ```
  
  ```c
  // fs/fs.c
  
  int walk_path(struct File *par_dir, char *path, struct File **pdir, struct File **pfile, char *lastelem) {
  	char *p;
  	char name[MAXNAMELEN];
  	struct File *dir, *file;
  	int r;
  
  	// start at the par_dir.
  	path = skip_slash(path);
  	file = par_dir;
  	dir = 0;
  	name[0] = 0;
  
  	if (pdir) {
  		*pdir = 0;
  	}
  
  	*pfile = 0;
  
  	// find the target file by name recursively.
  	while (*path != '\0') {
  		dir = file;
  		p = path;
  
  		while (*path != '/' && *path != '\0') {
  			path++;
  		}
  
  		if (path - p >= MAXNAMELEN) {
  			return -E_BAD_PATH;
  		}
  
  		memcpy(name, p, path - p);
  		name[path - p] = '\0';
  		path = skip_slash(path);
  		if (dir->f_type != FTYPE_DIR) {
  			return -E_NOT_FOUND;
  		}
  
  		if ((r = dir_lookup(dir, name, &file)) < 0) {
  			if (r == -E_NOT_FOUND && *path == '\0') {
  				if (pdir) {
  					*pdir = dir;
  				}
  
  				if (lastelem) {
  					strcpy(lastelem, name);
  				}
  
  				*pfile = 0;
  			}
  
  			return r;
  		}
  	}
  
  	if (pdir) {
  		*pdir = dir;
  	}
  
  	*pfile = file;
  	return 0;
  }
  ```

- 总之，通过本次实验，我深入理解了操作系统文件系统的核心机制与设备驱动的实现原理。在实现内存映射I/O时，我掌握了地址对齐验证和安全边界检查的重要性；开发IDE磁盘驱动过程中，我熟悉了设备控制流程和寄存器编程技巧，特别是28位LBA地址的分段设置。文件系统结构的实现让我认识到磁盘块缓存的设计价值，以及目录查找、文件操作等核心功能的实现逻辑。通过文件描述符和系统调用接口的实现，我理解了用户态与内核态之间的交互机制。这些实践不仅加深了我对操作系统理论知识的理解，也培养了我解决复杂系统问题的能力。
