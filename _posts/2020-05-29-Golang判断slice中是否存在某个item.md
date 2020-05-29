---
layout: post
title: '利用Go反射机制实现判断slice中是否存在某个item'
date: 2020-05-29
author: pyihe
tags: Golang
---

### 为什么需要？

日常开发过程中经常遇到需要判断某个slice(或者array)中是否包含某个item的情况，比如判断10是否在[]int{1,2,3}中

### 怎么做？
一般最容易想到的方法是遍历slice中的每个元素，直到找到了该元素，否则返回false，如下：
```go
package main

func IsInSlice(value int, sli []int) bool {
   	for _, v := range sli {
   		if value == v {
   			return true
   		}
   	}
   	return false
}    
```
这种方法实现略显笨拙，并且针对不同的数据类型无法通用，如果有不同的数据类型，则容易生成很多相同功能的函数，只是参数不同，这样的话代码可用性并不高

### 如何改进？
golang中可以通过reflect包中的TypeOf(), ValueOf()和DeepEqual()接口对方法进行改进，方法参数使用interface{}类型，代码实现如下：
```go
package main

import "reflect"    

func IsInSlice(value interface{}, sli interface{}) bool {
    switch reflect.TypeOf(sli).Kind() {
    case reflect.Slice, reflect.Array:
	    s := reflect.ValueOf(sli)
	    for i := 0; i < s.Len(); i++ {
		    if reflect.DeepEqual(value, s.Index(i).Interface()) {
			    return true
		    }
	    }
    }
    return false
}
```
瞬间不用再为不同数据类型需要写不同函数而心烦了！

### 提示
因为反射需要的时间开销比较大，所以通用写法的效率肯定比特定类型的写法低，所以需要根据实际情况来权衡使用，如下截图为int类型的基准测试: 
![](/assets/img/2020-05-29/基准测试.jpg)
