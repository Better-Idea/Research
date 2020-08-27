## BINARY_SEARCH
### C++ Source
```C++
int xnoinline binary_search(int * seq, int len, int const & v){
    int left   = 0;
    int center = len >> 1;
    int right  = len - 1;

    for (; left <= right; center = (left + right) >> 1) {
        if (seq[center] > v) {
            right = center - 1;
        }
        else if (seq[center] < v) {
            left = center + 1;
        }
        else {
            return center;
        }
    }
    return -1;
}
```

### Asm of MixC Micro
- 20 条指令
- 42 字节
```ASM
proc binary_search(r1.seq, r2.len, r3.ref_v):
    lddx        r5.v_v, [r3.ref_v]              | 2
    movqix      r3.i_left, 0                    | 2
    movqqx      r0.i_center, r2.len             | 2
    sft         r0.i_center, r0.i_center, 1     | 2
    sub         r2.i_right, r2.i_right, 1       | 2
lp.0:                                           | 2
    ifle        r3.i_left, r2.i_right, lp.0     | 2
    add         rt, r1.seq, r0.i_center * 4     | 4
    lddx        v_center, [rt]                  | 2
    ifgt        v_center, v_v, if.0             | 2
    movqq       r2.i_right, r0.i_center         | 2
    sub         r2.i_right, r2.i_right, 1       | 2
    jmp         if.1                            | 2
if.0:                                           | 2
    iflt        if.1                            | 2
    movqqx      r3.i_left, r0.i_center          | 2
    add         r3.i_left, r3.i_left, 1         | 2
if.1:                                           | 2
    add         rt, r3.i_left, r2.i_right       | 2
    sft         r0.i_center, rt, 1              | 2
    jmp         lp.0                            | 2
el.0:                                           | 2
    movqix      r0.i_center, -1                 | 2
ei.0:                                           | 2
    ret                                         | 2


申请：
- 精简分支指令
    ifxx if.0:
        ifxx if.0.1:
    if.0.1: -+
if.0: -------+-- 合并，并且下一条 eixx 属于第 0 个 if，每个 ifxx 都有独立的 sta 状态位组
    eixx if.1
if.1

适合
```

### Asm of Arm Cortex-M Thumb
- 20 条指令
- 44 字节
```ASM
8128 <binary_search>:
8128: b570                      push	{r4, r5, r6, lr}
812a: 2400                      movs	r4, #0
812c: 4605                      mov	r5, r0
812e: 1048                      asrs	r0, r1, #1
8130: 3901                      subs	r1, #1
8132: 428c                      cmp	r4, r1
8134: dc0b                      bgt.n	814e <binary_search+0x26>
8136: f855 6020                 ldr.w	r6, [r5, r0, lsl #2]
813a: 6813                      ldr	r3, [r2, #0]
813c: 429e                      cmp	r6, r3
813e: dd03                      ble.n	8148 <binary_search+0x20>
8140: 1e41                      subs	r1, r0, #1
8142: 1863                      adds	r3, r4, r1
8144: 1058                      asrs	r0, r3, #1
8146: e7f4                      b.n	8132 <binary_search+0xa>
8148: da03                      bge.n	8152 <binary_search+0x2a>
814a: 1c44                      adds	r4, r0, #1
814c: e7f9                      b.n	8142 <binary_search+0x1a>
814e: f04f 30ff                 mov.w	r0, #4294967295
8152: bd70                      pop	{r4, r5, r6, pc}
```

### Asm of X64 
- 19 条指令
- 49 字节
```ASM
4070 <binary_search>:
4070: 45 31 c9                  xor    %r9d,%r9d
4073: 89 d0                     mov    %edx,%eax
4075: ff ca                     dec    %edx
4077: d1 f8                     sar    %eax
4079: 41 39 d1                  cmp    %edx,%r9d
407c: 7f 1f                     jg     40409d <binary_search+0x2d>
407e: 4c 63 d0                  movslq %eax,%r10
4081: 45 8b 18                  mov    (%r8),%r11d
4084: 46 39 1c 91               cmp    %r11d,(%rcx,%r10,4)
4088: 7e 05                     jle    40408f <binary_search+0x1f>
408a: 8d 50 ff                  lea    -0x1(%rax),%edx
408d: eb 06                     jmp    404095 <binary_search+0x25>
408f: 7d 0f                     jge    4040a0 <binary_search+0x30>
4091: 44 8d 48 01               lea    0x1(%rax),%r9d
4095: 41 8d 04 11               lea    (%r9,%rdx,1),%eax
4099: d1 f8                     sar    %eax
409b: eb dc                     jmp    404079 <binary_search+0x9>
409d: 83 c8 ff                  or     $0xffffffff,%eax
40a0: c3                        retq   
```
