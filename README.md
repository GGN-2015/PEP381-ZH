# PEP381-ZH
An unofficial translation of PEP 381.



- 原文链接： https://peps.python.org/pep-0381/
- 日期：`2024-07-26`
- 免责声明：此翻译版本仅用于个人研究学习用途，不具有 PEP 协议的权威性。本译文不对翻译的正确性负责，也不对相关衍生作品内容的正确性负责。



- Original URL：https://peps.python.org/pep-0381/
- Date：`2024-07-26`
- Disclaimer: This translation is only for personal  study purposes and is not authorized by PEP administration. This translation is not responsible for the incorrectness of the relevant facts, nor is it responsible for the incorrectness of the content of related derivative works.



# PEP 381



## 摘要

本 PEP 描述了 PyPI 的镜像基础设施。



## PEP 的废除

2013 年 5 月，PyPI 的主要 Web 服务迁转而使用具有较快缓存的 CDN：https://mail.python.org/pipermail/distutils-sig/2013-May/020848.html

随后，PSF 接受了相关的实物赞助，使这一安排得以正式生效。同时，由 PSF 承担赞助终止而可能带来的风险。

先前由 PyPI 直接提供的统计数据，改为由 Google Big Query 提供：https://packaging.python.org/guides/analyzing-pypi-package-downloads/

因此，本 PEP 提供的提案不再被需要，于是标记为**已取消（Withdrawn）**。



## 基本原理

PyPI 目前拥有超过 6000 个项目，每天有无数的人们使用这些项目来构建自己的应用程序。其中，诸如 `easy_install` 和 `zc.buildout` 等系统需要大量使用 PyPI。

对于那些大量使用 PyPI 的人来说，PyPI 一旦出错，整个系统很可能因单点故障而失效。因此，人们需要设立对 PyPI 的镜像服务——这些镜像服务中既有私有镜像，也有公有镜像。它们都是网络上活跃的镜像服务，换言之，它们需要不时访问 PyPI 以对其进行同步。

为了让系统更为可靠，本 PEP 阐述了如下内容：

- PyPI 中镜像列表以及镜像在列表中的注册
- 一个共有镜像应当维护哪些页面，PyPI 要通过这些页面来获取 “命中数” 以及最后修改日期。
- 镜像服务应当如何与 PyPI 同步
- 客户端应当如何实现故障转移（fail-over）机制



## 镜像列表以及镜像在列表中的注册

想要为 PyPI 制作镜像的人需要在 catalog-SIG 上提出提案。当有人向邮件列表发起镜像提案后，镜像列表管理者会检查该镜像是否符合 PyPI 的镜像规则，如符合，则将其手动加入到 PyPI 应用中的镜像列表里。

镜像列表的主机名具有如下的形式：

> X*.pypi.python.org*

其中 X 的取值是一个字母序列：`a, b, c, ..., aa, ab, ...`。`a.pypi.python.org` 是镜像列表的主服务;从 X=b 开始为 PyPI 的镜像服务。我们使用 `last.pypi.org` 的 CNAME 解析记录指向当前所有镜像服务中的最后一个主机名。镜像服务管理员应使用静态地址，并提前将计划发生的地址改变告知 distutils-sig。

新的镜像也会出现在 http://pypi.python.org/mirrors ，这是一个人类可读的镜像服务列表。该页面也阐述了如何注册一个新的镜像。



### 统计页面

PyPI 在 `/stats` 目录下提供了若干静态页面以展示有关下载的统计信息。PyPI 每天会读取所有镜像站点本地的 `/stats` 文件并对其中提供的数据求和。

统计信息包含“日统计汇总“以及“月统计汇总“两种，分别位于 `/stats/days` 目录以及 `/stats/months` 目录。所有的统计信息文件均为 `bzip2` 文件，并以如下格式命名：

- 对于日统计汇总，文件名为 `YYYY-MM-DD.bz2`
- 对于月统计汇总，文件名为 `YYYY-MM.bz2`

下面给出了几个可能的路径实例：

- `/stats/days/2008-11-06.bz2`
- `/stats/days/2008-11-07.bz2`
- `/stats/days/2008-11-08.bz2`
- `/stats/months/2008-11.bz2`
- `/stats/months/2008-10.bz2`



## 镜像认证 

在分布式的镜像系统中，客户端可能想要对镜像站提供的项目副本进行验真。需要考虑的几个主要威胁列举如下：

1. 中央索引可能受损
2. 我们假定中央索引是收信任的，但镜像站提供的副本可能被篡改
3. 终端用户与中央索引间、终端用户与镜像站间可能存在中间人，而中间人可能会篡改数据包

本规格说明仅致力于解决其中第二种威胁。针对第三种威胁（中间人攻击），我们已经有一些现成可用的手段。对第一种威胁，我们要求软件包的作者使用他们的 PGP key 对软件包进行签名，这使得终端用户能够验证他们所下载的软件包是否确实来自受信任的作者。

中央索引在 `/serverkey` 目录下提供了一个 DSA key, 由 `openssl dsa -pubout` 生成并以 PEM 格式存储（例如，该 DSA key 可以根据  RFC 3280 SubjectPublicKeyInfo, 并由算法 1.3.14.3.2.12 生成）。这一 URL 不应被镜像，客户端必须直接从 PyPI 获取官方 serverkey 或使用随 PyPI 客户端软件一同安装的 serverkey 备份。但镜像站仍应该下载这一 serverkey 以检测密钥是否发生更新。

对于每个软件包，镜像站需要镜像它们的签名并将其安置在 `/serversig/<包名>` 路径，该签名是位于 `/simple/<包名>` 中的项目的  DSA 签名。签名应采用 DER 格式，使用带 DSA 的 SHA-1 进行签名。(例如，该签名可以遵守 RFC 3279 Dsa-Sig-Value, 并由算法 1.2.840.10040.4.3 生成)。

使用镜像服务的客户端需要使用如下步骤对软件包进行验真：

1. 下载 `/simple` 页面，计算其 `SHA-1` 哈希
2. 计算该哈希值的 DSA 签名
3. 下载相应的 `/serversig` 签名，并逐字节地将其与步骤二得到的签名进行比较
4. 计算所有下载文件的 MD-5 哈希值，并与 `/simple` 页面中提供的信息比对

此处提供了一个验证算法的具体实现：https://svn.python.org/packages/trunk/pypi/tools/verify.py

当从中央索引下载数据时，客户端不必进行验真，同时为了避免额外的计算开销，应尽可能避免验真。

所有签名密钥将大约每年更新一次。届时，镜像站必须重新获取 `/serversig` 下的全部页面。使用镜像服务的客户端需要下载一份新的可信 server key，一种获取的方式是从 https://pypi.python.org/serverkey 获取。为了检测中间人攻击,客户端需要验证由 CACert 权威机构签发的 SSL 服务器证书。



## 需要提供的特殊页面

镜像站是 PyPI 副本的一个子集，因此需要提供与 PyPI 一致的结构：

- simple：记录软件包索引的 rest 版本 **(?)**
- packages：按 Python 版本，以及包名字典序排序的软件包
- serversig：存储 simple 页面的数字签名

除此之外，镜像占还需要提供两个额外的具体信息：

- last-modified：上次修改时间
- local-stats：本地统计数据



### 上次修改日期

CPAN 使用新鲜度日期系统，其中镜像的最后同步日期可用。

对于 PyPI，每个镜像都需要维护一个带有简单文本内容的 URL，该 URL 代表镜像维护的最后同步日期。

日期以 GMT 时间为准，采用 ISO 8601 [^2]格式。每个镜像都应负责维护其最后修改日期。

该页面必须位于：`/last-modified` 路径下且必须是 text/plain 的纯文本页面。



## 本地统计数据

每个镜像站需要自行统计经由它下载的所有包的数量。PyPI 会使用这一数据来计算总的下载量。

这些统计信息需要以带有 header 首行信息的类 CSV 格式存储（服从 PEP 305）。基本上讲，它需要能够被 Python 的 `csv` 模块读取。

该文件中应当包含如下字段：

- package：包的 distutils id
- filename：下载时使用的文件名
- useragent：下载这一包时客户端使用何种用户代理（User-Agent）
- count：下载数

综上，该文件看起来类似如下：

```csv
# package,filename,useragent,count
zc.buildout,zc.buildout-1.6.0.tgz,MyAgent,142
...
```

统计工作自镜像启动之日开始，每天记录一个文件，使用 `bzip2` 格式压缩。每个文件都以日期命名。例如，`2008-11-06.bz2` 中存储 2008 年 11 月 6 日的日统计信息。

这些日统计信息需要存放于名为  `days` 的文件夹中，例如：

- `/local-stats/days/2008-11-06.bz2`
- `/local-stats/days/2008-11-07.bz2`
- `/local-stats/days/2008-11-08.bz2`

所有页面必须位于 `/local-stats` 目录下。



## 镜像站应如何与 PyPI 同步

基于 `easy_install` 的工作方式，Martin v. Loewis 和 Jim Fulton 共同描述并实现了名为 `Simple Index` 的镜像协议。 本节内容总结了 `Simple Index` 的相关工作，给出了几个相关的链接，并附加了一小段关于 `User-Agent` 的描述。



### 镜像协议

镜像站必须减少与中央服务器通信时所传递的数据量。为实现这一点，镜像站服务器**必须（MUST）**使用 changelog() PyPI XML-RPC 调用，且只重新获取自上次同步后发生过改变的包。钓鱼每个软件包 P，镜像站服务器**必须（MUST）**备份 `/simple/P` 以及 `/serversig/P`。当中央索引服务器删除某个包时，镜像站也**必须（MUST）**将所有相关文件全部删除。为了检测某个包是否发生了修改，镜像站服务器**可以（MAY）**缓存文件的 ETag 信息，且**可以（MAY）**使用 If-none-match 请求头的方式跳过未发生修改的文件。

所有镜像工具**必须（MUST）**使用 `User-Agent` 头信息注明自己的身份。

软件包 `pep381client` [^1]中提供了一个依本协议浏览 PyPI 的应用程序。



### User-Agent 请求头

为了能够区分不同客户端对 PyPI 的不同行为，镜像软件应该提供一个具体的用户代理名称。

以下的客户端用户代理服务同样需要符合这一标准：

- `zc.buildout` [^3]
- `setuptools` [^4]
- `pip` [^5]

~~用户代理在 PyPI 的注册机制有待讨论。~~ (?)



### 客户端如何使用 PyPI 及其镜像

访问 PyPI 的客户端应该也能够用来访问其他镜像源，使用 `last.pypi.python.org` 即可获得所有可用的镜像站中标号最后的一个。

代码示例：

```python
>>> import socket
>>> socket.gethostbyname_ex('last.pypi.python.org')[0]
'h.pypi.python.org'
```

到目前为止，以下客户端可以使用这一机制：

- `setuptools`
- `zc.buildout` (借助 `setuptools`)

- `pip`



## 故障转移机制（Fail-over）

当镜像源或着 PyPI 中心索引没有响应时，客户端应当使用故障转移机制。

具体使用哪个镜像源，应当由客户端自行决定，一种可行的做法是借助地理信息和镜像源的响应性的优劣。

本 PEP 文档并不具体描述故障转移机制应当如何工作，但我们强烈建议客户端尝试使用离自己最近的镜像服务。

到目前为止，能够支持这一机制的客户端有：

- `setuptools`
- `zc.buildout`（借助 `setuptools`）

- `pip`



## 额外包索引

显然，有些软件包出于某些原因并不会被上传到 PyPI，其中有的软件包是私有的，还有的软件包虽然没被上传到 PyPI 但却由维护者自行使用服务器提供下载，人们可以从这些非官方的渠道获得这些软件包。尽管如此，我们强烈建议那些公开的非官方包索引也要遵从 PyPI 以及 Distutils 的协议。

换言之，注册（`register`）以及上传（`upload`）命令应该与任何现存的包索引服务器兼容。

以下列举了一些现存的符合上述兼通条件的非官方包索引服务：

- `PloneSoftwareCenter` [^6]：用于 `plone.org` 产品部的运行
- `EggBasket` [^7]

额外包索引服务不是 PyPI 官方源的镜像，但它们也可以拥有自己的镜像。



## 包索引的合并

当客户端需要从几个不同的索引中获取一些包时，它应该能够使用每个索引作为包的潜在来源。客户端需要为不同的包索引制定优先顺序，以查找某个包。

每个独立的索引服务当然也可以提供针对它们自身的镜像。

~~本文未定义如何获取任意索引服务的主机名的方式。~~ (?)

这使得客户端可以对使用的包索引进行任意组合，根据不同的隐私等级形成一个可靠的包管理系统。

针对不同包索引的信息合并取决于客户端的具体实现。



## 特别鸣谢

Georg Brandl。



## 版权

本文件从属于公有域。


---

源代码：https://github.com/python/peps/blob/main/peps/pep-0381.rst

上次修改时间：`2023-09-09 17:39:29 GMT`


## 参考文献


[^1]: http://pypi.python.org/pypi/pep381client
[^2]: http://en.wikipedia.org/wiki/ISO_8601
[^3]: http://pypi.python.org/pypi/zc.buildout
[^4]: http://pypi.python.org/pypi/setuptools
[^5]: http://pypi.python.org/pypi/pip
[^6]: http://plone.org/products/plonesoftwarecenter
[^7]: http://www.chrisarndt.de/projects/eggbasket

