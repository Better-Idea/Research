# Reduced-ManchesterEncoding
在阅读计算机网络时偶然发现神奇的曼彻斯特编码格式，该编码可以在没有同步时钟的情况下传输数据，虽然 uart 也可以这么做，但使用曼彻斯特编码的方式传送数据具有更好的稳定性，同时也能跑到更高的频率。  

我们结合差分曼彻斯特编码，并做出以下定义：若当前传输的位与前一位相同就反转电平，否则保持电平并插入一个反转电平。  
假设每一位为 0 和 1 的概率随机
- 50% 的概率存在 100% 编码率(只用一位就能表示一个二进制)。
- 50% 的概率存在 50% 编码率(要用两位表示一个二进制)。
- 75% = 50% * 100% + 50% * 50%

### Verilog 参考代码

```verilog
module mserial_rx(
    input               reset,
    input               sampling,
    input               ipt,
    output  reg [7:0]   byte        = 1,
    output  reg         finish      = 0
)
    reg                 ignore_once = 1;
    reg                 old_bit     = 1;
    reg                 old_ipt     = 0;
always @ (negedge reset or posedge sampling) begin
    if (reset == 0) begin
        byte            = 1;
        finish          = 0;
        i               = 0;
        ignore_once     = 1;
        old_bit         = 1;
        old_ipt         = 0;
    end else if (ignore_once) begin
        ignore_once     = 0;
        old_ipt         = ipt;
    end else begin
        ignore_once     = ipt == old_ipt;
        old_bit         = ipt == old_ipt ? ~old_bit : old_bit;
        old_ipt         = ipt;
        {finish, byte}  = {byte, old_bit};
    end
end

module mserial_tx(
    input               reset,
    input               clock,
    input       [7:0]   byte,
    output  reg         opt         = 0,
    output  reg         finish      = 0
)
    reg         old_bit             = 0;
    reg         old_opt             = 0;
    reg  [2:0]  i                   = 0;
always @ (negedge reset or posedge clock) begin
    if (reset == 0) begin
        opt         = 0;
        finish      = 0;
        old_bit     = 0;
        old_opt     = 0;
        i           = 0;
    end else if (opt == old_opt) begin
        opt         = ~old_opt;
    end else begin
        old_opt     = opt;

        if (byte[i] == old_bit) begin
            opt     = ~opt;
        end else begin
            old_bit = byte[i];
        end
        
        i          += 1;
        finish      = i == 0;
    end
end
```
