# 快速除法设计指导
## 反函数与自然对数
```
对 1 / x 定积分得到自然对数 ln(x)
也就是 x 倒数等于 {ln(x + deta) - ln(x)} / deta
为了方便运算，让 deta = 2^(-n)，除以 deta 就变成左移 n 位，而 x + deta 就变成给无穷小位 置一（x << n | 1，需要扩展精度）

对于 a / b 的除法运算可转换成 a * {ln(b + deta) - ln(b)} / deta
```

在 Better-Idea/Mix-C 项目中给出了 ln 对数函数的多级快速查表算法  
https://github.com/Better-Idea/Mix-C/blob/master/math/private/extern.ln.hpp

...

