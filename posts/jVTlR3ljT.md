---
title: ' 漫画：什么是B+树？'
date: 2021-04-25 17:03:03
tags: [算法]
published: true
hideInList: false
feature: /post-images/jVTlR3ljT.png
isTop: false
---

原创  [程序员小灰(公众号，强烈推荐)] 



在上一篇漫画中，我们介绍了B-树的原理和应用，没看过的小伙伴们可以点击下面的链接：



[漫画：什么是B-树？](https://tinaxiawuhao.github.io/post/LhRxiTagh/)



这一次我们来介绍B+树。





—————————————————





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWnUZWGqFibB67D1jJSrCib7dbKwb9alIrbd1EGrtCMP5cApLl7fOfu8dg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWxXe8URSjDot9qRMPfwUgLR0vYIEXPZm2PtyKhW5cDX0YYwiatwwicPDQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWI1nOoibYtF5g3ViaasktduzvaQa82RYIYKK7PpjR9GowH1d9ngRQLDpA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWDGkGmp0pbWZHy0ARgq5mFnEU43icOKl2WTkmRTt26hsOxSvr040ISOQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





**一个m阶的B树具有如下几个特征：**





1.根结点至少有两个子女。



2.每个中间节点都包含k-1个元素和k个孩子，其中 m/2 <= k <= m



3.每一个叶子节点都包含k-1个元素，其中 m/2 <= k <= m



4.所有的叶子结点都位于同一层。



5.每个节点中的元素从小到大排列，节点当中k-1个元素正好是k个孩子包含的元素的值域分划。





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWlS1Ga5NPKfWy0oMOdwic51e1GmB6Ly86xtnHJuOvPojiaZiazfn3G8o9g/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





**一个m阶的B+树具有如下几个特征：**



1.有k个子树的中间节点包含有k个元素（B树中是k-1个元素），每个元素不保存数据，只用来索引，所有数据都保存在叶子节点。



2.所有的叶子结点中包含了全部元素的信息，及指向含这些元素记录的指针，且叶子结点本身依关键字的大小自小而大顺序链接。



3.所有的中间节点元素都同时存在于子节点，在子节点元素中是最大（或最小）元素。





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWvGPhcLC5KUR6nS0y43UkBickpRqNDHoCyeKmNDcpwxgteSsyrdJSxibQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWlKtp41Tb329jCECIe2a05icnlBlVOTCdeQKNP6BPS8mtksdLStWIqoQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWdKrHQez446RaDLFZ9GzkcdduW75BlwD4YicSn9vDVianRuJrdK1x3xBQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWlENKtSK6Hw1giabCJm8ictbI6RcYpe2ibQ5bptEiakbyJ9aPh2tQyozbicQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWTKSPRlWicrkEvJVcPibE9wGyjJzfbntOSCTdg2B23fpLFmwx8uQibW3nw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWkfCibcqSxZANC3vkHibXasqAibGY15qqc1QicpyboHorSpep9XZnaDIefg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWyqIwczSBNzHJF9UkBKaPJFva7z1zA9OlpVXac0xFiar3eapQFhfZnCg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pW9yjIQHedIcYbWGTdknjMb4k2YCJbu4R0oenib3aHKKmNLrNHFVHFjHA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWI9NoahgCbKFYcQZ0x3sMMTzzpOUx4WuHl6PaoNeicczBT9xxRvaqjpw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWu9JxaBSILSojEUdby9NHbApkSfJhCcqTR0zwma0p6CPLF6vTBFtGEg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWibeDfY4E8p2UFFm3UnBC02mwAI0ic7LvYGddoC1NT7E96XlKM5yDI9fg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWfgialxT2Lk6EKIW2keLbSCQI9oibwvnJthSGahclps7c1u0RtqzJ5UNA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





B-树中的卫星数据（Satellite Information）：





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pW7cPy9dgkyNkgRmSSR0n0Hueuc6xiatV8xjV3csnrLKQtibsbAzHODq6g/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWWUQgyrJINKO4tNI5R3XRdNAIYViaIS1icuUzYibtok7mPokibTYlc5iaYwg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





B+树中的卫星数据（Satellite Information）：





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWHh99F2iakg9snMXq2riaQvI96ZBKq8ibOyVABr12HichuYYBgDAU6JibmuQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





需要补充的是，在数据库的聚集索引（Clustered Index）中，叶子节点直接包含卫星数据。在非聚集索引（NonClustered Index）中，叶子节点带有指向卫星数据的指针。





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWlibEclqdicliczItOQ0BTCc94Tjx4hK3QOibTBYqTic9W99bfkZEepHhobg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pW97oNUTsZBesRIa9JMfgHPugQRiaXOR9ByFYawic5zp5O7ZIppicY9ibpaw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pW1O0oglp3SdXsN4iaKAuhQdpwkj9vMD4N5JjOgA4guCNPtmetOgiaP3Rg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





第一次磁盘IO：



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWALKA2cKa2Az6vFk9jcXkSCgXW23aECY3IM9qkZibEeJsW1133T32q0A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





第二次磁盘IO：



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWQ5xibk8HqcNj2Py6qSXaFSDFPicrtMlctR8ibwibp8NOHibY7UTYAGkXPNQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





第三次磁盘IO：



![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWRUfNhz8DaSYPmp49KBibHYmiab2oPFqa27pnNH22N6UdicgpVUibx7cjKA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWHTfuPgD3Okks1JvaiafgwCoFISZs4ja9ILVUXsupMHSv7p6K5licH0Vw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWrR88bBnTTqgjtjxxsYUpDdg9ra3rchusBudFN9b11ncLHHG0OQw3mA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWt280MzTs4m5ZE1LXib1pW2Kdm9y7icjtTq8lHzot5SneAh4ubibzFibiawA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWBeRTP5g5InIURA5g5V8hcgzvkqowr3d7m0p3dkt19Mmic3QNtqtFyaA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWkpblZmEgJKVZZtkhrTlf8JYxS5ohjmoWhkIww4WFficHkENauhl7F4g/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





**B-树的范围查找过程**

**
**

自顶向下，查找到范围的下限（3）：



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWRZKppF7LWMS5ibIvtRnIhgyp2Ufvg6iaGqaXiaPWLdA62yc7egmTPlnmg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





中序遍历到元素6：



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pW1Z3JotpdD0ibcfYg4hLwUyUmrN6ia4t7sVVh0yfpMmlib2XbQI6mVQRwA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





中序遍历到元素8：



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pW8F3jYDUmNbLyoBTKP5nDiblULjyWGpkicic7Icm5QvibzcfNfC05rvB6VA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





中序遍历到元素9：



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWEKahLygMy0dib0UU7MXs0PEicYFhicHSFTeiahGib5TCAWX9vFia94asDoyg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





中序遍历到元素11，遍历结束：



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWlVibahZaopYBD1QKLPk7pnCKbOeRmVa6iavA5o8O2GczP67oEu7rQqSg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWhp9SXcoMfE53EIEug81ibmJafmfTglXgwO7rwG9AZj9FY71icD6ujU5A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pW9KrEfPDm82hWPBByicp4JGmktZeQlvr6nEF7BC812o9jQ4ktmMcoPuw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





**B+树的范围查找过程**

**
**

自顶向下，查找到范围的下限（3）：



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWhgC1HqVwF8WQYv7icZ0uFmibfDLRCLQiby5pfPLnJ1BYxeW9bfxEG8LPQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





通过链表指针，遍历到元素6, 8：



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWQEQzMogDLI7WicDib78y1WDn4Jic4YN7m2FAEUyyfA82UVVdmSKQ5P6sw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





通过链表指针，遍历到元素9, 11，遍历结束：



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWxkfoKkeDiasB0srl8Aa34QfPhHmeurELvHic0nFzUOHLvO7v2FUW1ZfQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWkGfwgBuz8sufnXHjjLZPibHy5LM0vKlzf39Z9uuQoiaOhQcUKickNSGdw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pW0HcK41YwayXk62aBHiba7tKN5B1CLEPTmU0ZvcpOiaalMXujvJicr7JqQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWkWdGj1PIO1ria6VjicQZO8O7vubRGQtLCyXveOFEKRUhD8FlyloX11pQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWD8dD8T7Gp6VsVJTFiaxGNFjhyYJL5nnPN7hJ3WumQPSJdXHwo4sosSg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





**B+树的特征：**



1.有k个子树的中间节点包含有k个元素（B树中是k-1个元素），每个元素不保存数据，只用来索引，所有数据都保存在叶子节点。



2.所有的叶子结点中包含了全部元素的信息，及指向含这些元素记录的指针，且叶子结点本身依关键字的大小自小而大顺序链接。



3.所有的中间节点元素都同时存在于子节点，在子节点元素中是最大（或最小）元素。





**B+树的优势：**



1.单一节点存储更多的元素，使得查询的IO次数更少。



2.所有查询都要查找到叶子节点，查询性能稳定。



3.所有叶子节点形成有序链表，便于范围查询。





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrhjbBgkNEqGwLjaRu359pWGz1E1iciaq8bzs2miaDPcn7pibLThbjA5llpOjTh0DdyCQXT9g8evfibdPQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)







—————END—————