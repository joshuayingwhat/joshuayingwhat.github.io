---
layout: post
title:  "指针数组和数组指针基础"
categories: C语言
tags:  C基础
---

* content
{:toc}

# 指针数组
  首先它是一个数组里面存放着指针。
  int *parr[]在这行语句中[]的优先级最高所以parr先和[]结合成为一个数组,外面的*表示这个数组里面存放的是指针。
  
  如下代码:
  ```c
  #include <stdio.h>
/**
 * 指针数组 首先他是一个数组 里面存放着指针
 * 
 * @return int 
 */
int main() {
    int a = 1;
    int b = 2;
    int c = 3;
    int d =4;
    int e = 5;
    int  *parr[5] = {&a,&b,&c,&d,&e};
    for(int i = 0 ; i < 5; i++) {
        printf("%d\n",*(*parr+i));
    }
    return 0;
}
```

# 数组指针
  首先它是一个指针指向一个数组.
  int (*parr)[]在C语言中()比[]的优先级要高所以paar先和*结合成为一个指针所以它是一个指针然后在和外面的[]结合表示这个指针是指向一个数组的。

  如下代码:
  ```c
  #include <stdio.h>

/**
 * 数组指针 首先他是一个指针 指向一个数组
 *
 * @return int
 */
int main()
{
    int arr[5] = {0,1,2,3,4};
    int (*parr)[5] = arr;
    for(int i = 0 ; i < 5 ; i++) {
        printf("%d\n",*(*parr+i));
    }
    return 0;
}
```
# 小结

增加对指针的理解和认识.
