# Linux0.11 中断代码

Linux0.11中断代码在kernel(内核)目录下。主要包括

asm.s和trap.c

system_call.s和`fork.c signal.c exit.c sys.c`

汇编代码实现中断前的准备和中断后的处理，c代码具体实现对应的中断函数，供汇编代码调用

