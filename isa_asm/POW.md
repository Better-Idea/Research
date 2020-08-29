## POW
### C++ Source
```C++
template<class type>
xnoinline type pow(type a, uxx x){
    type r       = 1;

    for(; x != 0; x >>= 1, a *= a){
        if (x & 1){
            r *= a;
        }
    }
    return r;
}
```

### Asm of MixC Micro
- 7 条指令
- 14 字节

#### pow.f32
```ASM
proc pow(s1.a, r2.x):
    movsi       s0.r, 1                             | 2
lp.0:                                               |
    shr         r2.x, r2.x, 1                       | 2
    ifnz        el.0:                               | 2
    ifcf        if.0:                               | 2
    mul         s0.r, s0.r, s1.a                    | 2
if.0:                                               |
    mul         s1.a, s1.a, s1.a                    | 2
el.0:                                               |
    ret                                             | 2
```

### Asm of Arm Cortex-M Thumb
#### pow.f32
- 10 条指令
- 30 字节
```ASM
8110 <_Z3powfj>:
8110: eef7 7a00             vmov.f32	s15, #112	; 0x3f800000  1.0
8114: b140                  cbz	r0, 8128 <_Z3powfj+0x18>
8116: 07c3                  lsls	r3, r0, #31
8118: ea4f 0050             mov.w	r0, r0, lsr #1
811c: bf48                  it	mi
811e: ee67 7a80             vmulmi.f32	s15, s15, s0
8122: ee20 0a00             vmul.f32	s0, s0, s0
8126: e7f5                  b.n	8114 <_Z3powfj+0x4>
8128: eeb0 0a67             vmov.f32	s0, s15
812c: 4770                  bx	lr
```

#### pow.uxx
- 10 条指令
- 22 字节
```ASM
8108 <_Z3powjj>:
8108: 2301                  movs	r3, #1
810a: b131                  cbz	r1, 811a <_Z3powjj+0x12>
810c: 07ca                  lsls	r2, r1, #31
810e: ea4f 0151             mov.w	r1, r1, lsr #1
8112: bf48                  it	mi
8114: 4343                  mulmi	r3, r0
8116: 4340                  muls	r0, r0
8118: e7f7                  b.n	810a <_Z3powjj+0x2>
811a: 4618                  mov	r0, r3
811c: 4770                  bx	lr
```

### Asm of X64 
#### pow.f32
- 11 条指令
- 35 字节
```ASM
4070 <_Z3powfy>:
4070: f3 0f 10 0d 88 d8 01  movss  0x1d888(%rip),%xmm1        # 421900 <.rdata>
4077: 00 
4078: 48 85 d2              test   %rdx,%rdx
407b: 74 12                 je     408f <_Z3powfy+0x1f>
407d: f6 c2 01              test   $0x1,%dl
4080: 74 04                 je     4086 <_Z3powfy+0x16>
4082: f3 0f 59 c8           mulss  %xmm0,%xmm1
4086: f3 0f 59 c0           mulss  %xmm0,%xmm0
408a: 48 d1 ea              shr    %rdx
408d: eb e9                 jmp    4078 <_Z3powfy+0x8>
408f: 0f 28 c1              movaps %xmm1,%xmm0
4092: c3                    retq   
```

#### pow.uxx
- 10 条指令
- 29 字节
```ASM
4070 <_Z3powfy>:
4070: b8 01 00 00 00        mov    $0x1,%eax
4075: 48 85 d2              test   %rdx,%rdx
4078: 74 12                 je     40408c <_Z3powyy+0x1c>
407a: f6 c2 01              test   $0x1,%dl
407d: 74 04                 je     404083 <_Z3powyy+0x13>
407f: 48 0f af c1           imul   %rcx,%rax
4083: 48 0f af c9           imul   %rcx,%rcx
4087: 48 d1 ea              shr    %rdx
408a: eb e9                 jmp    404075 <_Z3powyy+0x5>
408c: c3                    retq   
```