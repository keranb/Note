# 进程的调度sched.c文件

这其中包含对文件`include/linux/sched.h`文件的分析。

`void show_task(int nr,struct task_struct * p)`用来打印pid号和state。`void show_stat(void)`用于打印当前所有的进程信息（pid号和state）。

## schedule()函数

```c
void schedule(void)
{
	int i,next,c;
	struct task_struct ** p;

/* check alarm, wake up any interruptible tasks that have got a signal */

	for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
		if (*p) {//alarm是用来设置警告，比如jiffies有1000个可能其中一些需要警告那么就用alarm来实现
			if ((*p)->alarm && (*p)->alarm < jiffies) {
					(*p)->signal |= (1<<(SIGALRM-1));
					(*p)->alarm = 0;
				}
				//~(_BLOCKABLE & (*p)->blocked  
				//&&(*p)->state==TASK_INTERRUPTIBLE
				//用来排除非阻塞信号
				//如果该进程为可中断睡眠状态 则如果该进程有非屏蔽信号出现就将该进程的状态设置为running
			if (((*p)->signal & ~(_BLOCKABLE & (*p)->blocked)) &&
			(*p)->state==TASK_INTERRUPTIBLE)
				(*p)->state=TASK_RUNNING;
		}

/* this is the scheduler proper: */
	// 以下思路，循环task列表 根据counter大小决定进程切换
	while (1) {
		c = -1;
		next = 0;
		i = NR_TASKS;
		p = &task[NR_TASKS];
		while (--i) {
			if (!*--p)
				continue;//进程为空就继续循环
			if ((*p)->state == TASK_RUNNING && (*p)->counter > c)//找出c最大的task
				c = (*p)->counter, next = i;
		}
		if (c) break;//如果c找到了，就终结循环，说明找到了
		//进行时间片的重新分配
		for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
			if (*p)//这里很关键，在低版本内核中，是进行优先级时间片轮转分配，这里搞清楚了优先级和时间片的关系
			//counter = counter/2 + priority
				(*p)->counter = ((*p)->counter >> 1) +
						(*p)->priority;
	}
	//切换到下一个进程 这个功能使用宏定义完成的
	switch_to(next);
}
```

首先是关于信号和警告的一些处理，这部分暂且不管，在进程间通信部分在介绍。

接着就是一个while循环，这个while循环就是我们的优先级时间片轮转调度算法。简单来说就是找寻最大时间片的进程。我们在`schedule`函数开头申请了一个指向`task_struct`指针的指针。原因如下：

![task](E:\操作系统（哈工大实验，笔记）\imags\task.png)

我们的`task`进程数组其实是各个`task_struct`的地址。我们在while循环开始的时候让p指针指向NR_TASKS号进程对应的地址（这个进程其实是不存在的，仅仅是方便我们后续操作）。那后续操作也就很简单了，找出counter最大且进程状态为`TASK_RUNNING`的进程。很明显，最后找到的那个进程号放在变量next中，对应的时间片放在变量c中。如果我们找到的c不为0，说明存在着这么一个进程，就退出。如果没找到，说明当前所有进程的时间片都用完了，进行时间片的重新分配。因为并未退出当前while循环，继续寻找最大counter对应的进程。所以，不管怎么样，最后一定会从while循环中退出，然后使用`switch_to`切换到我们找到的这个线程。

## switch_to()

`switch_to`是定义在`include/linux/sched.h`下的一段宏汇编

```c
#define switch_to(n) {\
struct {long a,b;} __tmp; \
__asm__("cmpl %%ecx,_current\n\t" \
	"je 1f\n\t" \
	"movw %%dx,%1\n\t" \
	"xchgl %%ecx,_current\n\t" \
	"ljmp %0\n\t" \
	"cmpl %%ecx,_last_task_used_math\n\t" \
	"jne 1f\n\t" \
	"clts\n" \
	"1:" \
	::"m" (*&__tmp.a),"m" (*&__tmp.b), \
	"d" (_TSS(n)),"c" ((long) task[n])); \
}
```

有些问题我们先不去深究，大概看下`switch_to`做了什么。首先将`__tmp.`和`__tmp.b`的值都放入内存中，至于为什么使用`*&__tmp.a`，我们这里不得而知。接着将进程n的TSS段的段选择子放入edx寄存器中，将进程n的`task_struct`的地址放入ecx寄存器中。开始执行汇编代码，首先判断n号进程是否为当前进程，若是则跳转到标号1处，相当于直接终止进程的切换。接着将edx寄存器中的值放入内存，偏移为1处（这里我们无需去管ds寄存器中的段选择子是是什么）。接着交换当前进程与ecx寄存器中的n号进程。

然后是一个非常重要的`ljmp`指令

按AS手册，ljmp指令存在两种形式，即：
  一、直接操作数跳转，此时操作数即为目标逻辑地址（选择子，偏移），即形如：ljmp \$seg_selector, \$offset的方式；
  二、使用内存操作数，这时候，AS手册规定，内存操作数必须用“\*”作前缀，即形如：ljmp \*mem48，其中内存位置mem48处存放目标逻辑地址: 高16bit存放的是seg_selector，低32bit存放的是offset。注意：这条指令里的“\*”只是表示间接跳转的意思，与C语言里的“\*”作用完全不同。

回到源码上，`ljmp %0`用的ljmp的第二种用法，`ljmp *%0`这条语句展开后相当于`ljmp *__tmp.a`，也就是跳转到`&__tmp.a`中包含的48bit逻辑地址处。而按struct \_tmp的定义，这也就意味着`__tmp.a`即为该逻辑地址的offset部分，`__tmp.b`的低16bit为seg_selector(高16bit无用)部分。通过以上说明，可以知道了ljmp将跳转到选择子指定的地方，大致过程是，ljmp判断选择子为TSS类型，于是就告诉硬件要切换任务，硬件首先它要将当前的PC,esp,eax等现场信息保存在当前自己的TSS段描述符中,然后再将目标TSS段描述符中的pc,esp,eax的值拷贝至对应的寄存器中。会实现由0特权级的内核代码到3特权级的用户代码的切换。

