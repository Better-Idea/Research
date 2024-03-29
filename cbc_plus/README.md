# 分组加密增强版(CBC PLUS)
## 背景
先介绍 ECB，ECB 加密方式比较简单，大致原理就是先将明文按指定长度分组（如 16 字节，不足 16 字节的部分填充 0 或按其他方式填充），
然后分别对每一组用相同的加密方式加密。
这么一说你可能就明白，相同的组加密后的结果也是相同的，而加密一些在运行过程中改动较小的明文，那么这种模式显得不那么安全。  
于是对于长的明文加密，安全工程师会推荐你使用 CBC 加密方式，在 CBC 模式下，明文组不再相互独立，而是依赖于前一个组的加密结果，
让前一个组的加密结果和当前组明文做异或之类的操作，然后再加密。而第一个组没有前一个组，所以它依赖与初始化向量 IV。  
**我们可以知道，只要 IV 不变，且明文前几个组的值不变，那么他们加密后的结果是不会变的。**  
而这里只要做一个小小的改变就能有不一样的变化，我们可以在第一个组中插入一个随机数，那么它后面所有的组即使明文内容一样，也会随着第一个组的变化而变化了。
我们甚至可以在其中任意位置插入存放随机数的组，这样可以在已知部分字段明文结构和内容的情况下，增加暴力破解的难度。  
- 把变动频繁的东西放到第一个组，如时间戳+随机数的组合，这样既可以防重放，又可以让后续的密文不断地变动
- 可以用一个组存放混淆组的位置信息，比如我们可以让第二个组记录混淆组的位置信息，每一个 1bit 表示从该组以后的组那些是混淆组
