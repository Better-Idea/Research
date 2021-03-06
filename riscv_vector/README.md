# RISCV_VECTOR
## 概述
**本次研究的对象是 RISCV 的向量指令**  
RISCV 的向量指令将 SIMD 最大处理元素个数从统一编码中分离出来，使得无需为不同数据位宽处理的指令单独编码。  
类似于先给寄存器定义类型，然后后续该寄存器执行操作时按先前给定的类型运算。  

## 优势
- 减少了编码空间的占用
- 降低了代码体积

## 问题
据官方文档上描述， RISCV 向量指令的使用可以大幅降低数据并行处理的循环次数。但实际上真有这种想象中那么好吗？  

粗暴的堆砌硬件可能会适得其反。  

#### C++ 样本
```C++
void xnoinline daxpy(uxx n, f64 a, cf64 x[], f64 y[]){
    while(n-- > 0){
        y[n] += x[n] * a;
    }
}
```

给定关键硬件参数配置：
- CPU：3GHz
- MEM：3200MHz * 4Channel * 64Bit
- MVL：64（RISCV 向量寄存器长度）

按上述配置，内存极限带宽为 102.4GB/s

#### RISCV 汇编
```
                                        | 理想耗时（时钟周期）
    li          t0, 2 << 25             | 01
    vsetdcfg    t0                      | 01 # 使能 f64 向量运算寄存器
lp.0:                                   | 
    setvl       t0, a0                  | 01 # 设置向量长度
    vld         v0, a1                  | 15 # 加载内存 64 x 8Byte x 3GHz -> 1536GB/s
                                        |    # 最少需要 1536/102.4 -> 15 个周期才能读取 64 x 8Byte 数据
    slli        t1, t0, 3               | 00 # 折叠
    vld         v1, a2                  | 15
    add         a1, a1, t1              | 00 # 折叠
    vfmadd      v1, v0, fa0, v1         | 05 # 融合乘加
    sub         a0, a0, t0              | 00 # 折叠
    vst         v1, a2                  | 15
    add         a2, a2, t1              | 00 # 折叠
    bnez        a0, lp.0                | 01 
    ret                                 | 

    # 消耗 64 个 f64 的 ALU 换来大约 15 x 3 + 5 -> 50 个时钟完成 64@f64 次运算
```

#### 流水线的折叠
```
    # 这里为了观察流水线直接将循环展开
    load        x0                      | 2 # 8 x 8Byte x 3GHz -> 192GB/s 
    load        y0                      | 2 # 每个 load/store 耗费 2 个时钟

    fma         y0, x0, a               | 5
    load        x1                      | 0 # 折叠
    load        y1                      | 0 # 折叠
                                        |   # 还可以折叠 1 时钟
    store       y0                      | 1 # 为下条指令折叠 1 个时钟
    fma         y1, x1, a               | 4 # 5 - 1，为下条指令折叠 4 个时钟
    load        x2                      | 0
    load        y2                      | 0

    # 从此处之后每批运算需要 6 个时钟
    store       y1                      | 2 # 为下条指令折叠 2 个时钟
    fma         y2, x2, a               | 3 # 5 - 2，为下条指令折叠 3 个时钟
    load        x3                      | 0
    load        y3                      | 1

    store       y2                      | 2
    fma         y3, x3, a               | 3 # 5 - 2，为下条指令折叠 3 个时钟
    load        x4                      | 0
    load        y4                      | 1
    ...

    # 消耗 8 个 f64 的 ALU 换来大约 6 * 8 -> 48 个时钟完成 64@f64 次运算
```

## 建议
- MVL 需要根据内存带宽动态配置
- ALU 受限于内存带宽，根据最大内存带宽可以算出实际需要的 ALU 个数

## 结论
- RISCV 的向量指令还是嫩了点
- 我们完全有理由期待超高带宽和更低延迟的内存
