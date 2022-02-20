---
title: Weekly Report 2022.08
description: 幸福如出一辙，不幸难觅一同
date: 2022-02-20
slug: 2022-08-week-report
image: img/2022/02/FaceOff.jpg
categories:
  - week-report
tags:
  - week-report
---

Combine theory and practice: Nothing works and don't know why.

## Reading

### 《经济常识一本通》

不能说这本书的内容有多好，但是作为一本通识的入门还是有一定效用的。另外记录了一些[笔记](https://rin.azusachino.cn/reading/%E7%BB%8F%E6%B5%8E%E5%B8%B8%E8%AF%86%E4%B8%80%E6%9C%AC%E9%80%9A/index.html)，以留后续速查。

## Learning

### 旋转图像的两个途径

- reverse upside down, then swap the symmetry
- reverse left to right, then swap the symmetry

```c++
/*
 * clockwise rotate
 * first reverse up to down, then swap the symmetry
 * 1 2 3     7 8 9     7 4 1
 * 4 5 6  => 4 5 6  => 8 5 2
 * 7 8 9     1 2 3     9 6 3
*/
void rotate(vector<vector<int> > &matrix) {
    reverse(matrix.begin(), matrix.end());
    for (int i = 0; i < matrix.size(); ++i) {
        for (int j = i + 1; j < matrix[i].size(); ++j)
            swap(matrix[i][j], matrix[j][i]);
    }
}

/*
 * anticlockwise rotate
 * first reverse left to right, then swap the symmetry
 * 1 2 3     3 2 1     3 6 9
 * 4 5 6  => 6 5 4  => 2 5 8
 * 7 8 9     9 8 7     1 4 7
*/
void anti_rotate(vector<vector<int> > &matrix) {
    for (auto vi : matrix) reverse(vi.begin(), vi.end());
    for (int i = 0; i < matrix.size(); ++i) {
        for (int j = i + 1; j < matrix[i].size(); ++j)
            swap(matrix[i][j], matrix[j][i]);
    }
}
```

### Centos8 Vault

由于 Centos8 已经进入 EOL 阶段，所以 yum/rpm 不能正常使用了。参考着 Tuna 的教程，更新了 yum 仓库的地址。

```shell
# CentOS 8
minorver=8.5.2111
sed -e "s|^mirrorlist=|#mirrorlist=|g" \
         -e "s|^#baseurl=http://mirror.centos.org/\$contentdir/\$releasever|baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos-vault/$minorver|g" \
         -i.bak \
         /etc/yum.repos.d/CentOS-*.repo
yum makecache
```

## Life

现在的当下之急就是找到一个能够持之以恒的目标。

## Quote

> - You may be morally and intellectually superior to everybody around you, but don’t try to make it too obvious unless you really intend to irritate somebody.
> - 有效地关注重要之事。这样的智慧，因为反思、理解并欣赏生活而伟大，并不仅仅因为掌控生活而伟大。

## Recommend Articles

- [Linux kernel management style](https://www.kernel.org/doc/html/latest/process/management-style.html)

## Reference

- [centos-vault 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/centos-vault/)
