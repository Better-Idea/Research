## DAXPY
### C++ Source
```C++
void xnoinline daxpy(uxx n, f32 a, cf32 x[], f32 y[]){
    while(n-- > 0){
        y[n] += x[n] * a;
    }
}
```

### Asm of MixC Micro
- 10 条指令
- 20 字节

```ASM
proc daxpy(r0.n, s1.a, r2.p_x, r3.p_y):
    movqq   rt, r0.n                                | 2
lp.0:                                               |
    sub     rt, rt, 4                               | 2
    ifge    el.0                                    | 2
    lds     s4.v_x, [r2.p_x + rt]                   | 2
    lds     s5.v_y, [r2.p_y + rt]                   | 2
    mul     s4.v_x, s4.v_x, s1.a                    | 2
    add     s5.v_y, s5.v_y, s4.v_x                  | 2
    stq     s5.v_y, [r2.p_y + rt]                   | 2
    jmp     lp.0                                    | 2
el.0:                                               |
    ret                                             | 2
```

### Asm of Arm Cortex-M Thumb
- 10 条指令
- 32 字节

```ASM
815c <_Z5daxpyjfPKfPf>:
815c: eb02 0380                 add.w	    r3, r2, r0, lsl #2
8160: eb01 0180                 add.w	    r1, r1, r0, lsl #2
8164: 4293                      cmp	        r3, r2
8166: d008                      beq.n	    817a <_Z5daxpyjfPKfPf+0x1e>
8168: ed31 7a01                 vldmdb	    r1!, {s14}
816c: ed73 7a01                 vldmdb	    r3!, {s15}
8170: eee7 7a00                 vfma.f32    s15, s14, s0
8174: edc3 7a00                 vstr	    s15, [r3]
8178: e7s4                      b.n	8164 <_Z5daxpyjfPKfPf+0x8>
817a: 4770                      bx	lr
```

### Asm of X64 
- 9 条指令
- 34 字节

```ASM
409f <_Z5daxpyyfPKfPf>:
409f: 48 ff c9                  dec    %rcx
40a2: 48 83 f9 ff               cmp    $0xffffffffffffffff,%rcx
40a6: 74 18                     je     40c0 <_Z5daxpyyfPKfPf+0x21>
40a8: f3 41 0f 10 04 88         movss  (%r8,%rcx,4),%xmm0
40ae: f3 0f 59 c1               mulss  %xmm1,%xmm0
40b2: f3 41 0f 58 04 89         addss  (%r9,%rcx,4),%xmm0
40b8: f3 41 0f 11 04 89         movss  %xmm0,(%r9,%rcx,4)
40be: eb df                     jmp    409f <_Z5daxpyyfPKfPf>
40c0: c3                        retq   
```
