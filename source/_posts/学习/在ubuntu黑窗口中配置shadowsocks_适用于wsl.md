---
title: 在ubuntu黑窗口终端中配置shadowsocks(适用于wsl)
date: 2023-8-21
update: 2023-8-21
tag: 
  - linux
  - shell
categories:
  - study
---

# 在ubuntu黑窗口中配置shadowsocks(适用于wsl)

#### 起因: 

* 最近想学ftxui美化一下项目终端, 但是发现这个东东由于win下C++ STL threads库和mutex库的问题, 只能在linux or macOS下运行. 但是在WSL ubuntu中运行时, 由于没有开代理, cmake构建中fetchcontent超级慢, 于是用了一整天的时间搞清楚了如何在黑窗口中配置shadowsocks代理

#### 说明: 

1. 版本:
   * ubuntu-22.04 (WSL)
   * python 3.8 + pip(不建议过高)
   * shadowsocks-valoroso 3.0.7(不建议过低)
2. shadowsocks比较敏感, 不知道文章会不会被禁, 且看且珍惜

#### 步骤:

1. 在windows系统安装v2rayN获取节点参数
     github官方链接: https://github.com/2dust/v2rayN/releases/download/5.22/v2rayN-Core.zip/
       下载解压后运行v2rayN.exe，然后点击软件顶部”订阅“按钮，选择”订阅设置“，在”地址 url“里粘贴你的订阅地址，点”确定“。
       再点v2rayN里的”订阅“，再点”更新订阅“，这时v2rayN里会拉取到你的节点，双击对应节点，即可打开对应节点的参数，只需查看里面的地址、端口、密码3个参数。

2. Linux中安装python2.8

     需要使用python2.8及以下版本

     这样操作:

     ```shell
     # 先看一下你的python默认版本
     python3 -V
     # 我的是Python 3.10, 需要下载3.8
     #安装依赖包
     sudo apt update
     sudo apt install software-properties-common
     #添加 deadsnakes PPA 源
     sudo add-apt-repository ppa:deadsnakes/ppa
     #安装 python 3.8
     sudo apt install python3.8
     #再看一下版本
     python3.8 -V
     #这时候应该成功了, 
     #然后配置 python3.8 为系统默认 python3，将 python 各版本添加到 update-alternatives
     which python3.8
     # 输出: /usr/bin/python3.8
     sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.8 1
     which python3.10
     # 输出: /usr/bin/python3.10
     sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.6 2
     # 配置 python3 默认指向 python3.8
     sudo update-alternatives --config python3
     # 输出: 
     # here are 2 choices for the alternative python3 (providing /usr/bin/python3).
     #Selection Path Priority Status
     #------------------------------------------------------------
     #* 0 /usr/bin/python3.10 2 auto mode
     #
     #1 /usr/bin/python3.10 2 manual mode
     #
     #2 /usr/bin/python3.8 1 manual mode
     #
     #Press to keep the current choice[*], or type selection number: 
     # 选择输入2, 回车
     # 测试 python 版本
     python3 -V
     # 输出: Python 3.8.2
     # 安装pip
     sudo apt install python3-pip
     # 这个地方如果报错, 基本上都是包下载的不全, 使用sudo apt install把缺失的包下载了就ok
     ```

3. 下载shadowsocks:

     可以上pipy官网搜索shadowsocks下载最新版本

     我下载的是: shadowsocks-valoroso 3.0.7, 一些旧版本可能不支持新的加密算法, 建议越新越好

     ```shell
     pip install shadowsocks-valoroso
     ```

4. 创建节点参数文件

     创建一个 /etc/shadowsocks.json 文件，格式如下：

     ```json
     {
         "server":"步骤一里获得的IP或域名",
         "server_port":步骤一里获得的端口号,
         "local_address": "127.0.0.1",
         "local_port":1080,
         "password":"步骤一里获得的密码",
         "timeout":300,
         "method":"chacha20-ietf-poly1305",
         "fast_open": false
     }
     ```

5. 启动代理

     ```shell
     /usr/local/bin/sslocal -c /etc/shadowsocks.json -d start       #启动
     /usr/local/bin/sslocal -c /etc/shadowsocks.json -d restart    #重启
     /usr/local/bin/sslocal -c /etc/shadowsocks.json -d stop        #停止运行
     ```

     检测是否代理成功

     ```shell
     # 通过以下命令查看结果是否为SS代理的IP，如果是，说明代理成功。（代理IP并非节点IP）
     curl ip.sb --socks5 127.0.0.1:1080
     ```

6. proxychains

     安装proxychains

     ```shell
     sudo apt-get install proxychains
     ```

     需要把/etc/proxychains.conf 最后一行改成:

     ```shell
     socks5 127.0.0.1 1080
     ```

     并确保proxy_dns没有被注释掉

     命令行需要代理时，只需在前面加上 proxychains 即可，例如：

     ```shell
     proxychains curl -4 ip.sb       #示例 通过此命令可以获得并显示代理IP
     proxychains npm install       #示例
     proxychains wget  https://xxx.aaa.com/bbb.sh    #示例
     # 即：在常规命令行前加上proxychains即可！
     ```

7. 最后我写了一个shell脚本

     ~/bashss/wallkiller.sh

     ```shell
     #!/bin/bash
     
     echo "sslocal 启动代理:"
     sudo /usr/local/bin/sslocal -c /etc/shadowsocks.json -d start
     echo "##############
     检测是否代理成功"
     curl ip.sb --socks5 127.0.0.1:1080
     if [[ $? == 0 ]] ; then
         echo "success"
         echo "##############
     proxychains链接代理"
         proxychains curl -4 ip.sb
         if [[ $? == 0 ]] ; then
             echo "wall killed successfully"
         else
             echo "sorry something was wrong!!!"
         fi
     else
         echo "fail"
     fi
     ```

     以后可以直接运行此脚本, 不用一个一个敲命令了

     
