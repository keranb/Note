# 从main函数进入操作系统2

## fork()

`fork()`是一个系统调用（准确来说是系统调用接口函数），一般系统调用函数都定义在`kernel`目录下。

```c
static inline _syscall0(int,fork)
```

`_syscall0`是一个宏，代表着没有参数的系统调用，这个宏定义在`include/unistd.h`头文件中，其中还定义了系统调用号。其中关键部分我们在系统调用部分已经详细讲解了。当我们把宏展开之后发现关键就是执行了一个`0x80`中断。

```c
#define _syscall0(type,name) \
type name(void) \
{ \
long __res; \
__asm__ volatile ("int $0x80" \
	: "=a" (__res) \
	: "0" (__NR_##name)); \
if (__res >= 0) \
	return (type) __res; \
errno = -__res; \
return -1; \
}
```

而`0x80`中断对应的中断函数就是`system_call`，在中断向量表中找到中断函数的入口地址会进行跳转，在这里我们先看看system_call函数在中断向量表中的初始化：

```c
// 在shced_init中完成0x80号中断即系统调用的挂载
set_system_gate(0x80,&system_call);
#define set_system_gate(n,addr) set_gate(&idt[n],15,3,addr)

// set_gate宏汇编代码在system.h中
#define _set_gate(gate_addr,type,dpl,addr) \
__asm__ ("movw %%dx,%%ax\n\t" \
	"movw %0,%%dx\n\t" \
	"movl %%eax,%1\n\t" \
	"movl %%edx,%2" \
	: \
	: "i" ((short) (0x8000+(dpl<<13)+(type<<8))), \
	"o" (*((char *) (gate_addr))), \
	"o" (*(4+(char *) (gate_addr))), \
	"d" ((char *) (addr)),"a" (0x00080000))
```

很明显，_set_gate所作的就是在IDT表中填好对应的中断描述符，需要注意的是中断描述符的DPL为3，这样用户程序的CPL作为3也能够访问到，**而对应的中断函数入口地址为CS:0x0008,EIP:&system_call。当我们通过中断进行跳转的时候，特权级发生了改变，由3特权级跳转到0特权级，会发生堆栈的切换，由用户栈切换到内核栈**（这里的内核栈就是当前进程TSS中记录的那个内核栈，中断发生的时候硬件会自动跳转），保存用户态程序的信息，会向内核栈压入一定信息，然后真正进行跳转。

```c
.align 2
_system_call:
	cmpl $nr_system_calls-1,%eax
	ja bad_sys_call
	push %ds
	push %es
	push %fs
	pushl %edx
	pushl %ecx		# push %ebx,%ecx,%edx as parameters
	pushl %ebx		# to the system call
	movl $0x10,%edx		# set up ds,es to kernel data space
	mov %dx,%ds
	mov %dx,%es
	movl $0x17,%edx		# fs points to local data space
	mov %dx,%fs
	call _sys_call_table(,%eax,4)
	pushl %eax
	movl _current,%eax
	cmpl $0,state(%eax)		# state
	jne reschedule
	cmpl $0,counter(%eax)		# counter
	je reschedule
ret_from_sys_call:
	movl _current,%eax		# task[0] cannot have signals
	cmpl _task,%eax
	je 3f
	cmpw $0x0f,CS(%esp)		# was old code segment supervisor ?
	jne 3f
	cmpw $0x17,OLDSS(%esp)		# was stack segment = 0x17 ?
	jne 3f
	movl signal(%eax),%ebx
	movl blocked(%eax),%ecx
	notl %ecx
	andl %ebx,%ecx
	bsfl %ecx,%ecx
	je 3f
	btrl %ecx,%ebx
	movl %ebx,signal(%eax)
	incl %ecx
	pushl %ecx
	call _do_signal
	popl %eax
3:	popl %eax
	popl %ebx
	popl %ecx
	popl %edx
	pop %fs
	pop %es
	pop %ds
	iret
```

关键是`call _sys_call_table(,%eax,4)`根据系统调用号找到对应的系统调用，我们这里对应的系统调用就是`sys_fork()`

现在主要看看对应的系统调用函数的实现。我们知道，系统调用作为一个软中断，中断前的处理和中断后的恢复定义在`kernel/system_call.s`文件中，而具体的中断实现就定义在`kernel`目录下的某个文件中。

```c
.align 2
_sys_fork://fork的系统调用
	call _find_empty_process//调用这个函数
	testl %eax,%eax
	js 1f
	push %gs
	pushl %esi
	pushl %edi
	pushl %ebp
	pushl %eax
	call _copy_process//
	addl $20,%esp
1:	ret
```

这里有一个浅显的认知是：int指令和iret指令的配合使用与call指令和ret指令的配合使用具有相似的思路。会进行一定的入栈和出栈。（`int`指令会使CPU硬件自动将SS，ESP，EFLAGS，CS，EIP这5个寄存器入栈）

首先调用_find_empty_process函数在task中找到第一个空的位置

```c
int find_empty_process(void)
{
	int i;

	repeat:
		if ((++last_pid)<0) last_pid=1;
		for(i=0 ; i<NR_TASKS ; i++)
			if (task[i] && task[i]->pid == last_pid) goto repeat;
	for(i=1 ; i<NR_TASKS ; i++)
		if (!task[i])
			return i;
	return -EAGAIN;//达到64的最大值后，返回错误码
}
```

`repeat:`标号处的代码目前不明所以。还有一个需要注意的点就是函数的返回值放在`%eax`寄存器中，也就是说现在寄存器中就是我们找到的那个空位。

接着执行`testl %eax %eax`指令。testl指令,这个指令说是将两个操作数做与来设置零标志位和负数标识,常用的方法是testl %eax,%eax来检查%eax是正数负数还是0。

`js 1f `  SF符号位为负则进行跳转，也就是说没有找到空位进行跳转。

copy_process函数执行之前内核栈的视图：

![fork](E:\操作系统（哈工大实验，笔记）\imags\fork.png)

内核栈中的CS，EIP保存的就是int0x80后的那一个代码，也就是用户程序即将要执行的那个代码`if (__res >= 0) return (type) __res; \`。通过分析你可以发现，这些内核栈中的寄存器信息都是中断之前用户程序的信息。而copy_process就是基于这些信息创建一个新的用户进程。

```c
int copy_process(int nr,long ebp,long edi,long esi,long gs,long none,
		long ebx,long ecx,long edx,
		long fs,long es,long ds,
		long eip,long cs,long eflags,long esp,long ss)
{
	struct task_struct *p; 
	int i;
	struct file *f;
	//其实就是malloc分配内存
	p = (struct task_struct *) get_free_page();//在内存分配一个空白页，让指针指向它
	if (!p)
		return -EAGAIN;//如果分配失败就是返回错误
	task[nr] = p;//把这个指针放入进程的链表当中
	*p = *current;//把当前进程赋给p，也就是拷贝一份	/* NOTE! this doesn't copy the supervisor stack */
	//后面全是对这个结构体进行赋值相当于初始化赋值
	p->state = TASK_UNINTERRUPTIBLE;
	p->pid = last_pid;
	p->father = current->pid;
	p->counter = p->priority;
	p->signal = 0;
	p->alarm = 0;
	p->leader = 0;		/* process leadership doesn't inherit */
	p->utime = p->stime = 0;
	p->cutime = p->cstime = 0;
	p->start_time = jiffies;//当前的时间
	p->tss.back_link = 0;
	p->tss.esp0 = PAGE_SIZE + (long) p;
	p->tss.ss0 = 0x10;
	p->tss.eip = eip;
	p->tss.eflags = eflags;
	p->tss.eax = 0;//把寄存器的参数添加进来
	p->tss.ecx = ecx;
	p->tss.edx = edx;
	p->tss.ebx = ebx;
	p->tss.esp = esp;
	p->tss.ebp = ebp;
	p->tss.esi = esi;
	p->tss.edi = edi;
	p->tss.es = es & 0xffff;// 仅后16位有效
	p->tss.cs = cs & 0xffff;
	p->tss.ss = ss & 0xffff;
	p->tss.ds = ds & 0xffff;
	p->tss.fs = fs & 0xffff;
	p->tss.gs = gs & 0xffff;
	p->tss.ldt = _LDT(nr);
	p->tss.trace_bitmap = 0x80000000;
	if (last_task_used_math == current)//如果使用了就设置协处理器
		__asm__("clts ; fnsave %0"::"m" (p->tss.i387));
	if (copy_mem(nr,p)) {//老进程向新进程代码段和数据段进行拷贝
		task[nr] = NULL;//如果失败了
		free_page((long) p);//就释放当前页
		return -EAGAIN;
	}
	for (i=0; i<NR_OPEN;i++)//
		if (f=p->filp[i])//父进程打开过文件
			f->f_count++;//就会打开文件的计数+1，说明会继承这个属性
	if (current->pwd)//跟上面一样
		current->pwd->i_count++;
	if (current->root)
		current->root->i_count++;
	if (current->executable)
		current->executable->i_count++;
	set_tss_desc(gdt+(nr<<1)+FIRST_TSS_ENTRY,&(p->tss));
	set_ldt_desc(gdt+(nr<<1)+FIRST_LDT_ENTRY,&(p->ldt));
	p->state = TASK_RUNNING;//把状态设定为运行状态	/* do this last, just in case */
	return last_pid;//返回新创建进程的id号
}
```

`copy_process`函数作为创建新进程这个业务的主体函数。所作的事情是：

首先，声明一个指向`task_struct`的指针，作为当前新创建进程的task_struct。在内存分配一个空白页，让指针指向它。然后用当前进程的task_struct去初始化新创建进程的task_struct，然后进行具体的修改。

在这个过程中有几个关键点：

- `p->tss.esp0 = PAGE_SIZE + (long) p; p->tss.ss0 = 0x10;`说明新创建的进程的内核栈栈段指向内核数据段0x10，而%esp栈顶指针指向p+PAGE_SIZE，即指向当前分配页的末尾，可以这样理解，一个进程页就是PCB，我们把内核栈放在最高地址，其它的task_struct从最低地址开始扩展。
- `p->tss.eip = eip; p->tss.cs = cs & 0xffff; `，根据前面的分析cs,eip指向`int 0x80`后面那一行代码

后续是对内存管理的相关操作，我们关注在GDT表中对当前进程TSS和LDT段描述符的生成。最后`}`相当于执行`ret`指令回到`sys_fork()`函数执行`addl $20,%esp`。这个操作的目的是让我们的栈指针指向正确的返回地址，使得`ret`指令能够返回到`system_call`中的正确位置。`addl $20,%esp`直接跳过前面入栈的五个寄存器。这里给了我们一定启示，当我们使用`ret`回到调用函数正确位置处要求我们在call指令执行的入栈操作之后没有再进行入栈操作（当然可以入栈再出栈）。

总而言之，现在就回到了`system_call`函数。从`pushl %eax`处开始执行。而`%eax`寄存器中存放的是`sys_fork()`函数的返回值，而`sys_fork()`函数并未对`%eax`进行修改，那么返回的就是`copy_process`函数的返回值，即新创建进程的pid。

> 这里有一点需要补充的是，我们为什么要push %eax,是因为%eax寄存器中的值我们需要使用，而%eax寄存器我们后续也需要使用，于是就先把%eax中的数据压栈。等到需要使用的时候再出栈，放到%eax寄存器中

对于后续代码，我们关注以下几行：

```c
movl _current,%eax
	cmpl $0,state(%eax)		# state
	jne reschedule
	cmpl $0,counter(%eax)		# counter
	je reschedule
ret_from_sys_call:
	movl _current,%eax		# task[0] cannot have signals
	cmpl _task,%eax
	je 3f
```

判断当前进程是否为就绪态，如果不是就绪态，跳转到reschedule。判断当前进程是否还有时间片，如果没有时间片，也跳转到reschedule。然后`cmpl _task,%eax`判断当前进程是否是0号进程，如果是就跳转到标号3处。而我们当前进程确为0号进程。

```c
3:	popl %eax
	popl %ebx
	popl %ecx
	popl %edx
	pop %fs
	pop %es
	pop %ds
	iret
```

执行`popl %eax`，那么当前`%eax`寄存器中存放的就是新创建进程的pid号。最后进行一系列出栈操作之后（因为我们在`system_call()`执行开始时先进行了入栈，为了能使iret指令能够正确的返回，故需要执行一系列出栈）。iret进行中断返回，返回到了系统调用接口函数。

```c
#define _syscall0(type,name) \
type name(void) \
{ \
long __res; \
__asm__ volatile ("int $0x80" \
	: "=a" (__res) \
	: "0" (__NR_##name)); \
if (__res >= 0) \
	return (type) __res; \
errno = -__res; \
return -1; \
}

```

这里返回之后，首先会将`%eax`寄存器的值赋给_res变量，也就是新创建进程的pid，我们这里也就是1。进入后面的循环判断，那么我们当前系统调用接口函数的返回值就是1。

ok，接下来回到main主函数。因为fork()函数的返回值就是1，故不会执行init，执行pause()函数。一个重要的理解是fork()返回值非0就说明我们进程创建成功，那么我们就不应该在当前进程继续执行代码，会切换到新的进程，当前来讲就是1号进程，而1号进程保存的状态就是0号进程执fork()时的状态。

```c
if (!fork()) {	//创建0号进程 fork函数就是用来创建进程的函数	/* we count on this going ok */
		//0号进程是所有进程的父进程
		init();
	}
	for(;;) pause();
}
```

`pause()`函数也是一个系统调用接口函数

```c
static inline _syscall0(int,pause)
```

对应在`system_call_table`中的系统调用函数为`sys_pause`，就定义在`kernel/sched.c`文件中。中断会基于中断向量表中的中断函数地址进行跳转，在这里就进入了0特权级的内核代码。

```c
int sys_pause(void)
{
	current->state = TASK_INTERRUPTIBLE;
	schedule();
	return 0;
}
```

将当前进程（也就是0号进程）状态置为可中断睡眠状态。然后调用`schedule`进程调用函数，`schedule()`函数肯定会找到1号进程号然后切换到1号进程。具体见对`schedule()`函数的详细描述。要注意，此时各个寄存器中的值都因为进程切换发生了变换，依照1号进程tss段中对寄存器环境的保存。而1号进程tss段中的环境保存还是初始化时的样子。我们再次回到`copy_process()`函数中看看当时保存的是什么样的环境。

```c
	p->tss.eip = eip;
	p->tss.eflags = eflags;
	p->tss.eax = 0;//把寄存器的参数添加进来
```

这里的`%eip`指向fork()函数中的`if (__res >= 0)`，也就是中断指令后的第一条语句。而此时eax=0，所以fork()函数返回值为0，会在1号进程中指向init操作：

```c
if (!fork()) {	//创建0号进程 fork函数就是用来创建进程的函数	/* we count on this going ok */
	//0号进程是所有进程的父进程
	init();
}
```



