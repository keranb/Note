# 进入操作系统

> 系统达到怠速状态前所作的一切准备工作的核心目的就是让用户程序能够以”进程的“方式正常运行。能够实现这一目的的标准包含三个方面的内容：用户程序能够在主机上运行计算，能够与外设进行交互，以及能够让用户以它为媒介进行人机交互。

引导程序结束之后，通过main.c进入操作系统。其中main()函数是入口

```c
void main(void)		/* This really IS void, no error here. */
{			/* The startup routine assumes (well, ...) this */
/*
 * Interrupts are still disabled. Do necessary setups, then
 * enable them
 */
//前面这里做的所有事情都是在对内存进行拷贝
 	ROOT_DEV = ORIG_ROOT_DEV;//设置操作系统的根文件
 	drive_info = DRIVE_INFO;//设置操作系统驱动参数
	 //解析setup.s代码后获取系统内存参数
	memory_end = (1<<20) + (EXT_MEM_K<<10);
	//取整4k的内存大小
	memory_end &= 0xfffff000;
	if (memory_end > 16*1024*1024)//控制操作系统的最大内存为16M
		memory_end = 16*1024*1024;
	if (memory_end > 12*1024*1024) 
		buffer_memory_end = 4*1024*1024;//设置高速缓冲区的大小，跟块设备有关，跟设备交互的时候，充当缓冲区，写入到块设备中的数据先放在缓冲区里，只有执行sync时才真正写入；这也是为什么要区分块设备驱动和字符设备驱动；块设备写入需要缓冲区，字符设备不需要是直接写入的
	else if (memory_end > 6*1024*1024)
		buffer_memory_end = 2*1024*1024;
	else
		buffer_memory_end = 1*1024*1024;
	main_memory_start = buffer_memory_end;
#ifdef RAMDISK
	main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
#endif
//内存控制器初始化
	mem_init(main_memory_start,memory_end);
	//异常函数初始化
	trap_init();
	//块设备驱动初始化
	blk_dev_init();
	//字符型设备出动初始化
	chr_dev_init();
	//控制台设备初始化
	tty_init();
	//加载定时器驱动
	time_init();
	//进程间调度初始化
	sched_init();
	//缓冲区初始化
	buffer_init(buffer_memory_end);
	//硬盘初始化
	hd_init();
	//软盘初始化
	floppy_init();
	sti();// 开中断
	//从内核态切换到用户态，上面的初始化都是在内核态运行的
	//内核态无法被抢占，不能在进程间进行切换，运行不会被干扰
	move_to_user_mode();
	if (!fork()) {	//创建1号进程 fork函数就是用来创建进程的函数	/* we count on this going ok */
		//0号进程是所有进程的父进程
		init();
	}
/*
 *   NOTE!!   For any other task 'pause()' would mean we have to get a
 * signal to awaken, but task0 is the sole exception (see 'schedule()')
 * as task 0 gets activated at every idle moment (when no other tasks
 * can run). For task0 'pause()' just means we go check if some other
 * task can run, and if not we return here.
 */
//0号进程永远不会结束，他会在没有其他进程调用的时候调用，只会执行for(;;) pause();
	for(;;) pause();
}
```

首先，纵观整个main函数，所作的无非四件事：

- 设置根设备，硬盘
- 规划物理内存格局，设置缓冲区，虚拟盘，主内存
- 一些初始化
- 用fork()函数创建0号进程
- 调用init()函数继续初始化

## 设置根设备，硬盘

内核首先初始化根设备和硬盘，用bootsect中写入机器系统数据0x901FC的信息（根设备为软盘），设置软盘为根设备，并用起始自0x90080的32字节的机器系统数据的硬盘参数表设置内核中的硬盘信息dirve_info。

```c
#define DRIVE_INFO (*(struct drive_info *)0x90080)
#define ORIG_ROOT_DEV (*(unsigned short *)0x901FC)
ROOT_DEV = ORIG_ROOT_DEV;//设置操作系统的根文件
drive_info = DRIVE_INFO;//设置操作系统驱动参数
```

## 规划物理内存格局，设置缓冲区，虚拟盘，主内存

具体规划如下：除内核代码和数据所占的内存空间之外，其余物理内存主要分为三部分，分别是主内存区，缓冲区和虚拟盘。主内存区是进程代码运行的空间，也包括内核管理进程的数据结构；缓冲区主要作为主机与外设进行数据交互的中转站；虚拟盘区是一个可选的区域，如果选择使用虚拟盘，就可以将外设上的数据先复制进虚拟盘区，然后加以使用。

```c
#define EXT_MEM_K (*(unsigned short *)0x90002)
memory_end = (1<<20) + (EXT_MEM_K<<10);// 1MB + 扩展内存(MB)数
memory_end &= 0xfffff000;// 按页的倍数取整，忽略内存末端不足一页的部分
if (memory_end > 16*1024*1024)// 控制操作系统的最大内存为16M
	memory_end = 16*1024*1024;
if (memory_end > 12*1024*1024) 
	buffer_memory_end = 4*1024*1024;
else if (memory_end > 6*1024*1024)
	buffer_memory_end = 2*1024*1024;
else
	buffer_memory_end = 1*1024*1024;
main_memory_start = buffer_memory_end;// 缓冲区之后就是主存
#ifdef RAMDISK
	main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
#endif
```

## time_init()

加载定时器驱动，`time_init`就定义在`main.c`文件中。

```c
static void time_init(void)//这一段代码起到从CMOS中读取时间信息的作用
{
	struct tm time;

	do {
		time.tm_sec = CMOS_READ(0);
		time.tm_min = CMOS_READ(2);
		time.tm_hour = CMOS_READ(4);
		time.tm_mday = CMOS_READ(7);
		time.tm_mon = CMOS_READ(8);
		time.tm_year = CMOS_READ(9);
	} while (time.tm_sec != CMOS_READ(0));
	BCD_TO_BIN(time.tm_sec);//把CMOS中读出来的数据进行转换
	BCD_TO_BIN(time.tm_min);
	BCD_TO_BIN(time.tm_hour);
	BCD_TO_BIN(time.tm_mday);
	BCD_TO_BIN(time.tm_mon);
	BCD_TO_BIN(time.tm_year);
	time.tm_mon--;
	startup_time = kernel_mktime(&time);//存在startup_time这个全局变量中，并且之后会被JIFFIES使用
	
}
```

系统从CMOS硬件中读取时间信息，并将读到的时间参数传入`kernel_mktime`函数中，`kernel_mktime`函数定义在`kernel/mktime.c`中。

```c
long kernel_mktime(struct tm * tm)//这个tm是开机的时候从CMOS读出来的
{
	long res;
	int year;

	year = tm->tm_year - 70;
/* magic offsets (y+1) needed to get leapyears right.*/
	res = YEAR*year + DAY*((year+1)/4);
	res += month[tm->tm_mon];
/* and (y+2) here. If it wasn't a leap-year, we have to adjust */
	if (tm->tm_mon>1 && ((year+2)%4))
		res -= DAY;
	res += DAY*(tm->tm_mday-1);
	res += HOUR*tm->tm_hour;
	res += MINUTE*tm->tm_min;
	res += tm->tm_sec;
	return res;
}
```

这个函数会基于传入的时间返回当前时间距离1970年1月1日0时所过的秒数，在`time_init`中，会将结果存放在全局变量`start_time`中。`start_time`后续会被jiffies系统滴答所使用（10ms一个滴答）。

每隔10ms会引发一个定时器中断`_timer_interrupt`，这个中断定义在`system_call.s`中（软中断放在system_call.s中这是可以预见的）。

```c
_timer_interrupt:
	push %ds		# save ds,es and put kernel data space
	push %es		# into them. %fs is used by _system_call
	push %fs
	pushl %edx		# we save %eax,%ecx,%edx as gcc doesn't
	pushl %ecx		# save those across function calls. %ebx
	pushl %ebx		# is saved as we use that in ret_sys_call
	pushl %eax
	movl $0x10,%eax
	mov %ax,%ds
	mov %ax,%es
	movl $0x17,%eax
	mov %ax,%fs
	incl _jiffies //自加自身
	movb $0x20,%al		# EOI to interrupt controller #1
	outb %al,$0x20
	movl CS(%esp),%eax
	andl $3,%eax		# %eax is CPL (0 or 3, 0=supervisor)
	pushl %eax//以上是在中断时对现场进行保存
	call _do_timer		# 'do_timer(long CPL)' does everything from
	addl $4,%esp		# task switching to accounting ...
	jmp ret_from_sys_call
```

`_timer_interrupt`所做的很简单，首先进行一些压栈，然后将`0x10`赋给`%ds`和`%es`寄存器，把`0x17`赋给`%fs`寄存器。`0x10`段选择子代表内核数据段，`0x17`代表0号进程数据段。然后将`_jiffies`自加。后面两步操作不太懂，暂且不管。然后是取CS寄存器后三位赋给`%eax`，并将其入栈，然后调用函数`_do_timer`，那刚入栈的那个值就作为`_do_timer`函数的参数。对于`_do_timer`函数，我们主要关注它基于`CPL`将对应的运行时间+1。

```c
void do_timer(long cpl)
{
	extern int beepcount;
	extern void sysbeepstop(void);

	if (beepcount)
		if (!--beepcount)
			sysbeepstop();

	if (cpl)//cpl表示当前被中断的进程是用户态还是内核态
		current->utime++;//给用户程序运行时间+1
	else
		current->stime++;//内核程序运行时间+1

	if (next_timer) {// next_timer 是连接jiffies变量的所有定时器的事件链表
	// 可以这样想象，jiffies是一个时间轴，然后这个时间轴上每个绳结上绑了一个事件，运行到该绳结就触发对应的事件
		next_timer->jiffies--;
		while (next_timer && next_timer->jiffies <= 0) {
			void (*fn)(void);
			
			fn = next_timer->fn;
			next_timer->fn = NULL;
			next_timer = next_timer->next;
			(fn)();
		}
	}
	if (current_DOR & 0xf0)//取高四位
		do_floppy_timer();
	if ((--current->counter)>0) return;
	current->counter=0;//counter进程的时间片为0，task_struct[]是进程的向量表 
	//counter在哪里用？ 进程的调度就是task_struct[]中检索，找时间片最大的进程对象来运行 直到时间片为0退出 之后再进行新一轮调用
	//counter在哪里被设置？ 当task_struct[]所有进程的counter都为0，就进行新一轮的时间片分配
	if (!cpl) return;
	schedule();//这个就是进行时间片分配
}
```

## shced_init()

我们先忽略内核拷贝，看看初始化，前面的一些硬件初始化我们暂且不管，先来看看`sched_init()`进程间调度初始化。`sched_init()`函数定义在`kernel/sched.c`中，同时存在与之对应的`include/linux/sched.h`文件。两个结合着理解。`shed_init()`代码如下：

```c
void sched_init(void)
{
	int i;
	struct desc_struct * p;

	if (sizeof(struct sigaction) != 16)
		panic("Struct sigaction MUST be 16 bytes");
	//gdt是全局描述符（系统级别）和前面所说的ldt（局部描述符）对应
	//内核的代码段
	//内核的数据段
	//进程0...n的数据
	set_tss_desc(gdt+FIRST_TSS_ENTRY,&(init_task.task.tss));
	set_ldt_desc(gdt+FIRST_LDT_ENTRY,&(init_task.task.ldt));
	p = gdt+2+FIRST_TSS_ENTRY;
	for(i=1;i<NR_TASKS;i++) {//0-64进程进行遍历
		task[i] = NULL;
		p->a=p->b=0;
		p++;
		p->a=p->b=0;
		p++;
	}//作用是清空task链表
/* Clear NT, so that we won't have troubles with that later on */
	__asm__("pushfl ; andl $0xffffbfff,(%esp) ; popfl");
	ltr(0);
	lldt(0);
	//以下都是设置一些小的寄存器组
	outb_p(0x36,0x43);		/* binary, mode 3, LSB/MSB, ch 0 */
	outb_p(LATCH & 0xff , 0x40);	/* LSB */
	outb(LATCH >> 8 , 0x40);	/* MSB */
	set_intr_gate(0x20,&timer_interrupt);
	outb(inb_p(0x21)&~0x01,0x21);
	//设置系统中断
	set_system_gate(0x80,&system_call);
}
```

**首先，声明了一个指针p，指向`desc_struct`**

```c
struct desc_struct * p;
```

`struct desc_strudt`定义在`include/linux/head.h`中

```c
typedef struct desc_struct {
	unsigned long a,b;
} desc_table[256];
```

我们可以看出`desc_struct`就是包含两个4字节变量一共八个字节的单元。对应在内核中就是我们的段描述符，a是段描述符的低32位（包括段限长和段基址），b是段描述符的高32位（包括段属性）。

**接下来是设置进程0的TSS和LDT**

```c
set_tss_desc(gdt+FIRST_TSS_ENTRY,&(init_task.task.tss));
set_ldt_desc(gdt+FIRST_LDT_ENTRY,&(init_task.task.ldt));
```

我们先看看里面的参数，其中`FIRST_TSS_ENTRY`，`FIRST_LDT_ENTRY`是两个宏

```c
#define FIRST_TSS_ENTRY 4
#define FIRST_LDT_ENTRY (FIRST_TSS_ENTRY+1)
```

为啥子FIRST_TSS_ENTRY的值定义为4呢？ 这是因为所有进程的TSS和LDT都是保存在系统的GDT 全局描述符表里面的。GDT前四项分别是空描述符，内核代码段描述符，内核数据段描述符，以及空描述符。之所以内核数据段后面任然有一个空描述符，设计者主要是想将内核描述符与进程的TSS,LDT分开。我们可以基于head.s中对gdt的设定进行验证

```c
gdt:	
	.quad 0x0000000000000000	/* NULL descriptor */
	.quad 0x00c09a0000000fff	/* 16Mb */
	.quad 0x00c0920000000fff	/* 16Mb */
	.quad 0x0000000000000000	/* TEMPORARY - don't use */
	.fill 252,8,0			/* space for LDT's and TSS's etc */
```

结合下面的图片可以非常直观的理解

![屏幕截图 2022-05-30 120558](E:\操作系统（哈工大实验，笔记）\imags\屏幕截图 2022-05-30 120558.png)

接着就是gdt参数，定义在`include/linux/head.h`中

```c
extern desc_table idt,gdt;
```

`desc_table`就是来自刚才看到的

```c
typedef struct desc_struct {
	unsigned long a,b;
} desc_table[256];
```

也就是说gdt和idt就是段描述符的数组。那这样`gdt+FIRST_TSS_ENTRY`，就指向了GDT表中刚好应该放第0号进程描述符的位置。

然后我们看看对应调用的函数`set_tss_desc`

```c
#define _set_tssldt_desc(n,addr,type) \
__asm__ ("movw $104,%1\n\t" \
	"movw %%ax,%2\n\t" \
	"rorl $16,%%eax\n\t" \
	"movb %%al,%3\n\t" \
	"movb $" type ",%4\n\t" \
	"movb $0x00,%5\n\t" \
	"movb %%ah,%6\n\t" \
	"rorl $16,%%eax" \
	::"a" (addr), "m" (*(n)), "m" (*(n+2)), "m" (*(n+4)), \
	 "m" (*(n+5)), "m" (*(n+6)), "m" (*(n+7)) \
	)
#define set_tss_desc(n,addr) _set_tssldt_desc(((char *) (n)),((int)(addr)),"0x89")
#define set_ldt_desc(n,addr) _set_tssldt_desc(((char *) (n)),((int)(addr)),"0x82")

```

我们以`set_tss_desc`为例，首先将n转换为指向char的指针，也就是指向一个字节的指针，这样做是为了在`_set_tssldt_desc`宏中在对n进行加法的时候每次只加一个字节，可以一个字节一个字节的传参。将addr转换为int类型，传入`_set_tssldt_desc`宏中进行处理。

对于这个内联汇编

第0个参数："a" (addr)，将addr也就是&(init_task.task.tss)给eax.
第1个参数："m" (\*(n))，一个内存数，将n的值也就是gdt+4这个数，作为一个内存地址，这个其实就是gdt中进程0的TSS描述符的首字节。第0-1个字节共16位应该放着这个段的限长。
第2个参数： "m" (\*(n+2))，一个内存数，gdt中进程0的TSS描述符的第2个字节。第2-3字节放着这个基地址的低16位。
第3个参数："m" (\*(n+4))，进程0的tss段描述的第4个字节。这里面放的基地址的中8位。
第4个参数： "m" (\*(n+5))，进程0的tss段描述符的第5个字节，里面放的类型
第5个参数："m" (\*(n+6))，进程0的tss段描述符的第6个字节，里面放的段限长，粒度啥的
最后一个参数："m" (*(n+7))，进程0的tss段描述符的第7个字节，里面放的基地址的高8位。
所以这些参数其实就是构成了这个段描述符的各个部分。

"movw \$104,%1\n\t"设置段限长为104个单位。具体多少得看G位。也就是倒数第二个字节的最高位。
"movw %%ax,%2\n\t" \将输入的&(init_task.task.tss)地址给基地址的低16位。
"rorl \$16,%%eax\n\t" \，将eax右循环移动16位
"movb %%al,%3\n\t" \，al其中就是地址的中8位，第3个参数。
"movb \$" type ",%4\n\t" \,编译后就是"movb \$0x89,%4\n\t" \,而0x89=1000 1001。底4位1001是类型域，第5位S为0，说明这是一个系统描述符，而类型又是1001，说明这是一个32位的TSS系统描述符。第6-7位说明这个段的DPL为0，最高位存在位置1 。
"movb \$0x00,%5\n\t" \,给第5个参数，这个参数是控制粒度啥的，通过这个最高位为0，可以看出这个段的粒度为1B。所以这个段的限长为104个字节。其实TSS结构的长度就是104字节。
"movb %%ah,%6\n\t" \将基地址的最后的最高8位写到第6个参数。
"rorl \$16,%%eax" \,eax再右循环移位就是让eax设置为0了。

**经过上述过程，将gdt里面的第一个TSS描述符就做好了，这个描述符的基地址指向`init_task.tss`。同理LDT0也是这样操作，结合上副图进行理解。**

对于`init_task`就是我们0号进程的`task_struct`，定义在`include/linux/sched.h`中

```c
#define INIT_TASK \
/* state etc */	{ 0,15,15, \
/* signals */	0,{{},},0, \
/* ec,brk... */	0,0,0,0,0,0, \
/* pid etc.. */	0,-1,0,0,0, \
/* uid etc */	0,0,0,0,0,0, \
/* alarm */	0,0,0,0,0,0, \
/* math */	0, \
/* fs info */	-1,0022,NULL,NULL,NULL,0, \
/* filp */	{NULL,}, \
	{ \
		{0,0}, \
/* ldt */	{0x9f,0xc0fa00}, \
		{0x9f,0xc0f200}, \
	}, \
/*tss*/	{0,PAGE_SIZE+(long)&init_task,0x10,0,0,0,0,(long)&pg_dir,\
	 0,0,0,0,0,0,0,0, \
	 0,0,0x17,0x17,0x17,0x17,0x17,0x17, \
	 _LDT(0),0x80000000, \
		{} \
	}, \
}
```

同时在`sched.c`中有这么一段代码

```c
union task_union {
	struct task_struct task;
	char stack[PAGE_SIZE];
};

static union task_union init_task = {INIT_TASK,};

long volatile jiffies=0;
long startup_time=0;
struct task_struct *current = &(init_task.task);//全局变量 指向当前运行的进程
struct task_struct *last_task_used_math = NULL;

struct task_struct * task[NR_TASKS] = {&(init_task.task), };

long user_stack [ PAGE_SIZE>>2 ] ;
```

我们只挑关键的来看，首先我们将`INIT_TASK`放入`init_task`联合体中，那么我们对`INIT_TASK`的访问就应该是`init_task.task`。然后我们将`init_task.task`的地址赋给current（当前进程，此时就是0号进程）。最后初始化了我们的task数组，并且让第一项指向`init_task.task`的地址。代表着我们task任务数组中的第一个任务（进程）就是0号进程。

OK，那么总的来看，经过下面两条语句，我们将TSS0和LDT0就做好了。

```c
set_tss_desc(gdt+FIRST_TSS_ENTRY,&(init_task.task.tss));
set_ldt_desc(gdt+FIRST_LDT_ENTRY,&(init_task.task.ldt));
```

接着：

```c
p = gdt+2+FIRST_TSS_ENTRY;
	for(i=1;i<NR_TASKS;i++) {
		task[i] = NULL;
		p->a=p->b=0;
		p++;
		p->a=p->b=0;
		p++;
	}
```

那这段程序就很简单了，让指针指向`gdt+2+FIRST_TSS_ENTRY`，也就是gdt表中的第七个元素（索引为6）。然后进行遍历，将`task`第一个到第NR_TASKS-1个元素都赋值为NULL。在每次赋值的过程中，都让对应的gdt表中的两个描述符清0（对应进程的ldt和tss）

接着：

```c
ltr(0);
lldt(0);
```

`ltr`和`lldt`是两个宏，定义在`include/linux/sched.h`中

```c
#define _TSS(n) ((((unsigned long) n)<<4)+(FIRST_TSS_ENTRY<<3))
#define _LDT(n) ((((unsigned long) n)<<4)+(FIRST_LDT_ENTRY<<3))
#define ltr(n) __asm__("ltr %%ax"::"a" (_TSS(n)))
#define lldt(n) __asm__("lldt %%ax"::"a" (_LDT(n)))
```

首先看`_TSS(n)`，这其实就是生成第n号进程TSS段的段选择子，首先`(FIRST_TSS_ENTRY<<3)` 代表将后三位空出来，`000`代表全局描述表以及特权级。而`(((unsigned long) n)<<4)`代表首先左移三位也是空出后三位，左移一位是因为一个进程有两个描述符（前面是偏移）。 `lldt(0)`的分析类似。需要注意的是我们`_TSS(n)`返回的是`unsigned long`类型的数据，并将其赋给eax寄存器，但是最后ltr指令仅操作低16位也就是`%ax`寄存器。

最后，挂载时间中断服务程序和系统调用也就是0x80中断服务程序。

```c
	outb_p(0x36,0x43);		/* binary, mode 3, LSB/MSB, ch 0 */
	outb_p(LATCH & 0xff , 0x40);	/* LSB */
	outb(LATCH >> 8 , 0x40);	/* MSB */
	set_intr_gate(0x20,&timer_interrupt);
	outb(inb_p(0x21)&~0x01,0x21);
	set_system_gate(0x80,&system_call);
```

**总的来讲：**

sched_init所作的是如下几件事：

- 根据task数组中初始化的task[0]指向的task_struct即init进程在GDT表中创建0号进程的TSS和LDT描述符
- 将task数组中的后续进程清空，并将GDT表中这些进程的描述符清空
- 生成进程0的LDT和TSS的段选择子（在GDT/LDT表中的索引，这里是在GDT表中的索引，当前特权级）。并设置对应的TR和LDTR寄存器。
- 挂载时钟中断服务程序和系统调用中断服务程序

## 开启中断

现在，系统中所有中断服务程序都已经和IDT正常挂接。这意味着中断服务体系已经构建完毕，系统可以在32位保护模式下处理中断，重要意义之一是可以使用系统调用。

## move_to_user_mode()

**从内核态切换到用户态**

我们先回过去看看我们的`INIT_TASK`

```c
#define INIT_TASK \
/* state etc */	{ 0,15,15, \
/* signals */	0,{{},},0, \
/* ec,brk... */	0,0,0,0,0,0, \
/* pid etc.. */	0,-1,0,0,0, \
/* uid etc */	0,0,0,0,0,0, \
/* alarm */	0,0,0,0,0,0, \
/* math */	0, \
/* fs info */	-1,0022,NULL,NULL,NULL,0, \
/* filp */	{NULL,}, \
	{ \
		{0,0}, \
/* ldt */	{0x9f,0xc0fa00}, \
		{0x9f,0xc0f200}, \
	}, \
/*tss*/	{0,PAGE_SIZE+(long)&init_task,0x10,0,0,0,0,(long)&pg_dir,\
	 0,0,0,0,0,0,0,0, \
	 0,0,0x17,0x17,0x17,0x17,0x17,0x17, \
	 _LDT(0),0x80000000, \
		{} \
	}, \
}
```

我们知道这个就是我们的0号进程的`task_struct`，我们关注其中的ldt和tss。

**ldt:**

```c
{ \
		{0,0}, \
		{0x9f,0xc0fa00}, \
		{0x9f,0xc0f200}, \
}\
```

ldt[0]是一个空描述符，ldt[1]是进程0的代码段描述符，ldt[2]是进程0的数据段描述符。（每个LDT中含有三个描述符，其中第一个不用，第二个是任务代码段的描述符，第三个是任务数据段和堆栈段的描述符。） 

在这其中有一个需要注意的点是：进程0的代码段就是内核的代码段。

那我们就先来看看进程0的ldt[1]。

`{0x9f,0xc0fa00}`展开完整(8个字节):`0x0000 009f,0x00c0 fa00`;其中第一个数在低32位,第二数在高32位
也就是:
高：
`0x00c0
0xfa00`
低：
`0x0000
0x009f`
转换为bit:

```c
63-48:00000000 11000000
47-32:11111010 00000000
31-16:00000000 00000000
15-00:00000000 10011111
```

对照着段描述符来看：

![屏幕截图 2022-05-29 085321](E:\操作系统（哈工大实验，笔记）\imags\屏幕截图 2022-05-29 085321.png)

15-00为段限长:`0x009f`
31-16(最低16位):`00000000 00000000` ,39-32(次低8位):`00000000`,63-56(高8位):`00000000` 合并起来构成段基址：`00000000 00000000 00000000 00000000`,就是地址0。它的DPL为46-45位:11,也就是3

再来看看内核GDT的代码段描述符，在`head.s`中

```c
gdt:
	.quad 0x0000000000000000	/* NULL descriptor */
	.quad 0x00c09a0000000fff	/* 16Mb */
	.quad 0x00c0920000000fff	/* 16Mb */
	.quad 0x0000000000000000	/* TEMPORARY - don't use */
	.fill 252,8,0			/* space for LDT's and TSS's etc */
```

第2项`0x00 c0 9a 00 00 00 0f ff`为内核代码段
高:
`0x00c0
0x9a00`
低:
`0x0000
0x0fff`

```
63-48:00000000 11000000
47-32:10011010 00000000
31-16:00000000 00000000
15-00:00001111 11111111
```

它的31-16(最低16位)为`00000000 00000000 `,39-32(次低8位)为`00000000`,63-56(高8位)为`00000000` 合并起来构成段基址：`00000000 00000000 00000000 00000000`
DPL:00,也就是0。因此，进程0的代码段基址与内核的段基址是相同的，都为0。

![屏幕截图 2022-05-31 154331](E:\操作系统（哈工大实验，笔记）\imags\屏幕截图 2022-05-31 154331.png)

**第二个，我们来看看tss的内容**

```c
/*tss*/	{0,PAGE_SIZE+(long)&init_task,0x10,0,0,0,0,(long)&pg_dir,\
	 0,0,0,0,0,0,0,0, \
	 0,0,0x17,0x17,0x17,0x17,0x17,0x17, \
	 _LDT(0),0x80000000, \
		{} \
	},
```

可以看到第二项为`PAGE_SIZE+(long)&init_task`，`init_task`我们前面介绍了，就是一个`task_union`联合体。

```c
union task_union {
	struct task_struct task;//小于4096
	char stack[PAGE_SIZE];//4096B
};
```

每个task_union都是包含4096字节，这刚好是一个页的大小。而这个task_union的前104个字节是一个task_struct结构。也就是说每个task_union的布局其实是这样的：
|---------------------------------------4096个字节----------------------------------------------------|
|-------task_struct----------|----------------------剩余未用---------------------------------------|
为什么会有这么多的剩余未用空间呢？其实这个剩余未用部分，就作为一个个进程的0特权栈来使用。我们都晓得，进程在运行期间很有可能使用调用system call函数的，而一调用system call函数，CPL就会从原来的3特权，翻转到0特权。而翻转特权以后，原来3特权的栈将不得被0特权的内核使用，于是Intel要求每个进程在创建之初都得指定一个0特权的栈，用于将来进程进入0特权时使用。

**move_to_user_mode()宏**

通过push和ret去修改寄存器的值从而完成特权级的切换

```c
#define move_to_user_mode() \
__asm__ (
	"movl %%esp,%%eax\n\t" \
	"pushl $0x17\n\t" \   
	"pushl %%eax\n\t" \
	"pushfl\n\t" \
	"pushl $0x0f\n\t" \ 
	"pushl $1f\n\t" \  
	"iret\n" \ 
	"1:\tmovl $0x17,%%eax\n\t" \ 
	"movw %%ax,%%ds\n\t" \
	"movw %%ax,%%es\n\t" \
	"movw %%ax,%%fs\n\t" \
	"movw %%ax,%%gs" \
	:::"ax")
```

我们一行行的分析。前面一部的压栈其实是在模拟INT n过程中需要执行的依次将ss,esp,eflag,cs,eip压栈。

`movl %%esp,%%eax`将esp内容给eax，esp是当前栈的栈顶，而当前用的是哪个栈呢？就是那个大小为一个页的user_stack的那个栈。
`pushl $0x17`，将0x17压入栈
`pushl %%eax`，将eax压入栈，也就是刚刚的那个旧的栈顶。
`pushfl` ，将eflag压栈
`pushl $0x0f` ，将0x0f压栈
`pushl $1f` ，将标号为1处指令的偏移压栈（它是iret的下一条指令）
看到这些push其实还不知道它要干什么，可以当看到iret时，我们就知道。
iret会隐含的执行以下指令：

```c
popl eip
popl cs
popl eflag
popl esp
popl ss
```

而这些pop就是与上面的push一一对应。因此，我们再回过头看看这些出栈以后寄存器保存的是啥。
首先eip 就是标号为1的指令的偏移，这是iret返回的下一条指令的偏移。

然后**cs为0x0f**,这是一个段选择子。0x0f=0000 1111,表示以RPL=3去选择LDT里面的第2项（索引为1）。当前的LDT表由对应的ldtr寄存器给出，在前面`shced_init()`函数中我们知道当前ldtr寄存器保存的是0号进程的ldt段选择符和ldt段描述符。那也就是说我们这个段选择子选择的是**0号进程的代码段**。
再然后是eflag为eflag，这个没啥好说的。
这样就把特权级从3转到0了，其他的东西依然没变化。
之后就是一波ds,es,fs,gs的对齐，都对齐到0x17选择子上。这个选择子0x17=0001 0111,即**以RPL=3选择LDT的第3项(二进制10为2）**，即**0号进程的数据段和栈段。**

当我们回顾整个流程，`move_to_user_mode()`函数执行完之后，就正式进入0号进程了。（也就是说我们可以在0号进程中执行指令了，而后续的任务fork()创建1号进程确实是在0号进程中进行的）。

一个进程想要在主机中正常运算需要做到以下条件：

1）在task中有一个指向自己task_struct的指针，且自己的task_struct已经创建并且初始化完毕。在GDT中存在着这个进程的TSS和LDT的段选择符。（在`sched_init()`中完成）

2）需要具备处理系统调用的能力（在`sched_init()`和`trap_init()`设置中断表，`sti()`函数开中断）

3）当前LDTR寄存器指向这个进程的ldt段选择符和段描述符，TR寄存器指向这个进程的tss段选择符和段描述符。（在`sched_init()`完成）

4）当前段寄存器保存正确的段选择符。（在`move_to_user_mode()`中完成）

**接着就是在0号进程中fork()创建1号进程**

