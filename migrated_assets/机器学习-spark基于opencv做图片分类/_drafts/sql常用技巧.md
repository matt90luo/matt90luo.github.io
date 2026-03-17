---
title: sql常用技巧
tags:
---
sql写的好 代码写得少, sql + excel 是我一直推崇的数据分析和特征观察的工具组合, 可以彻底从脚本代码中解放出来, 直接依托excel强大的统计库处理功能事半功倍 但是对于sql技巧就会有要求 所以特别总结了这篇博文

1. 计算连续出现某个item的个数并且汇总
描述一下场景:
0, 0, 1, 1, 1, 0, 1, 1, 0, 0, 0
分别出现了连续的2个0 连续的3个1 连续的1个0 连续的2个1 连续的3个0
所以最后希望得到的结果是

|item| cnt|
| -- | --|
| 0 | 2|
| 1 | 3|
| 0 | 1|
| 1 | 2|
| 0 | 3|

关键sql
```sql
select t.*,
(row_number() over (partition by cookieid, seg_ts, seg_cate order by rowsid) -
 row_number() over (partition by cookieid, seg_ts, seg_cate, flag order by rowsid) ) as grp
from tab_b t
```
我们要对 flag 出现的连续次数进行统计 并且需要按照 cookieid seg_ts seg_cate 分组统计


2. 累计求和
* 第一种解决方法 row_num 然后使用join 解决
* 分析函数 over(order by ... rows between unbounded preceding and current row)
[参考1](https://stackoverflow.com/questions/2120544/how-to-get-cumulative-sum)
[参考2](https://stackoverflow.com/questions/36927685/count-number-of-consecutive-occurrence-of-values-in-table)




