---
layout: post
title: Java 校验是否为连续的区间
category: java
tags: [java]
copyright: java

---

> 工作中遇到需要校验是否为连续

给出示例

|  ----  | ----  |
| 0  | 100 |
| 100  | 600 |
| 600  | -1 |

从0到正无穷的连续区间。

使用-1代表无穷大

可以考虑使用二维数组array来存放数据，同样使用二维数组比较数据是最方便的。

可以找到规则

 - `array[0][0]=0`，第一个数据总是等于0
 - `array[0][1]=array[1][0]`
 - `array[1][1]=array[2][0]`，从第二个数据开始，等于下一个的第一个数据，以此类推
 - `array[2][1]=-1`，最后一个总是等于-1（正无穷大）

![二维数组临近相等][1]

通过以上分析，可以使用二维数组来校验是否为连续的区间

![代码截图][2] 


  [1]: https://images.niaobulashi.com/typecho/uploads/2020/09/1047437796.png
  [2]: https://images.niaobulashi.com/typecho/uploads/2020/12/1173788277.png



