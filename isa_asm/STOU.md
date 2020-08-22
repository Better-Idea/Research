## STOU
### C++ Source
```C++
u64 xnoinline stou(asciis str, uxx len){
    auto is_neg = true;
    auto end    = str + len;
    auto r      = u64(0);

    if (str[0] == '+'){
        str    += 1;
    }
    else if (str[0] == '-'){
        str    += 1;
        is_neg  = false;
    }

    while(str < end){
        r       = r * 10 + (str[0] & 0xf);
        str    += 1;
    }
    if (is_neg){
        r       = u64(0) - r;
    }
    return r;
}
```

### Asm of MixC Micro
- 20 条指令
- 44 字节
```ASM
proc stou(r1 str, r2 len):
    bdcqi   r0, r3, 0           | 2
    add     r2, r2, r0          | 2
    ldb     rt, [r1]            | 2
    cmpr    rt, '-', '+'        | 4
    ifnel   if.0                | 2
    ifner   if.1                | 2
    jmp     if.2                | 2
if.0:                           |
    movqi   r3, 1               | 2
if.1:                           |
    add     r1, 1               | 2
if.2:                           |
lp.0:                           |
    cmp     r1, r2              | 2
    ifne    lp.0                | 2
    ldb     rt, [r1]            | 2
    and     rt, rt, 0xf         | 2
    add     r0, rt, r0 * 10     | 4
    add     r1, r1, 1           | 2
    jmp     lp.0                | 2
el.0:                           |
    cmp     r3, 0               | 2
    ifne    if.3                | 2
    sub     r0, 0, r0           | 2
if.3:                           |
    ret                         | 2

优化笔记：
- 当有多个对象赋值同一个值时，可以考虑用 bdc 系列广播赋值指令
- 当与多个值比较时，可以用 cmpr 范围比较指令
```

### Asm of Arm Cortex-M Thumb
- 30 条指令
- 74 字节
```ASM
8110 <_Z4stouPKcj>:
8110: b5f0                      push {r4, r5, r6, r7, lr}
8112: 7802                      ldrb r2, [r0, #0]
8114: 4603                      mov r3, r0
8116: 2a2b                      cmp r2, #43 ; 0x2b
8118: eb00 0701                 add.w r7, r0, r1
811c: d102                      bne.n 8124 <_Z4stouPKcj+0x14>
811e: 3301                      adds r3, #1
8120: 2201                      movs r2, #1
8122: e003                      b.n 812c <_Z4stouPKcj+0x1c>
8124: 2a2d                      cmp r2, #45 ; 0x2d
8126: d1fb                      bne.n 8120 <_Z4stouPKcj+0x10>
8128: 2200                      movs r2, #0
812a: 3301                      adds r3, #1
812c: 2000                      movs r0, #0
812e: 2100                      movs r1, #0
8130: f04f 0c0a                 mov.w ip, #10
8134: 42bb                      cmp r3, r7
8136: d20b                      bcs.n 8150 <_Z4stouPKcj+0x40>
8138: fba0 450c                 umull r4, r5, r0, ip
813c: f813 6b01                 ldrb.w r6, [r3], #1
8140: fb0c 5501                 mla r5, ip, r1, r5
8144: f006 060f                 and.w r6, r6, #15
8148: 19a0                      adds r0, r4, r6
814a: f145 0100                 adc.w r1, r5, #0
814e: e7f1                      b.n 8134 <_Z4stouPKcj+0x24>
8150: b112                      cbz r2, 8158 <_Z4stouPKcj+0x48>
8152: 4240                      negs r0, r0
8154: eb61 0141                 sbc.w r1, r1, r1, lsl #1
8158: bdf0                      pop {r4, r5, r6, r7, pc}

```

### Asm of X64 
- 25 条指令
- 63 字节
```ASM
1560:	8a 01                   mov    (%rcx),%al
1562:	48 01 ca                add    %rcx,%rdx
1565:	3c 2b                   cmp    $0x2b,%al
1567:	75 08                   jne    401571 <_Z4stouPKcy+0x11>
1569:	48 ff c1                inc    %rcx
156c:	41 b1 01                mov    $0x1,%r9b
156f:	eb 0d                   jmp    40157e <_Z4stouPKcy+0x1e>
1571:	41 b1 01                mov    $0x1,%r9b
1574:	3c 2d                   cmp    $0x2d,%al
1576:	75 06                   jne    40157e <_Z4stouPKcy+0x1e>
1578:	48 ff c1                inc    %rcx
157b:	45 31 c9                xor    %r9d,%r9d
157e:	31 c0                   xor    %eax,%eax
1580:	48 39 d1                cmp    %rdx,%rcx
1583:	73 11                   jae    401596 <_Z4stouPKcy+0x36>
1585:	4c 6b c0 0a             imul   $0xa,%rax,%r8
1589:	8a 01                   mov    (%rcx),%al
158b:	48 ff c1                inc    %rcx
158e:	83 e0 0f                and    $0xf,%eax
1591:	4c 01 c0                add    %r8,%rax
1594:	eb ea                   jmp    401580 <_Z4stouPKcy+0x20>
1596:	45 84 c9                test   %r9b,%r9b
1599:	74 03                   je     40159e <_Z4stouPKcy+0x3e>
159b:	48 f7 d8                neg    %rax
159e:	c3                      retq   
```

