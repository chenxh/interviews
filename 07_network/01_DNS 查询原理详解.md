## DNS 服务器
域名对应的 IP 地址，都保存在 DNS 服务器。
我们输入域名，浏览器就会在后台，自动向 DNS 服务器发出请求，获取对应的 IP 地址。这就是 DNS 查询。


网上有很多公用的 DNS 服务器，Cloudflare 公司提供的 1.1.1.1 .

## dig 命令
命令行工具 dig 可以跟 DNS 服务器互动.

查询域名对应的IP的语法：
dig @[DNS 服务器] [域名]

```
dig @1.1.1.1 es6.ruanyifeng.com
```
答案在 ANSWER SECTION 部分。

## 域名的树状结构
DNS 是一个分布式系统，1.1.1.1 只是用户查询入口，它也需要再向其他 DNS 服务器查询，才能获得最终的 IP 地址。

要说清楚 DNS 完整的查询过程，就必须了解 域名是一个树状结构。

![缓存](https://raw.githubusercontent.com/chenxh/interviews/master/imgs/domain-tree.webp "图片title")

（1）根域名

所有域名的起点都是根域名，它写作一个点.，放在域名的结尾。因为这部分对于所有域名都是相同的，所以就省略不写了，比如example.com等同于example.com.（结尾多一个点）。


（2）顶级域名

根域名的下一级是顶级域名。它分成两种：通用顶级域名（gTLD，比如 .com 和 .net）和国别顶级域名（ccTLD，比如 .cn 和 .us）。

顶级域名由国际域名管理机构 ICANN 控制，它委托商业公司管理 gTLD，委托各国管理自己的国别域名。

（3）一级域名

一级域名就是你在某个顶级域名下面，自己注册的域名。比如，ruanyifeng.com就是我在顶级域名.com下面注册的。

（4）二级域名

二级域名是一级域名的子域名，是域名拥有者自行设置的，不用得到许可。比如，es6 就是 ruanyifeng.com 的二级域名

## 域名的逐级查询
这种树状结构的意义在于，只有上级域名，才知道下一级域名的 IP 地址，需要逐级查询。

每一级域名都有自己的 DNS 服务器，存放下级域名的 IP 地址。
ruanyifeng.com的查询步骤。

第一步，查询根域名服务器，获得顶级域名服务器.com（又称 TLD 服务器）的 IP 地址。

第二步，查询 TLD 服务器 .com，获得一级域名服务器 ruanyifeng.com 的 IP 地址。

第三步，查询一级域名服务器 ruanyifeng.com，获得二级域名 es6 的 IP 地址

## 根域名服务器

根域名服务器全世界一共有13台（都是服务器集群）。它们的域名和 IP 地址如下：
![根域名服务器](https://raw.githubusercontent.com/chenxh/architect-awesome/master/imgs/root-domain-server.webp "图片title")

根域名服务器的 IP 地址是不变的，集成在操作系统里面。

操作系统会选其中一台，查询 TLD 服务器的 IP 地址。

因为它给不了 es6.ruanyifeng.com 的 IP 地址，所以输出结果中没有 ANSWER SECTION，只有一个 AUTHORITY SECTION，给出了com.的13台 TLD 服务器的域名。

域名服务器能够查询 顶级域名的IP。


## TLD 服务器

顶级域名服务器。能够查询一级域名的IP。

## 一级域名的 DNS 服务器
查询二级域名的IP


## DNS 服务器的种类
* 递归 DNS 服务器： 把分步骤的查询过程自动化，方便用户一次性得到结果，所以它称为递归 DNS 服务器（recursive DNS server），即可以自动递归查询， 例如 1.1.1.1。
* 权威 DNS 服务器：IP 地址由它给定，不像递归服务器自己做不了主。我们购买域名后，设置 DNS 服务器就是在设置该域名的权威服务器。
* 根域名服务器
* TLD 服务器



