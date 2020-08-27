---
layout:     post
title:      shell tool
subtitle:   shell工具
date:       2020-08-27
author:     Static
header-img: 
catalog: true
tags:
    - shell
    
---

### 1. git 一键提交代码

三步合一

```bash
git add -all
git commit -m 'xxx'
git push
```

**git_push.sh**

```bash
#!/bin/bash

# git add -all ; git commit -am "优化" ; git push 合成一个命令

add_flag=`git status | grep 'Untracked'`
if [[ ${add_flag}!="" ]]
then
	git add --all
fi

commit_log=$1

if [[ ! -n "${commit_log}" ]]; then
	commit_log="优化"
fi

git commit -am "${commit_log}"

git push

```

> 可在 ~/.bash_profile 中配置 alias 别名

```bash
alias push='sh /Users/xxx/Documents/shell/git_push.sh'

# 执行提交
$ push '提交代码'
```

### 2. 一键执行c成语

> mac 执行c程序，先编译，再执行

两步合一

**run_c.sh**

```bash
#!/usr/bin/env bash

# --- run_c test.c  直接编译运行 --- #

# 获取目录
path=`pwd`

c_filename=$1 # c文件
c_target=${c_filename} # c编译文件

# 判断输入是否以 .c 结尾，补充.c
if [[ !c_filename =~ '.c'$ ]]
then
    c_target=$1
    c_filename="${c_filename}.c"
else
    c_target=${c_target/%.c/}
fi

# echo "c_filename:${c_filename}"
# echo "c_target:${c_target}"

# 判断是否存在
if [[ ! -f ${path}/${c_filename} ]]
then
    echo "${path}/${c_filename} does not exist"
    exit
fi

# 判断debug文件是否存在
if [[ ! -d ${path}/../debug ]]
then
    echo ${path}/../debug" does not exist"
    exit
fi

# 编译到 ../debug 路径中
`gcc -o ${path}/../debug/${c_target} ${path}/${c_filename}`

# 执行C程序
${path}/../debug/${c_target}
```

> ~/.bash_profile 中配置 alias 别名

```bash
alias run_c='sh /Users/didi/Documents/shell/run_c.sh'

# 执行c
$ run_c hello.c
```