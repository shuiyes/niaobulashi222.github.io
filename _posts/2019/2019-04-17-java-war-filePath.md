---
layout: post
title: 打包成war包之后如何读取配置文件
category: java
tags: [java]
copyright: java
---

今天工作开发中遇到一个问题：在idea运行的项目读取配置文件没有问题，打包成war包之后就会报错java.io.FileNotFoundException: class path resource

---
## 原因
打包成war包后，配置文件在war包中，不是一个独立的文件了，无法通过File的方式访问
```
String filePath = "classpath:template_xml/readexcel/test.xml";
File file = ResourceUtils.getFile(filePath);
InputStream fis = new FileInputStream(file);
```

## 解决方案
通过文件流的形式读取文件
```
String filePath = "template_xml/readexcel/test.xml";
InputStream fis = this.getClass().getResourceAsStream(filePath);
```

## 总结
开发过程中遇到读取文件的，尽量用文件流的形式读取文件，可避免在不同环境下可以正确读取