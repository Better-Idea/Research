## INSERT_SORT
### C++ Source
```C++
void xnoinline insert_sort(uxx * ary, uxx len){
    for(uxx i = 1, j; i < len; i++){
        uxx x = ary[i];
        for(j = i; j > 0 and ary[j - 1] > x; j--){
            ary[j] = ary[j - 1];
        }
        ary[j] = x;
    }
}
```

### Asm of MixC Micro
- 16 条指令
- 32 字节

```ASM
proc insert_sort(r0.ary, r1.len):
    bdcqq       r4.p_i, r5.p_end, r0.ary            | 2
    add         r5.p_end, r5.p_end, r1.len          | 2
lp.0:                                               | 2
    add         r4.p_i, r4.p_i, 8                   | 2
    bdcqq       r1.p_new_j, r2.p_old_j, r4.p_i      | 2
    ciflt       r4.p_i, r5.p_end, el.0              | 2
    ldq         r7.v_v, [r4.p_i]                    | 2
lp.1:                                               | 2
    sub         r1.p_new_j, r1.p_new_j, 8           | 2
    cifgt       r2.p_old_j, r0.ary, el.1            | 2
    ldq         r6.v_new_j, [r1.p_new_j]            | 2
    cifgt       r6.v_new_j, r7.v_v, el.1            | 2
    stq         r6.v_new_j, [r2.p_old_j]            | 2
    movqq       r2.p_old_j, r1.p_new_j              | 2
    jmp         lp.1                                | 2
el.1:                                               | 2
    stq         r7.v_v, [r2.p_old_j]                | 2
    jmp         lp.0                                | 2
el.0:                                               | 2
    ret                                             | 2

申请：
- cifgt/cifge/ciflt/cifle/cifeq/cifne 融合比较转移指令支持 16 + 1 短距离向下跳转
- add/sub 指令中的立即数无需符号位扩展
```

### Asm of Arm Cortex-M Thumb
- 18 条指令
- 44 字节

```ASM
8120 <_Z11insert_sortPii>:
8120: b5f0                      push	{r4, r5, r6, r7, lr}
8122: 2301                      movs	r3, #1
8124: 4604                      mov	r4, r0
8126: 428b                      cmp	r3, r1
8128: da0f                      bge.n	814a <_Z11insert_sortPii+0x2a>
812a: f854 6f04                 ldr.w	r6, [r4, #4]!
812e: 461a                      mov	r2, r3
8130: 4625                      mov	r5, r4
8132: f855 7c04                 ldr.w	r7, [r5, #-4]
8136: 42b7                      cmp	r7, r6
8138: dd03                      ble.n	8142 <_Z11insert_sortPii+0x22>
813a: 3a01                      subs	r2, #1
813c: f845 7904                 str.w	r7, [r5], #-4
8140: d1f7                      bne.n	8132 <_Z11insert_sortPii+0x12>
8142: f840 6022                 str.w	r6, [r0, r2, lsl #2]
8146: 3301                      adds	r3, #1
8148: e7ed                      b.n	8126 <_Z11insert_sortPii+0x6>
814a: bdf0                      pop	{r4, r5, r6, r7, pc}
```

### Asm of X64 
- 15 条指令
- 47 字节

```ASM
4070 <_Z11insert_sortPyy>:
4070: 41 b8 01 00 00 00         mov    $0x1,%r8d
4076: 49 39 d0                  cmp    %rdx,%r8
4079: 73 23                     jae    40409e <_Z11insert_sortPyy+0x2e>
407b: 4e 8b 0c c1               mov    (%rcx,%r8,8),%r9
407f: 4c 89 c0                  mov    %r8,%rax
4082: 4c 8b 54 c1 f8            mov    -0x8(%rcx,%rax,8),%r10
4087: 4d 39 ca                  cmp    %r9,%r10
408a: 76 09                     jbe    404095 <_Z11insert_sortPyy+0x25>
408c: 4c 89 14 c1               mov    %r10,(%rcx,%rax,8)
4090: 48 ff c8                  dec    %rax
4093: 75 ed                     jne    404082 <_Z11insert_sortPyy+0x12>
4095: 4c 89 0c c1               mov    %r9,(%rcx,%rax,8)
4099: 49 ff c0                  inc    %r8
409c: eb d8                     jmp    404076 <_Z11insert_sortPyy+0x6>
409e: c3                        retq   
```
