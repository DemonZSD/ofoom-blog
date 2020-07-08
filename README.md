# 搭建hexo个人博客
## 安装git 和 nodejs
### 安装git

```
$yum install -y git
```

### 安装nodejs

```
$curl -sL https://rpm.nodesource.com/setup_10.x | sudo bash -
$sudo yum install nodejs 
```
**NOTE:** 高版本的nodejs已经默认安装npm，无需单独安装npm

验证：
```
$node --version
v10.21.0
$npm --version
6.14.4
```

## 安装hexo
### 安装hexo

```
$nmp install -g hexo-cli
```

### 初始化文件夹
```
$hexo init <forder>
$cd <forder>
$npm install #在此文件夹安装hexo所有的依赖模块
```

## 安装插件

### 安装git部署的插件

```
$npm install hexo-deployer-git --save
```

### 设置 hexo 发布方式（后续会用ofoom blog进行覆盖，此步骤也可以省略）

##### 在根目录修改_config.yml文件
```
$ vim _config.yml
```
###### 设置发布方式
```
...
deploy:
  type: git
  repo: https://gitee.com/DemonZSD/DemonZSD.git
  branch: master
...
```
###### 设置语言
```
...
title: ofoom
subtitle: ''
description: ''
keywords:
author: DemonZSD
language: zh-Hans
timezone: ''
...

```


## 安装 ofoom blog
### 覆盖 ofoom blog 

```
git clone https://github.com/DemonZSD/ofoom-blog.git
cp DemonZSD/* <forder>
```

## 部署测试
### 生成静态页面
```
$hexo clean
$hexo generate
```

### 开启本地服务, 默认端口4000
```
$hexo server 
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.

```

### 发布到 git Pages 或者 gitee Pages
根据在_config.yml中配置的deploy类型，进行发布

```
$hexo deploy  # 执行此命令后，会输入 github 或者 gitee 的用户名和密码
```

在github Pages 页镜像静态文件的部署