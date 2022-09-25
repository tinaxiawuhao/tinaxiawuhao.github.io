---
title: '漫画：什么是B-树？'
date: 2021-04-25 16:58:34
tags: [算法]
published: true
hideInList: false
feature: /post-images/LhRxiTagh.png
isTop: false
---
原创  [程序员小灰(公众号，强烈推荐)] 

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4omperuU9tXVmGtiaebVYhIH8LzNQ4SQ9gcmt8lFRmbVsote0EWZnYWkFQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4omvE3DX4LSX7UibSEHDpZOf4OXTIkmqCKs1ib0kbiaiauTePPDaxIqwLyLeg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4omELrLF4iagicxUdv5w5btvMcQQ1ohptiaaSwAia1Fkuch7G3fkknia7fNePg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4omSKmWO5RhbfETA3zpFeHZ4cibNoIicvH53qDjqG8ACNCicNcWEfzw4XWAg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





————————————





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4om3Qb9JbdqRkZrupQiao3q5icrLj50oKiclmhqoQ7KYhrMw9U0RkccRn8LA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4omZgqJwlvKKIEiboZYN1bj1ptC8JDBpSgMK5sGsXVbfvqLhmAsYnCwa1A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4om164ApDQhpkSADVz47icNO3o935TepKIUW9xrsR2G3D5I1mXGBTEuHXg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4omcoibibpZkKHvjlCKsltt6ibWFekKiaVmJhVolWSicZxNu7PdWF0gyoia1xuQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4omicanwicic4BYm8mbD3Q8pke9DVg0PtxeURueV3GgANAejY08UFSsThehQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4om0W1TkatXcgYSTJ7oJ2LqfJmw3qQug4b9P3mUyTc0K0fzKj1EIBJynQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4omBUWHsicmVibFZUZZReEcs6Zc3ib6QXLYK4ia47FTFicTMX0OUhdAph2v64Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4omjicVxJO4kgHOBz8b22xEXHHzGr1pYlAibauqDbm2VA0MI1oITlpzZ5Yw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





————————————





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4omdmtOGTVtHfdQf3LclwKpo1CKYmUfMR4l2QUksDsl1oX13yMBe9FD0Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4omaDShib6S2yT1WpbviaA9LIUfkoiaGNh0l4CXvUm1ZJCrAhibqaZm06RgOQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4om6Hic9ucmUQKD1sOvC8y3JicHFGXEaOBl2MqpFsggOShs3g7BQzzPXicXA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4ommNia994ceJWX4EWfoMh1WX62ZaicIichj5saiaVpexWyKrLicubIjHckKrA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoX9aS1vS0cNOWWdNpKV4omSOUSOqWqT1nIZABVxuOOj73vcJwzWXnTdLpNTuGgib4MJnQoLTxaMdg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)





![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)





![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsCoxqcduibyKDpCnAnmibnVNaqJV0YibhpYYp3gE65N1W771XqzzibZQelA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2Ecibs1G1ickrTiaqeCZU52j6ibcj17vYGUiccE8oALmTQXbiaeOKcc5ZibAwj966Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibscvQzdW5SqAhOjV8y1Q3XewCCkG8YbzlBFibQMMAeOJQLFgtiaqcJYnwA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsLu2WteEnEZdVUbm2XLh5rDfKcWeLYYbUkDl75lo24jib4CbWmGHCZug/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsoaByag5Eh52qcZNialGyVKHvcicQ9HFxrWTsvJ7Zeh02evJN2ufTZ5Rw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsKyTxsxAClzrIgScDUhDEhO1zTcncial0ZJwu3ZJQRwKRug9F1VlU9icQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





二叉查找树的结构：



![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)





第1次磁盘IO：





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsMKibN9giaNVMbFIlnvTkruWuatuQdvoN07INpbQMFfibtlamFibYTYZjTw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





第2次磁盘IO：





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsonW75DsA6sbibqEEDib8e127Bmlx1cjNiacNAtJRtHdgssedibHxWVz0Bg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





第3次磁盘IO：





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsnjKOKCu20S5b2apqe3RHqiaL5cRTPxicF2NQ4j4ic8dbvBwTGgdCLHbxg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





第4次磁盘IO：





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsqUOlpMg643uxxrb9ygP4Shq20MEicRsv5GqzxqImolibJOVf5Qn9ySgQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsRbGPSjia4RQWOVQ2DZVhYibibJasdg3v3QMNS0p4IJtDnAcOnVP4xYp8Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2Ecibsr6LX79azuhDhNsNibBibVanpqDbccbibgkFziaNHsTPQNsLPq8GSxgNQtQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsmRy2uqE5dPtRSbhIJ4ciaZTMGD3dzDYTAQETXGa3zxj0AdLfFphhCtA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsibqHeTIn9hibc3GJGiarSS9xH0SpPhaclQvKBYy5nbmTg1cYnJM3AFfpg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





下面来具体介绍一下B-树（Balance Tree），一个m阶的B树具有如下几个特征：





1.根结点至少有两个子女。



2.每个中间节点都包含k-1个元素和k个孩子，其中 m/2 <= k <= m



3.每一个叶子节点都包含k-1个元素，其中 m/2 <= k <= m



4.所有的叶子结点都位于同一层。



5.每个节点中的元素从小到大排列，节点当中k-1个元素正好是k个孩子包含的元素的值域分划。





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsjkSxcKGRzGmJ3iaxkBAnc0vqOfTn2nMJWTQLICNFh8f4S8TeVpzeX6w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2EcibsBx8AoeUa9IVbk2JbdsjMZ5p0tdicHkO6fN6dCIwdIojkxicK4GdAv7XQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)







![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGoBqEgxvibyZ3S2O8GG2Ecibs5LFXxLRhEVH55mqXMhGf9C2icZ4iaFcBhUhABThvG0hiaoD1z4icgQJbbA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRSQb6eDxZctbAxvNt5gYKPlFEiaNaPcqor2hJ6peyuxGQDmQ8Gaougibw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRictJGn6axEj3BMbc5RwBiaAPPC9mWxYl6yhQ8icB1YAnux37NmYiatKQXQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRwHXAS5B5icGP98ChYyCQ3jiaw7ickeibRBibZTtu5HwWM30R15VAbCh4tOQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRxZ6vZjclNWA40cibmZAic9ZZYeZT4aOibCZVTAc4CaauXJN76UvDb7m3Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





第1次磁盘IO：







![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRCCdQPjxVB531QYltibYibYIo6ialib6bg8sXJOBOeicDvEUvniastwicibaeibg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





在内存中定位（和9比较）：





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRIiasUvxOs9ibvlu4PP4gtPZmAbhEVich1tpja1PDWeHyx5491J9bLE12A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





第2次磁盘IO：





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqR4ccGREXzfjztA98p0b4dICrLAXrsibqYaQU5MRezYuAchHlpdl9MIzw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





在内存中定位（和2，6比较）：







![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRKf8XQytR3X9B321Nkva8Fovzn9Fmn95HZP4oK1Il7aFFtCdyhHffUw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





第3次磁盘IO：





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRMqrICiaHEH2ucwtbSgh4AT3usOO98Axv3mwHGWOea2GkJQ77Svxfs5g/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





在内存中定位（和3，5比较）：





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRu8LjAqtagyeicY27yaUG0SmmLQLCOP3VQrQNIeMCaxfibY6PpicuF1tyg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqROjXMmxDcDp47DBcuORbpMnxg50rsBQ96CN1V8kxL2Z7KjsgJCl5diaw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRY7fntn7UFgicRUrjHaD0bdmWVYwVFHbcxYNVcibSeWIXbhiaAoVzccEiaQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRzlhRceia6QePPtGlxXchpFllA5Pj2uxicwJGQNLiaJQcicDNYQ3ptnqn2g/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRXpicfJ3TeTpKia3Q3O8g2GveeiadPpGQLU4LAt4hNCKra1fFSlHTvCWNw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRK4Kd1rCcGicTltzlzVFUWiblatibLy2rYfiaYrCKwv8IXb8egabgXJgWJg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





自顶向下查找4的节点位置，发现4应当插入到节点元素3，5之间。



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRYk5euDrbMgVubTR4gvIL3U4LdicK3Fu1f7ATq9tGChL9YLXibJeuCTCA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





节点3，5已经是两元素节点，无法再增加。父亲节点 2， 6 也是两元素节点，也无法再增加。根节点9是单元素节点，可以升级为两元素节点。于是**拆分**节点3，5与节点2，6，让根节点9升级为两元素节点4，9。节点6独立为根节点的第二个孩子。



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRSBqOKQc3NYe8HA7mX4QjSjUpfkWLqGBfWm8wHmic7qajFaickGSTX8NA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)







![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqR9v2h25hzn192XuqicoUVCWIBEt6gKRocujXzDJiamrgib1EV1ibSpxNQUA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRbtPeqBtFvXneXlicy7aPRWiaQ59goTjyCAmnX1tTQ0KlY24NbMxtfF0w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRmC7NLG3HUZoqpvP30UDSMTj0WNwoUYhakYNibAQaUlKiclicvyMNt3bGA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





自顶向下查找元素11的节点位置。





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRfnGwPGrX4NhnkJfBuKqNdXAyHWJiaBd5b0ruibuhHJvUnmCYpt1eJxdg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





删除11后，节点12只有一个孩子，不符合B树规范。因此找出12,13,15三个节点的中位数13，取代节点12，而节点12自身下移成为第一个孩子。（这个过程称为**左旋**）





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRta9bOwANA1gjzduWKKAhl5N9IkFgZPNYNNzzdByY7LYiautLXVNnJ4w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRbDOA2SYhksUO3BlibgZciauxcTZFfXobCnFwPegUjLHYqwq4CW7gTYGg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRJL3EcNOic0fHafn4ltyInugPY8jtiaUkfQWMYOia3zcsqEmibZ7xszsvvw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRdkMYKpR4OMWfwLVEjyVTFvN247FyLu2xh6tiaAaZHLq3CdtbpHJNCicQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqRDNv7EqMgsEBvBibODkjW2iaO8ib2pOvBlarsmXK3bVPTJcqCV6qS9yTqA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/NtO5sialJZGrbI83f8meYn1UlzibHGBcqR9e717QtA8PSsFvkWnwjs9HuI28mbpmvaLFFfSiakAlfnotvdZ0gibApA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)







—————END—————