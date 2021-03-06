#【实现】用户态切换到内核态

CPU在用户态执行到switch_to_kernel()函数（即执行“int T_SWITCH_TOK”）时，由于当前处于用户态，而中断产生后，CPU会进入内核态，所以存在特权级转换。硬件会在内核栈中压入Error Code（可选）、EIP、CS和EFLAGS、ESP（用户态特权级的）、SS（用户态特权级的）（如下图所示），然后跳转到到IDT中记录的中断号T_SWITCH_TOK所对应的中断服务例程入口地址处继续执行。通过2.3.7小节“中断处理过程”可知，会执行到trap_disptach函数（位于trap.c）：

    case T_SWITCH_TOK:
        if (tf->tf_cs != KERNEL_CS) {
    //发出中断时，CPU处于用户态，我们希望处理完此中断后，CPU继续在内核态运行，
    //所以把tf->tf_cs和tf->tf_ds都设置为内核代码段和内核数据段
          tf->tf_cs = KERNEL_CS;
          tf->tf_ds = tf->tf_es = KERNEL_DS;
          //设置EFLAGS，让用户态不能执行in/out指令
          tf->tf_eflags &= ~(3 << 12);

          switchu2k = (struct trapframe *)(tf->tf_esp - (sizeof(struct trapframe) - 8));
          //设置临时栈，指向switchu2k，这样iret返回时，CPU会从switchu2k恢复数据，
    //而不是从现有栈恢复数据。
          memmove(switchu2k, tf, sizeof(struct trapframe) - 8);
          *((uint32_t *)tf - 1) = (uint32_t)switchu2k;
         }
         break;

这样在trap将会返回，在\__trapret:中，根据switchk2u的内容完成对返回前的寄存器和栈的回复准备工作，最后通过iret指令，CPU返回“int T_SWITCH_TOU”的后一条指令处，以内核态模式继续执行。
