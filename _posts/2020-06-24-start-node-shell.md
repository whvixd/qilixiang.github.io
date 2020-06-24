---
layout:     post
title:      启动node项目脚本
subtitle:   
date:       2020-06-24
author:     Static
header-img: 
catalog: true
tags:
    - shell
    
---

> 前端项目启动依赖node的版本，每次启动自己手动切换太麻烦，这里推荐用脚本来切换和启动node项目

## 1. 环境

#### 1. 依赖 `nvm`

> nvm和n都是node版本管理工具,这里用nvm来管理

**nvm两种安装方式**

```shell
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.0/install.sh | bash

wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.0/install.sh | bash
```

**nvm 命令**

```shell
nvm ls-remote # 列出所有可以安装的node版本号
nvm install v10.4.0 # 安装指定版本号的node
nvm use v10.3.0 # 切换node的版本，这个是全局的
nvm current # 当前node版本
nvm ls # 列出所有已经安装的node版本
```

---

#### 2. 依赖 `npm`

> Nodejs下的包管理器

**npm 安装**

```shell
brew install node
```

**npm 命令**

```shell
npm install # 安装模块
npm uninstall # 卸载模块
npm update # 更新模块
npm outdated # 检查模块是否已经过时
npm ls # 查看安装的模块
npm init # 在项目中引导创建一个package.json文件
npm help # 查看某条命令的详细帮助
npm root # 查看包的安装路径
npm config # 管理npm的配置路径
npm cache # 管理模块的缓存
npm start # 启动模块
npm stop # 停止模块
npm restart # 重新启动模块
npm test # 测试模块
npm version # 查看模块版本
npm view # 查看模块的注册信息
npm adduser # 用户登录
npm publish # 发布模块
npm access # 在发布的包上设置访问级别
```

> 依赖 `yarn` , 安装：`npm install -g yarn`

---

## 3. 脚本介绍

```bash
#!/bin/bash

# 进入项目路径
cd /Users/xxx/Documents/workspace/webstorm/***

#  yarn 淘宝镜像配置，若公司有自己镜像，修改为公司的镜像
yarn_registry=`yarn config get registry`
if test ${yarn_registry} != "http://registry.npm.taobao.org/"
then
	`yarn config set registry http://registry.npm.taobao.org/`
	echo "switch registry to http://registry.npm.taobao.org/ success"
fi

# nvm 环境配置
export NVM_DIR="/Users/xxx/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# 判断或切换node版本
node_version=`nvm current`
if test ${node_version} != "v12.16.3"
then
	nvm use v12.16.3
fi

# 依赖包安装
yarn install

# 另启线程监测node是否启动，启动后打开首页
{
while true
do
# 根据端口号判断项目是否启动
pid=`lsof -i:9090|awk '{print $2}'|grep 'PID'`
echo ${pid}
# 如果项目启动，则打开浏览器
if   [[ ${pid} != "" ]]
then
# 用Google打开网址
open -a "Google Chrome.app" http://baidu.com
break
else
# 项目没有启动，睡眠5秒
sleep 5
fi
done
} &

# 启动项目，npm启动时间较长，所以在之前添加线程监测项目是否启动
npm run dev
```

---

## 4. 命令启动

#### 1. 打开 `.bash_profile`

```shell
vi ~/.bash_profile
```

#### 2. 添加 `alias`

```shell
alias start_node='sh /Users/xxx/Documents/shell/start_node.sh' # 添加别名
```

#### 3. 终端输入 `start_node` 启动项目

```shell
start_node # 启动项目
```