---
title: json
date: 2023-01-08T18:06:57+08:00
draft: false
categories: [""]
tags: [""]
---

# json
#go

go用首字母的大小写来判断一个变量函数等能不能被其他包引用，**小写字母开头的只能包内使用，不能被其他包使用**。
所以对于你这种情况，成员是小写的，函数内赋值都是没有问题的，但是json.Unmarshal是另外一个包，json这个包没法给你现在所在的包里的任何私有变量(小写字母开头的)赋值。
所以把成员改为大写字母开头的，这样json.Unmarshal就可以给它赋值了。

字符串类型encode成json串时，会被强制转义成有效utf-8编码，同时会把utf-8无法识别的字符用uncode代替。尖括号“<”和“>”被转义为“\ u003c”和“\ u003e”，以防止某些浏览器将JSON输出误解为HTML。出于同样的原因，标签“＆”也被转移到“\ u0026”。 可以使用在其上调用SetEscapeHTML（false）的编码器禁用此转义。
```go
	  //第二种方法，SetEscapeHTML(False)
    bf := bytes.NewBuffer([]byte{})
    jsonEncoder := json.NewEncoder(bf)
    jsonEncoder.SetEscapeHTML(false)
    jsonEncoder.Encode(htmlJson)
    fmt.Println("第二种解决办法：", bf.String())
```