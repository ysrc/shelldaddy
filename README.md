# ShellDaddy

跨平台 webshell 静态扫描器，在 Yara 的基础上做了一些优化和扩展，只要会写正则就能写扫描规则。

这是一个 YSRC 内部 “烂尾”的项目，感兴趣的同学可以联系 sec@ly.com，完善后我们走公司开源流程把代码开源出来。

## 使用方式

```
Shelldaddy -r -s -d path rule.yar

-h 查看帮助信息

-s 打印匹配的字符串，方便调试规则

-a 设置扫描超时

-r 扫描子目录

-p 算法判断出的可疑文件会被标记为 suspicious，加了 -p 参数后，ShellDaddy 会把 suspicious 状态的文件传递给 YSRC 的云端做进一步验证。

-t 启用tlsh算法，需配合 -m 参数使用

-m 启用算法

-d 指定扫描目录，默认扫描 web 目录

-o 指定扫描结果输出目录，默认为 ShellDaddy 执行目录
```

## 编译方式

win
...

Linux

Debain / Ubuntu 系

```
sudo apt install autoconf make libtool libjansson-dev libssl-dev
```

RH / CentOS 系

```
yum -y install autoconf libtool jansson-devel.x86_64 openssl-devel.x86_64
```

```
autoreconf -vi
chmod +x build.sh
./build.sh
```
	

## 说明

虽然说 webshell静态查杀比较局限，但是实际上市面上并没有能放心在生产环境用的、占用资源可控的，查杀率和误报率相对平衡的、真正跨平台的、开源 webshell 静态查杀的 tool。并不是说 ShellDaddy 就做到了，但是我们希望提供一个基础的轮子交由社区来优化。

webshell 判断是这样的，规则是一块，加上几个算法来发现可疑文件。 然后有用到TLSH算法来做模糊哈希、黑白名单，最后的话我们有提供一个在线的接口和web界面，用了一些其他技术来判定。

我们遇到的问题就是，目前我们自用是能满足需求的，毕竟我们没有 php 的生产级应用，可以不考虑变化最多的 php webshell。
但是如果开源出来的话，就得考虑普适性了，如果规则写的比较激进又会导致误报的情况，目前的规则说实话比较糙，不足以覆盖我们收集到的[外部样本](https://github.com/ysrc/webshell-sample)。

ShellDaddy 项目本身是准备作为我们主机安全的子模块先行开源出来的，但是因为各种原因，主机安全项目我们暂时放弃了webshell静态扫描的功能。

## 算法

算法部分主要借鉴了 [NeoPI](https://github.com/Neohapsis/NeoPI)， 和用 c 重新实现了趋势科技开源出来的 [TLSH](https://github.com/trendmicro/tlsh) 算法。


NeoPI 相关的算法很多地方都介绍过，就简单介绍下了。


**文件的重合指数（IC）**

设a是一个n长度的字符串，A=a1a2a3…an, ai是密文，a的重合指数为a中两个随机元素相同的概率。文件被加密后，随机性变大，重合指数就会变低，因此这种方式可以用来判断文件的加密可能性。


**熵值（entropy）**

熵是概率统计中的一种概念（离散随机）。一组数据越有序，熵值越低，越乱熵值越高。
 
 
**最长单词**

在正常的脚本中，为了便于阅读性和代码规范，几乎不可能有人写的代码一连几百行没有空格或回车换行。如果检测到就可判断这是加密脚本，加密脚本是webshell的概率就很大了。


**压缩比**

文件的压缩比=文件压缩后的大小/文件的原始大小。
压缩消除特定字符分布上的不均衡，通过将短码分配给高频字符，而长码对应低频字符实现长度上的优化。


**特殊字符串比重**

比如一个文件中像^ ~ 等等不常用的符号出现的频率超过一定的值可判断文件是webshell的可疑程度。


**TLSH**

一种新型的模糊 hash 算法，经过测试对比发现比 ssdeep 的速度快了接近一倍。不过有个问题就是不支持太小的文件。我们已经根据我们收集到的样本https://github.com/ysrc/webshell-sample 生成了TLSH hash 库。另外你也可以使用 生成 tlsh 库，欢迎提交。


**简易模型**

引入算法是为了补充规则在覆盖率上的不足，但是以上的算法除了 TLSH 算法之外 ，如果单独使用都会存在误报，所以我们引入了一个简单的朴素贝叶斯模型，每种算法各占一些权重。

检测方法有5种，那么A = [A1,A2,A3,A4,A5]，假设我们检测一个文件后各个检测方法的结果为：
A1 = 1，A2=0，A3=0，A4=0，A5=1 那么我们的目的是计算出P(B=1|A)和P(B=0|A)，也就是计算出该文件是 webshell 和不是 webshell 的概率各是多少。

贝叶斯公式为：

![](https://sec.ly.com/pic/20170524163421.jpg)

P(B=1|A)和P(B=0|A)的分母都是相同的，所以只需比较分子。
p(A1=1|B=1)，p(A2=0|B=1)，p(A3=0|B=1)，p(A4=0|B=1)，p(A5=1|B=1)，p(B=1) 这些未知的值需要根据样本去训练。

这里我们只提供参数的配置，webshell那个概率是根据我们自己收集到的样本获得。
不是 webshell 的那部分概率请根据生产环境去配置。如果 P(B=1|A)>P(B=0|A),我们就认为是可疑文件。

## Todo
1. 优化规则的质量，覆盖现有的样本。
2. ShellDaddy windows版本资源占用上做了限制，最多只能占用到25左右的 CPU资源，Linux的资源占用暂未实现。
3. ShellDaddy 基于 Yara 3.4，还是15年的版本，后续看社区反响看是否需要跟进下 Yara upstream
