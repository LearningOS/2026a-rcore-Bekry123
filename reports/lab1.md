```text
SBI: RustSBI
RustSBI Version: 0.3.0-alpha.2
RISC-V 适配版本： 1.0.0
RustSBI-QEMU Version: 0.2.0.alpha.2
```

"参考代码："

```asm
__alltraps:
    csrrw sp, sscratch, sp
    # now sp->kernel stack, sscratch->user stack
    # allocate a TrapContext on kernel stack
    addi sp, sp, -34*8
    # save general-purpose registers
    sd x1, 1*8(sp)
    # skip sp(x2), we will save it later
    sd x3, 3*8(sp)
    # skip tp(x4), application does not use it
    # save x5~x31
    .set n, 5
    .rept 27
        SAVE_GP %n
        .set n, n+1
    .endr
    # we can use t0/t1/t2 freely, because they were saved on kernel stack
    csrr t0, sstatus
    csrr t1, sepc
    sd t0, 32*8(sp)
    sd t1, 33*8(sp)
    # read user stack from sscratch and save it on the kernel stack
    csrr t2, sscratch
    sd t2, 2*8(sp)
    # set input argument of trap_handler(cx: &mut TrapContext)
    mv a0, sp
    call trap_handler
```

```asm
__restore:
    # now sp->kernel stack(after allocated), sscratch->user stack
    # restore sstatus/sepc
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    ld t2, 2*8(sp)
    csrw sstatus, t0
    csrw sepc, t1
    csrw sscratch, t2
    # restore general-purpuse registers except sp/tp
    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n
        .set n, n+1
    .endr
    # release TrapContext on kernel stack
    addi sp, sp, 34*8
    # now sp->kernel stack, sscratch->user stack
    csrrw sp, sscratch, sp
    sret
```

1.1 刚进入 `__restore` 时，sp 指向`_alltraps`中为它分配的`TrapContext`的初始地址，这个`TrapContext`保存GPR，user sp，`sstatus`，`sepc`等，sp会给 `ld` 命令提供基址。

1.2 `__restore`的两种应用场景，一个是OS第一次启动用户程序时`__restore`通过 `TrapContext` 恢复好留存的现场，然后`sret`进入U态；另一个是User程序发生traps后`__alltraps`保存 context ， 唤起 `trap_handler` ， 解决完要通过 `__restore` 恢复现场返回用户态

2.1 这几行代码分别恢复了先前保存到`TrapContext`的 `sstatus` 、 `sepc` 、 `sscratch`。

2.2 `sstatus` 保存当前处理器的状态信息，里面的特权级标志位会决定返回后去到什么模式

2.3 `sepc` 保存需要返回的程序地址。执行`sret`时会将`sepc`中的地址作为返回后的指令地址

2.4 `sscratch` 用于保存临时数据，用它辅助完成用户UserStack和KernelStack的交换

3 跳过x2是因为sp不像普通寄存器那样直接恢复，我们先前保存了`sscratch`，它在最后通过交换恢复

4 L60 后 sp 指向用户栈，`sscratch` 指向内核栈

5 `__store` 通过`sret`切换状态，这由spp和`sepc`决定，它们保存在`TrapContext`

6 L13后 sp指向内核栈，`sscratch`指向用户栈

7 U到S的切换应当是由`ecall`指令跳到S模式这也是那个文档的

#功能总结

    添加了record_syscall和get_syscall函数，添加了syscall_times统计数组，修改syscall/mod.rs中的系统调用分发逻辑，调用的时候记一次数，实现了sys_trace（410）系统调用，通过trace_request参数决定调用读、写还是查询调用次数（包含自身）。

#荣誉准则

我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。