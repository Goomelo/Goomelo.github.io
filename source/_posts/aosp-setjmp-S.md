---
title: aosp_setjmp.S
date: 2022-02-18 14:52:51
tags:
---
# setjmp.S分析
## 如何使用setjmp
setjmp一般会创建本地的jmp_buf缓冲区并且初始化，用于将来跳转回此处。这个子程序保存程序的调用环境于env参数所指的缓冲区，env将被longjmp使用。如果是从setjmp直接调用返回，setjmp返回值为0。如果是从longjmp恢复的程序调用环境返回，setjmp返回非零值。

示例程序如下所示：

    1 #include <stdio.h>
    2 #include <setjmp.h>
    3 static jmp_buf buf;
    4 void func(){
    5     printf("in func\n");
    6     longjmp(buf,1);
    7 }
    8 int main(){
    9     if(setjmp(buf)) {
    10         printf("after jmp\n");
    11     }else{
    12         func();
    13     }
    14     return 0;
    15 }

运行结果为：

    in func
    after jmp

可以看出第一次运行到setjmp时，setjmp的返回值为0并且进入else分支，进入func（）函数中，在func（）函数中会调用longjmp，并返回到上一次调用setjmp的地方接着运行，并且本次setjmp的返回值为longjmp的第二个参数，本例为1.

## 如何实现setjmp
setjmp和goto有一些不同，setjmp是可以支持在函数间跳转的non-local jump，换言之，goto只能在函数内部的标签之间进行跳转。

不难看出，让longjmp返回到setjmp上一次调用位置，起到作用的应该是全局变量buf，即setjmp将调用函数的信息保存在buf中，而下一次buf作为参数传入longjmp时，longjmp会根据buf中的信息返回到上一次调用setjmp的位置。

## RVI的aosp中Bionic库setjmp.S实现
这里我们将setjmp.S分开阅读，首先看到两个函数入口，分别是_setjmp和setjmp：

    ENTRY(setjmp)
    __BIONIC_WEAK_ASM_FOR_NATIVE_BRIDGE(setjmp)
    li    a1, 1
    #ifdef __PIC__
    auipc    a2, 0
    /* FIXME:riscv */
    jalr    x0, 42(a2)
    #else
    j    sigsetjmp
    #endif
    END(setjmp)

    ENTRY(_setjmp)
    __BIONIC_WEAK_ASM_FOR_NATIVE_BRIDGE(_setjmp)
    li    a1, 0
    #ifdef __PIC__
    auipc    a2, 0
    jalr    x0, 18(a2)
    #else
    j    sigsetjmp
    #endif
    END(_setjmp)
可以看到这两个函数的不同点在于load immediate指令，a1寄存器即返回值的值分别为0，1，共同点在于非PIC时，会跳转到sigestjmp函数，那么我们接着看一下sigsetjmp。

    // int sigsetjmp(sigjmp_buf env, int save_signal_mask);
    ENTRY(sigsetjmp)
    __BIONIC_WEAK_ASM_FOR_NATIVE_BRIDGE(sigsetjmp)
    addi    sp, sp, -24
    sd    a0, 8(sp)
    sd    ra, 16(sp)

    mv    a0, a1
    #ifdef __PIC__
    call    __bionic_setjmp_cookie_get@plt
    #else
    j    __bionic_setjmp_cookie_get
    #endif
    mv    a1, a0
    ld      a0, 8(sp)
    sd    a1, 0(a0)
    andi    a1, a1, 1

    beqz    a1, 1f

    li    a1, 0
    addi    a2, a0, 8
    #ifdef __PIC__
    call    sigprocmask@plt
    #else
    j    sigprocmask
    #endif
    ld    a1, 0(sp)
    1:
    ld    a0, 8(sp)
    ld    ra, 16(sp)
    addi    sp, sp, 24

    sd    ra, 16(a0)
    sd    s0, 24(a0)
    sd    s1, 32(a0)
    sd    s2, 40(a0)
    sd    s3, 48(a0)
    sd    s4, 56(a0)
    sd    s5, 64(a0)
    sd    s6, 72(a0)
    sd    s7, 80(a0)
    sd    s8, 88(a0)
    sd    s9, 96(a0)
    sd    s10, 104(a0)
    sd    s11, 112(a0)
    sd    sp, 120(a0)

    fsd    fs0, 128(a0)
    fsd    fs1, 136(a0)
    fsd    fs2, 144(a0)
    fsd    fs3, 152(a0)
    fsd    fs4, 160(a0)
    fsd    fs5, 168(a0)
    fsd    fs6, 176(a0)
    fsd    fs7, 184(a0)
    fsd    fs8, 192(a0)
    fsd    fs9, 200(a0)
    fsd    fs10, 208(a0)
    fsd    fs11, 216(a0)

    li    a0, 0
    ret
    END(sigsetjmp)

sigsetjmp函数比较长，但是可以主要分为三部分，即riscv进行函数调用的prologue部分，一些信息的处理部分和最后信息将寄存器保存到内存中的部分（包括整型和浮点型）。

首先看第一部分就是riscv标准的函数调用prologue部分，即

    addi    sp, sp, -24
    sd    a0, 8(sp)
    sd    ra, 16(sp)
可以看到这里是将栈地址向高地址移动24（riscv是小端模式，栈移动一般以8为单位），并且将a0和ra压入栈中。

再看到下面很长遗传的sd和fsd指令，可以看出这部分setjmp的行为是将调用函数部分的信息写入到全局变量buf中以便下一次恢复。每一个sd或fsd都是将寄存器的值存入buf中，其中大部分寄存器中的内容对于我们可能并非多么重要，我们只需要重点关注三个寄存器的值：

    sd    ra, 16(a0)
    sd    s0, 24(a0)
    sd    sp, 120(a0)

可以看出将ra、s0、sp寄存器的值存入buf（a0寄存器保存了函数调用参数）中，并且stack pointer和frame pointer也都被存入buf中，以便下一次恢复到当前位置。

结束时将a0寄存器置0，原因是第一次调用setjmp时将会返回0。

longjmp的实现如下：

    // void siglongjmp(sigjmp_buf env, int value);
    ENTRY(siglongjmp)
    __BIONIC_WEAK_ASM_FOR_NATIVE_BRIDGE(siglongjmp)
    ld    a2, 0(a0)
    andi    a2, a2, 1
    beqz    a2, 1f

    addi    sp, sp, -16
    sd    a0, 0(sp)
    sd    ra, 8(sp)

    mv    t0, a1

    mv    a2, a0
    li    a0, 2
    addi    a1, a2, 8
    li    a2, 0
    #ifdef __PIC__
    call    sigprocmask@plt
    #else
    j    sigprocmask
    #endif
    mv    a1, t0

    ld    a0, 0(sp)
    ld    ra, 8(sp)
    addi    sp, sp, 16

    ld      a2, 0(a0)
    1:
    ld    ra, 16(a0)
    ld    s0, 24(a0)
    ld    s1, 32(a0)
    ld    s2, 40(a0)
    ld    s3, 48(a0)
    ld    s4, 56(a0)
    ld    s5, 64(a0)
    ld    s6, 72(a0)
    ld    s7, 80(a0)
    ld    s8, 88(a0)
    ld    s9, 96(a0)
    ld    s10, 104(a0)
    ld    s11, 112(a0)
    ld    sp, 120(a0)

    addi    sp, sp, -24
    sd    ra, 0(sp)
    sd    a0, 8(sp)
    sd    a1, 16(sp)
    ld    a0, 0(a0)
    #ifdef __PIC__
    call    __bionic_setjmp_cookie_check@plt
    #else
    jal    __bionic_setjmp_cookie_check
    #endif
    ld    ra, 0(sp)
    ld    a0, 8(sp)
    ld    a1, 16(sp)
    addi    sp, sp, 24

    fld    fs0, 128(a0)
    fld    fs1, 136(a0)
    fld    fs2, 144(a0)
    fld    fs3, 152(a0)
    fld    fs4, 160(a0)
    fld    fs5, 168(a0)
    fld    fs6, 176(a0)
    fld    fs7, 184(a0)
    fld    fs8, 192(a0)
    fld    fs9, 200(a0)
    fld    fs10, 208(a0)
    fld    fs11, 216(a0)

    // Set return value.
    beqz    a1, 2f
    li    a0, 1
    2:
    mv    a0, a1
    ret
    END(siglongjmp)
    ALIAS_SYMBOL(longjmp, siglongjmp)
    __BIONIC_WEAK_ASM_FOR_NATIVE_BRIDGE(longjmp)
    ALIAS_SYMBOL(_longjmp, siglongjmp)
    __BIONIC_WEAK_ASM_FOR_NATIVE_BRIDGE(_longjmp)

可以看出这里使用ALIAS_SYMBOL将longjmp全部当作siglongjmp处理。细看实现，可以发现，和setjmp类似，这里主要操作在于将buf中的值load回寄存器中，最后进行了判断，

    beqz    a1, 2f
    li    a0, 1
    2:
    mv    a0, a1
    ret
这里可以看到会判断a1是否等于0，如果是的话将跳转到2位置即将a1的值（第二个参数）当作返回值，否则返回值为1，也符合定义，然后将回到ra中指定的地址从而恢复buf。
