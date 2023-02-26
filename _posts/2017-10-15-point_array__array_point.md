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
    int  *paar[5] = {&a,&b,&c,&d,&e};
    for(int i = 0 ; i < 5; i++) {
        printf("%d\n",*(*paar+i));
    }
    return 0;
}
```

# 数组指针
  首先它是一个指针指向一个数组

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
    int (*paar)[5] = arr;
    for(int i = 0 ; i < 5 ; i++) {
        printf("%d\n",*(*paar+i));
    }
    return 0;
}
```
# 小结

增加对指针的理解和认识.
