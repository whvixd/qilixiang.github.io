---
layout:     post
title:      git console setting
subtitle:   记录
date:       2020-06-11
author:     Static
header-img: 
catalog: true
tags:
    - git
    
---

> 记录Mac控制台中如何设置git命令自动补全和git命令颜色功能，git常用命令 -> [如意门](http://whvixd.com/2017/08/20/git/)

## 1. git command completion setting

#### 1. clone `git-completion.bash` 到本地

```shell
curl https://raw.githubusercontent.com/git/git/master/contrib/completion/git-completion.bash -o ~/.git-completion.bash 
```

#### 2. 修改 `~/.bash_profile`

> 添加以下命令

```shell
# 判断文件是否存在
if [ -f ~/.git-completion.bash ]; then 
# 若存在，则执行
. ~/.git-completion.bash 
fi 
```

#### 3. 重新加载生效 `.bash_profile` 文件

```shell
source ~/.bash_profile
```

#### 4. 效果图

<html>
    <img src="/img/tool/git-completion.png" width="500" height="500" /> 
</html>

---

## 2. git console color setting

#### 1. `~/.bash_profile` 添加以下命令

```shell
find_git_branch () {
local dir=. head
until [ "$dir" -ef / ]; do
if [ -f "$dir/.git/HEAD" ]; then
head=$(< "$dir/.git/HEAD")
if [[ $head = ref:\ refs/heads/* ]]; then
git_branch=" (${head#*/*/})"
elif [[ $head != '' ]]; then
git_branch=" → (detached)"
else
git_branch=" → (unknow)"
fi
return
fi
dir="../$dir"
done
git_branch=''
}

PROMPT_COMMAND="find_git_branch; $PROMPT_COMMAND"

black=$'\[\e[1;30m\]'
red=$'\[\e[1;31m\]'
green=$'\[\e[1;32m\]'
yellow=$'\[\e[1;33m\]'
blue=$'\[\e[1;34m\]'
magenta=$'\[\e[1;35m\]'
cyan=$'\[\e[1;36m\]'
white=$'\[\e[1;37m\]'
normal=$'\[\e[m\]'
PS1="$white[$white@$green\h$white:$cyan\W$yellow\$git_branch$white]\$ $normal"

```

#### 2. 重新加载生效 `.bash_profile`文件

```shell
source ~/.bash_profile
```

#### 3. 效果图

<html>
    <img src="/img/tool/git-color-setting.png" width="500" height="500" /> 
</html>

> 添加以上配置，终端的界面是不是看起来更舒服了 😆