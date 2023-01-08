---
title: go module版本选择
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["golang"]
tags: ["golang"]
---

# go module版本选择
#技术/golang/版本控制

最小版本选择算法
最小版本选择算法用于选择build中使用的所有modules的版本。对于build中的每个module，通过最小版本选择算法选择的版本始终是主module或其依赖项之一通过require指令显式列出的版本中语义最高的版本，简单来说就是：使用require中的最高版本。

例如：如果你的module，依赖的module A需要require D v1.0.0，依赖的module B需要require D v1.1.1，最小版本选择算法将会选择D的v1.1.1版本用于build。即使后期D有v1.2.0版本可用，未显式require至go.mod中，仍将使用v1.1.1版本用于build。

[浅谈Go Modules原理](https://duyanghao.github.io/golang-module/)


Minimal version selection也即最小版本选择，如果光看上述的引用可能会很迷惑(或者矛盾)：明明是选择最新的版本(keep only the newest version)，为什么叫最小版本选择？
我对最小版本选择算法中’最小’的理解如下：
* 最小的修改操作
* 最小的需求列表
* 最小的模块版本。这里比较的对象是该模块的最新版本：如果项目需要依赖的模块版本是v1.2，而该模块实际最新的版本是v1.3，那么最小版本选择算法会选取v1.2版本而非v1.3(为了尽可能提高构建的稳定性和重复性)。也即’最小版本’表示项目所需要依赖模块的最小版本号(v1.2)，而不是该模块实际的最小版本号(v1.1)，也并非该模块实际的最大版本号(v1.3)
这里，我们举例子依次对Go Modules最小版本选择的算法细节进行阐述：

## 构建依赖列表
[Algorithm 1: Construct Build List](https://research.swtch.com/vgo-mvs#algorithm_1) 
[image:832B7162-7849-42CE-BE19-E599088ED0F9-49645-0000D8876DD25237/init_eg.png]

简单来说可以通过图遍历以及递归算法(图递归遍历)来构建依赖列表。从根节点出发依次遍历直接依赖B1.2以及C1.2，然后递归遍历。这样根据初始的依赖关系(指定版本：A1->B1.2，A1->C1.2)，会按照如下路径选取依赖：
[image:633F190F-F726-48FD-9557-EFD799251C89-49645-0000D8876D7CAFF3/version-select-2.png]
首先构建empty build list，然后从根节点出发递归遍历依赖模块获取rough build list，这样rough build list中可能会存在同一个依赖模块的不同版本(如D1.3与D1.4)，通过选取最新版本构建final build list(最终的构建列表一个模块只取一个版本，即这些版本中的最新版)，如下：
[image:F4D5296C-DE2C-4EAA-A899-721D9092FAEA-49645-0000D8876D1A47C8/version-select-list.png]
```go
module A1
go 1.14
require (
	B1.2
	C1.2
	D1.4
	E1.2
)
```


## 升级所有依赖
  [Algorithm 2. Upgrade All Modules](https://research.swtch.com/vgo-mvs#algorithm_2) 
一次性升级所有模块(直接&间接)可能是对构建列表最常见的修改，go get -u 就实现了这样的功能。当执行完这个命令后，所有依赖包都将升级为最新版本，如下(Algorithm 1例子基础上进行升级)：
[image:A58B660B-E0ED-48CE-8A4F-EA4C9FF6DB53-49645-0000D908B7E1C0AB/version-select-3.png]

这里面添加了新的依赖模块：E1.3，G1.1，F1.1以及C1.3。新rough build list会将新引入的依赖模块和旧的rough build list模块(黄色部分)进行合并，并从中选取最大版本，最终构建final build list(上图红线标识模块)。为了得到上述结果，需要添加一些依赖模块到A的需求列表中，因为按照正常的构建流程，依赖包不应该包括D1.4和E1.3，而是D1.3和E1.2。这里面就会涉及 [Algorithm R. Compute a Minimal Requirement List](https://research.swtch.com/vgo-mvs#algorithm_r) ，该算法核心是创建一个能重复构建依赖模块的最小需求列表，也即只保留必须的依赖模块信息(比如直接依赖以及特殊依赖)，并存放于go.mod文件中。比如上述例子中A1的go.mod文件会构建如下：

```go
module A1
go 1.14
require (
	B1.2
	C1.3
	D1.4 // indirect
	E1.3 // indirect
)
```
可以看到上述go.mod文件中，没有出现F1.1以及G1.1，这是因为F1.1存在于C1.3的go.mod文件中，而G1.1存在于F1.1的go.mod文件中，因此没有必要将这两个模块填写到A1的go.mod文件中；而D1.4和E1.3后面添加了indirect标记，这是因为D1.4和E1.3都不会出现在B1.2，C1.3以及它们的依赖模块对应的go.mod文件中，因此必须添加到模块A1的需求列表中(go需要依据这个列表中提供的依赖以及相应版本信息重复构建这个模块，反过来，如果不将D1.4和E1.3添加到go.mod，那么最终模块A1的依赖构建结果就会是D1.3以及E1.2)
另外，这里也可以总结出现indirect标记的两种情况：
* A1的某个依赖模块没有使用Go Modules(也即该模块没有go.mod文件)，那么必须将该模块的间接依赖记录在A1的需求列表中
* A1对某个间接依赖模块有特殊的版本要求，必须显示指明版本信息(例如上述的D1.4和E1.3)，以便Go可以正确构建依赖模块

## 升级某一个依赖
  [Algorithm 3. Upgrade One Module](https://research.swtch.com/vgo-mvs#algorithm_3) 
相比一次性升级所有模块，比较好的方式是只升级其中一个模块，并尽量少地修改构建列表。例如，当我们想升级到C1.3时，我们并不想造成不必要的修改，如升级到E1.3以及D1.4。这个时候我们可以选择只升级某个模块，并执行go get命令如下(Algorithm 1例子基础上进行升级)：
go get C@1.3
[image:EB6ABDB4-3355-42BC-848E-DBE6C53D5E37-49645-0000D92BB4237F25/version-select-4.png]
当我们升级某个模块时，会在构建图中将指向这个模块的箭头挪向新版本(A1->C1.2挪向A1->C1.3)并递归引入新的依赖模块。例如在升级C1.2->C1.3时，会新引入F1.1以及G1.1模块(对一个模块的升级或者降级可能会引入其他依赖模块)，而新的rough build list(红线)将由旧rough build list(黄色部分)与新模块(C1.3，F1.1以及G1.1)构成，并最终选取其中最大版本合成新的final build list(A1，B1.2，C1.3，D1.4，E1.2，F1.1以及G1.1)
注意最终构建列表中模块D为D1.4而非D1.3，这是因为当升级某个模块时，只会添加箭头，引入新模块；而不会减少箭头，从而删除或者降级某些模块。比如若从 A 至 C 1.3 的新箭头替换了从 A 至 C 1.2 的旧箭头，升级后的构建列表将会丢掉 D 1.4。也即是这种对 C 的升级将导致 D 的降级(降级为D1.3)，这明显是预料之外的，且不是最小修改
一旦我们完成了构建列表的升级，就可运行前面的算法 R 来决定如何升级需求列表(go.mod)。这种情况下，我们最终将以C1.3替换C 1.2，但同时也会添加一个对D1.4的新需求，以避免D的意外降级(因为按照算法1，D1.3才会出现在最终构建列表中)。如下：
```go
module A1
go 1.14
require (
	B1.2
	C1.3
	D1.4 // indirect
)
```

