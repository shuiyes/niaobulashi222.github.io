---
layout: post
title: Java处理excel时遇到的一些问题及总结
category: java
tags: [java]
copyright: java
---

>最近的开发过程中遇到需要导出excel，以及导入解析excel的需求
>需要先导出excel，修改之后，再用该excel导入处理
---

## poi无法解析ooxml格式excel
遇到了第一个问题，poi无法解析ooxml格式的excel文件，[poi][1]是apache解析excel文件以及相关操作的jar包。
当时平台导出的excel格式文件，无法导入解析，总是报错
```
java.io.IOException: Invalid header signature; read 0x6576206C6D783F3C, expected 0xE11AB1A1E011CFD0 - Your file appears not to be a valid OLE2 document
```
在网上查阅资料好久，终于找到了一位博友的文章

[使用apache-poi生成word文档不能被office读取的问题][2]

可惜现在他的网站打不开了 ::aru:crying:: ，还好当时有记录问题截图

![博友描述的问题][3]

很明显了：poi并不支持ooxml格式的excel解析

所以找到一个excel公共处理项目，个人觉得很方便

github：[easy-excel][4]
具体项目的操作已有说明，支持导入导出

该项目导出的excel为office的xls正常格式，可以满足这次导入的解析 ::aru:cheer:: 



  [1]: https://poi.apache.org/
  [2]: https://www.fanyeong.com/2016/03/20/%E4%BD%BF%E7%94%A8apache-poi%E7%94%9F%E6%88%90word%E6%96%87%E6%A1%A3%E4%B8%8D%E8%83%BD%E8%A2%ABoffice%E8%AF%BB%E5%8F%96%E7%9A%84%E9%97%AE%E9%A2%98/
  [3]: https://niaobulashi.com/usr/uploads/sina/5d1c5144b6c4c.jpg
  [4]: https://github.com/niaobulashi/easy-excel