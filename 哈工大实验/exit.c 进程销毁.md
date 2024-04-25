# exit.c 进程销毁

进程的销毁流程：

- 释放进程的代码段和数据段占用的内存
- 关闭进程打开的所有文件，对当前的目录和i节点进行同步（文件操作）
- 如果当前要销毁的进程有子进程，那么就让1号进程作为新的父进程（init进程）
- 如果当前进程是一个会话头进程，则会终止会话中的所有进程
- 改变当前进程的运行状态，编程TASK_ZOMBLE僵死状态，并且向其父进程发送SIGCHLD信号。

从父进程的角度考虑：父进程在运行子进程的时候，一般都会运行wait waitpid 这两个函数（父进程等待耨个子进程的终止）。当父进程收到SIGCHLD信号时父进程会终止僵死状态的子进程。父进程会把子进程的运行时间累加到自己的进程变量中，把对应的子进程描述结构体进程释放，置空任务数组中的空槽。

对应exit.c文件中关键的函数do_exit以及sys_waitpid（代码略）。

```c
int do_exit(long code)
{
	int i;
	//释放内存页
	free_page_tables(get_base(current->ldt[1]),get_limit(0x0f));
	free_page_tables(get_base(current->ldt[2]),get_limit(0x17));
	//current->pid就是当前需要关闭的进程
	for (i=0 ; i<NR_TASKS ; i++)
		if (task[i] && task[i]->father == current->pid) {//如果当前进程是某个进程的父进程
			task[i]->father = 1;//就让1号进程作为新的父进程
			if (task[i]->state == TASK_ZOMBIE)//如果是僵死状态
				/* assumption task[1] is always init */
				(void) send_sig(SIGCHLD, task[1], 1);//给父进程发送SIGCHLD
		}
	for (i=0 ; i<NR_OPEN ; i++)//每个进程能打开的最大文件数NR_OPEN=20
		if (current->filp[i])
			sys_close(i);//关闭文件
	iput(current->pwd);
	current->pwd=NULL;
	iput(current->root);
	current->root=NULL;
	iput(current->executable);
	current->executable=NULL;
	if (current->leader && current->tty >= 0)
		tty_table[current->tty].pgrp = 0;//清空终端
	if (last_task_used_math == current)
		last_task_used_math = NULL;//清空协处理器
	if (current->leader)
		kill_session();//清空session
	current->state = TASK_ZOMBIE;//设为僵死状态
	current->exit_code = code;
	tell_father(current->father);
	schedule();
	return (-1);	/* just to suppress warnings */
}
```

还有一些就是do_exit以及sys_waitpid中会调用的函数

```c
// 释放对应内存页
void release(struct task_struct * p)
{
    /*
    	释放内存页，并没有释放task_struct
    */
    free_page((long) p );
    schedule();
}

// 给指定的p进程发送信号
static inline int send_sig(long sig, struct task_struct * p, int priv);

// 关闭会话
static void kill_session(void);

// 系统调用 向任何进程 发送任何信号
/*
    在sys_kill存在对pid号的判断
    pid > 0 给对应的pid发送sig
    pid = 0 给当前进程的进程组发送sig
    pid = -1 给任何进程发送
    pid < -1 给进程组号为-pid的进程组发送信号
*/
int sys_kill(int pid, int sig);

// 告诉父进程
static void tell_father(int pid);
```

