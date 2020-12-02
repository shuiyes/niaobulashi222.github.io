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

```
package javatest;
 
import org.apache.commons.lang.StringUtils;
 
import java.util.List;
 
/**
 * @version V1.0
 * @Description: 测试
 * @author: niaobulashi
 * @date: 2020/09/23
 */
public class Test {
    public static void main(String[] args) {
        // 创建一个二维数组
        String a[][] = {{"0", "800000"}, {"800000", "1600000.2"}, {"1600000.2", "-1"}};
        if (!"0".equals(a[0][0])) {
            System.out.println("返回错误信息：不是以0开头，该区间为不连续区间");
        }
        if (!"-1".equals(a[a.length - 1][a[0].length - 1])) {
            System.out.println("返回错误信息：不是以正无穷结尾，该区间为不连续区间");
        }
        for (int k = 0; k < a.length - 1; k++) {
            if (!StringUtils.equals(a[k][1], a[k + 1][0])) {
                System.out.println("返回错误信息：该区间为不连续区间");
            }
        }
    }
}
```


  [1]: https://images.niaobulashi.com/typecho/uploads/2020/09/1047437796.png



