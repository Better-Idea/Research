## BUBBLE_SORT
### C++ Source
```C++
void xnoinline bubble_sort(const int * ary, int len){
    for(int i = 0; i < len; i++){
        for(int j = 1, max_i = 0; ; j++){
            if (j == len - i){
                auto t      = ary[j - 1];
                ary[j - 1]  = ary[max_i];
                ary[max_i]  = t;
                break;
            }
            if (ary[max_i] < ary[j]){
                max_i = j;
            }
        }
    }
}
```

### Asm of MixC Micro
- 21 条指令
- 42 字节

```ASM
proc bubble_sort(r0.ary, r1.len):
    sft         rt, r1.len, 2                           | 2
    add         r1.p_end_0, r0.ary, rt                  | 2
lp.0:                                                   |
    cmp         r1.p_end_0, r0.ary                      | 2
    ifgt        el.0                                    | 2
    bdcqq       r4.p_new_j, r5.p_max_i, r0.ary          | 2
    lddx        r6.v_old_j, r7.v_max_i, [r0.ary]        | 2
lp.1:                                                   |
    movqq       r2.p_old_j, r4.p_new_j                  | 2
    add         r4.p_new_j, r4.p_new_j, 4               | 2
    cmp         r4.p_new_j, r1.p_end_0                  | 2
    ifeq        if.0                                    | 2
    std         r6.v_old_j, [r5.p_max_i]                | 2
    std         r7.v_max_i, [r2.p_old_j]                | 2
    sub         r1.p_end_0, r1.p_end_0, 4               | 2
    jmp         lp.0                                    | 2
if.0:                                                   |
    lddx        r6.v_old_j, r2.v_new_j, [r4.p_new_j]    | 2
    cmp         r7.v_max_i, r2.v_new_j                  | 2
    iflt        if.1                                    | 2
    movqqx      r7.v_max_i, r2.v_new_j                  | 2
    movqq       r5.p_max_i, r4.p_new_j                  | 2
if.1:                                                   |
    jmp         lp.1                                    | 2
el.0:                                                   |
    ret                                                 | 2

优化笔记：
- 将数组长度变换成尾地址
- 先用符号代替寄存器号，在排列完毕后为其分配合适的寄存器

申请：
- ldx 广播式加载内存
- ifx 融合 cmp 的超短范围跳转
```

### Asm of Arm Cortex-M Thumb
- 24 条指令
- 60 字节

```ASM
8152 <_Z11bubble_sortPii>:
8152: b5f0                      push {r4, r5, r6, r7, lr}
8154: 460c                      mov r4, r1
8156: eb00 0581                 add.w r5, r0, r1, lsl #2
815a: 1b0b                      subs r3, r1, r4
815c: 428b                      cmp r3, r1
815e: da15                      bge.n 818c <_Z11bubble_sortPii+0x3a>
8160: 2200                      movs r2, #0
8162: 2301                      movs r3, #1
8164: 42a3                      cmp r3, r4
8166: f850 6022                 ldr.w r6, [r0, r2, lsl #2]
816a: eb00 0782                 add.w r7, r0, r2, lsl #2
816e: d106                      bne.n 817e <_Z11bubble_sortPii+0x2c>
8170: f855 3c04                 ldr.w r3, [r5, #-4]
8174: 3c01                      subs r4, #1
8176: f845 6d04                 str.w r6, [r5, #-4]!
817a: 603b                      str r3, [r7, #0]
817c: e7ed                      b.n 815a <_Z11bubble_sortPii+0x8>
817e: f850 7023                 ldr.w r7, [r0, r3, lsl #2]
8182: 42b7                      cmp r7, r6
8184: bfb8                      it lt
8186: 461a                      movlt r2, r3
8188: 3301                      adds r3, #1
818a: e7eb                      b.n 8164 <_Z11bubble_sortPii+0x12>
818c: bdf0                      pop {r4, r5, r6, r7, pc}
```

### Asm of X64 
- 24 条指令
- 73 字节

```ASM
159f: 89 d0                     mov    %edx,%eax
15a1: 41 89 d0                  mov    %edx,%r8d
15a4: 41 29 c0                  sub    %eax,%r8d
15a7: 41 39 d0                  cmp    %edx,%r8d
15aa: 7d 3b                     jge    4015e7 <_Z11bubble_sortPii+0x48>
15ac: 41 b8 01 00 00 00         mov    $0x1,%r8d
15b2: 45 31 c9                  xor    %r9d,%r9d
15b5: 4d 63 d1                  movslq %r9d,%r10
15b8: 4e 8d 1c 91               lea    (%rcx,%r10,4),%r11
15bc: 45 8b 13                  mov    (%r11),%r10d
15bf: 44 39 c0                  cmp    %r8d,%eax
15c2: 75 16                     jne    4015da <_Z11bubble_sortPii+0x3b>
15c4: 44 8d 40 ff               lea    -0x1(%rax),%r8d
15c8: 4c 89 c0                  mov    %r8,%rax
15cb: 4e 8d 04 81               lea    (%rcx,%r8,4),%r8
15cf: 45 8b 08                  mov    (%r8),%r9d
15d2: 45 89 10                  mov    %r10d,(%r8)
15d5: 45 89 0b                  mov    %r9d,(%r11)
15d8: eb c7                     jmp    4015a1 <_Z11bubble_sortPii+0x2>
15da: 46 39 14 81               cmp    %r10d,(%rcx,%r8,4)
15de: 45 0f 4c c8               cmovl  %r8d,%r9d
15e2: 49 ff c0                  inc    %r8
15e5: eb ce                     jmp    4015b5 <_Z11bubble_sortPii+0x16>
15e7: c3                        retq   
```
