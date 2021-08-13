# 基于DGHV的同态加密
基本概念、学习笔记、参考内容：

> 概念、个人笔记：
>
> 1. [同态加密的原理详解与go实践](https://blog.csdn.net/weixin_43988498/article/details/118802616)
> 2. [DGHV:整数上的同态加密(1)-算法构建](https://blog.csdn.net/weixin_43988498/article/details/119459857)
> 3. [DGHV:整数上的同态加密(2)-解决噪声与构建全同态蓝图](https://blog.csdn.net/weixin_43988498/article/details/119570507)
>
> 参考：
>
> 1. 论文：[《一种基于智能合约的全同态加密方法》](https://kns.cnki.net/kcms/detail/detail.aspx?dbcode=CJFD&dbname=CJFDLAST2020&filename=AQJS202009005&v=kzaijASy61Kjw7dUtnNnML6O%25mmd2Bv886ZZ4Mq9RnlqNape%25mmd2BABO%25mmd2Bfioot2MYlYfxTEcj)
> 2. [1] M. Dijk, C. Gentry, S. Halevi, and V.Vaikuntanathan. Fully homomorphic encryption over the integers[J]. Applications of Cryptographic Techniques: Springer, Berlin, 2010, 24-43.
> 3. http://blog.sciencenet.cn/blog-411071-617182.html

> <font color='#e54d42'>**同态加密等计算量大的算法不适合在合约中计算，合约仅作为测试**</font>

> 本项目代码地址: https://github.com/xwjahahahaha/DGHV

# 基本设计

## 1. 随机数

我们所说的随机函数都是伪随机函数即PRF

随机函数的一般构成是：`随机种子 +  随机数生成算法`

目前有很多优秀的伪随机算法已经实现，但是在区块链智能合约上的最大困难是**区块链的封闭性**

可以将区块链看作一个封闭式的信息世界，所以不像一般网络中有丰富的熵增源.

Solidity通常采用keccak256哈希函数 作为随机数的生成器，该函数有一定的随机数性质，但是随机数生成的过程容易被攻击。

传统的随机数生成过程需要本结点的 Nonce值作为随机数种子，恶意节点会大量计算Nonce的值，直到随机事件的结果对自己有利，所以项目采用区块时间戳作为随机种子。

使用线性求余法生成随机数，再采用keccak256 Hash函数将区块时间戳与随机数合并取最终的随机数

生成公式如下：
$$
\begin{cases} X_{n+1} = (aX_n+c)\  mod\  m, \ n \geq 0 \\ R_{n+1} = keccak256(x_{n+1} \ + \ Block\_TimeStamp) \ mod \ k \end{cases}\begin{cases} X_{n+1} = (aX_n+c)\  mod\  m, \ n \geq 0 \\ R_{n+1} = keccak256(x_{n+1} \ + \ Block\_TimeStamp) \ mod \ k \end{cases}
$$

## 2. 整数上的全同态加密

定义一套**对称加密**方法：

$keyGen(\lambda)$根据安全参数$\lambda$生成一个**大奇数$p$**密钥, $\eta$（bit）是生成密钥$p$的位数

$Encrypto(pk, m)$​表示加密,其中$pk$​表示公钥, m是明文，根据Dijk中的规定$m \in \{0,1\}$​也即明文m只有一位;

 $r, q$​都是正随机数, 长度分别为$\rho、 \gamma$​​ , 其中的要求是$q>p$​ 且 $q$是公开的 , $r$​是一个随机小整数（可为负数）

则加密过程为:
$$
Encrypto(pk, m) = m + 2r + pq
$$
对应的解密过程$Decrypto(sk, c)$, 其中$sk$表示私钥、$c$表示密文:
$$
Decrypto(sk, c) = (c \ mod  \ p) \ mod \ 2 = （c - p* \ulcorner\dfrac{c}{p}\lrcorner）mod \ 2 = Lsb(c) \ XOR \ Lsb(\ulcorner\dfrac{c}{p}\lrcorner)
$$

这里是对称加密，所以公钥和私钥是相同的，即$pk =sk = p$

**正确性的验证：**

对于解密算法显然可以看出$mod \ p$​将$pq$项消除，$mod2$将随机数$r$项消除，最后的结果就是$m$

**安全性讨论：**

论文中已说明当参数$r \approx 2 ^{\sqrt{\eta}} , q \approx 2^{\eta^3}$​时，该方案是安全的

# 功能测试

合约实现了输入为单Bit（即$m \in \{0, 1\}$​）的**加法同态加密**（使用对称秘钥）

## step_1 选择参数

编译、运行`syn_DGHV.go`

对于参数η，加法同态始终满足，但是乘法同态满足有要求（因为算法噪音）：

* 经测试η>=9时，乘法同态满足（小于9时不稳定，可见评估结果输出）
* 智能合约中3<=η<=5, 因为η过大会导致参数q过大无法部署合约（solidity最大为int256，没有大数操作）

```shell
go build -o dghv.exe ./ && chmod +x dghv.exe
./dghv.exe 5		# 参数1：η （建议>=9, 合约中<=5）
```

运行结果中包含：

* 生成的秘钥`p`
* 参数`q`

输出示例（输出包含了多组测试，选择一组参数即可）：

```shell
==============================================
p =  31
q = 42535295865117307932921825928971026418
m0 = 0, m1 = 1
解密结果：n0 = 0, n1 = 1
加法测试：0 + 1 , true
加法测试：0 + 0 , true
加法测试：1 + 1 , true
加法测试：1 + 0 , true
==============================================
乘法测试：0 * 1 , true
乘法测试：0 * 0 , true
乘法测试：1 * 1 , true
乘法测试：1 * 0 , true
==============================================
```

## step_2 部署合约，输入参数

以remix IDE为例

输出初始化参数：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/MwFMIc.png" alt="MwFMIc" style="zoom:67%;" />

## step_3 测试

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210812230115286.png" alt="image-20210812230115286" style="zoom:67%;" />

1的密文为：1318594171818636545920576603798101818973

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/KBUSK7.png" alt="KBUSK7" style="zoom:67%;" />

0的密文为：1318594171818636545920576603798101818962

#### 同态加法

1+0 = 1

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/ns3yge.png" alt="ns3yge" style="zoom:67%;" />

1+1 = 0

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/Zf6t09.png" alt="Zf6t09" style="zoom:67%;" />

0+0 = 0

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/hJzStL.png" alt="hJzStL" style="zoom:67%;" />

# 拓展与改进

1. 在合约中实现字符串**大数的基本计算**就可以实现合约上的同态乘法（或许有更好的办法）

2. 虽然输入只支持1bit，但是可以通过组合电路实现高阶的计算：

   * 同态加法 等价于 逻辑异或
   * 同态乘法 等价于 逻辑与
   * 逻辑与与逻辑异或具有完备性，可以实现组合电路任意高阶计算

   （图片来自论文）

   <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210812230914554.png" alt="image-20210812230914554" style="zoom: 50%;" />

3. 设计电路时注意使用Bootstappable算法减少噪声，不然会失效

欢迎Start，后续继续更新
