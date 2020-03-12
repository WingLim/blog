---
title: "算法学习：寻找第K大的数-Python"
date: 2020-03-12T23:51:34+08:00
slug: algorithm-learning-findk-python
description:
draft: false
hideToc: false
enableToc: true
enableTocContent: false
tocPosition: outer
tocLevels: ["h2", "h3", "h4"]
tags:
- algorithm
- Python
series:
- Algorithm Learning
categories:
- 学习
image:
libraries: katex
---

给出一个未排序的数组，在这个数组中找出第 K 大的元素。

<!--more-->

## 原理

### 挑选基准值

从数组中挑出一个元素作为基准，这里选择数列中第一个元素作为基准。

### 分割

重新排序数组，所有比基准值小的元素摆放在基准前面，所有比基准值大的元素摆在基准后面（与基准值相等的数可以到任何一边）。

### 递归寻找

分割完成后，返回基准值在数组中的位置 `i`，因为数组下标从 0 开始，所以第 K 大的数在数组中的下标为 `k-1` ，

1. 如果 `i == k-1`，则说明这个基准值就是第 K 大的数
2. 如果 `i > k-1`，则第 K 大的数在基准值的左半部分
3. 如果 `i < k-1`，则第 K 大的数在基准值的右半部分

情况 1 直接返回基准值，情况 2，3 则继续递归寻找。



## 实现

```python
# 将数组分割
def partition(s, low, high):
    # 取第一个数为基准值
    pivot, j = s[low], low
    for i in range(low + 1, high + 1):
        if s[i] <= pivot:
            j += 1
            s[i], s[j] = s[j], s[i]
    s[j], s[low] = s[low], s[j]
    return j

def findK(s, start, end, k):
    i = partition2(s, start, end)
    if i == k-1:
        return s[k-1]
    elif i > k-1:
        return findK(s, start, i-1, k)
    elif i < k-1:
        return findK(s, i+1, end, k)
    return 0

if __name__ == "__main__":
    arr = [int(n) for n in input("输入数组:").split()]
    k = int(input("K 的值:"))
    result = findK(arr, 0, len(arr)-1, k)
    print("第 {} 大的数为 {}".format(k, result))
```

### 测试

```bash
输入数组: 4 7 6 5 3 2 8 1
K 的值: 5
第 5 大的数为 5
```

### 复杂度分析

在分割运算中，数组的元素都会在每次循环中走访过一次，使用 $O(n)$ 的时间

每运行一次分割，我们会把一个数组分为两个几近相等的部分。则每次递归调用处理一半大小的数列。因此，在到达大小为一的数组前，我们只要作 $logk$ 次嵌套的调用。

所以这个算法的时间复杂度为 $O(nlogk)$，因为 `k` 为常数，所以时间复杂度为 $O(n)$