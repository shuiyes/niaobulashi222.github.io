---
layout: post
title: JavaScript判断对象是否是NULL
category: java
tags: [java]
copyright: java
---

写js经常会遇到非空判断

自己没有做总结，特地转载。很有帮助

```
function isEmpty(obj) {
// 检验 undefined 和 null
    if (!obj && obj !== 0 && obj !== '') {
        return true;
    }
    if (Array.prototype.isPrototypeOf(obj) && obj.length === 0) {
        return true;
    }
    if (Object.prototype.isPrototypeOf(obj) && Object.keys(obj).length === 0) {
        return true;
    }
    return false;
}
```