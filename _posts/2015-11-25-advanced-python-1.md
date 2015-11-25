---
layout: post
title:  "Python 技能提升（1）"
date:  2015-11-25
categories: 技术学习
---

## 语法最佳实践A —— 对象级以下

### List 遍历

问题：  
  numbers是1-10的数组，找出numbers中的所有偶数。  
  常见的做法是写while循环或for循环：  

    numbers = range(10)
    size = len(numbers)
    evens = []
    i = 0
    while i < size:
      if i % 2 == 0:
        evens.append(i)
      i += 1
