---
title: git 常见问题
date: 2023-01-08T18:06:57+08:00
draft: false
categories: [""]
tags: [""]
---

# git 常见问题
#git

```
“Unable to find remote helper for 'https'” during git clone
```

安装curl-devel然后重新编译git
```sh
$ yum install curl-devel
$ # cd to wherever the source for git is
$ cd /usr/local/src/git-1.7.9  
$ ./configure
$ make
$ make install
```

git升级v2版本
```sh
$ yum remove git
注意，需要把git依赖的包删掉，例如go，不然的话yum删不掉

$ wget https://www.kernel.org/pub/software/scm/git/git-2.0.0.tar.gz --no-check-ce
rtificate

$ tar -xzvf git-2.0.0.tar.gz

$ cd git-2.0.0

$ ./configure

$ make

$ make install
```

# git撤销merge
git merge —abort