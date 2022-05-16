---
title: 科汛cms9.0通用版源码审计
copyright: true
date: 2022-05-16 16:40:42
tags: 源码审计
password: wybzdd@
message: 需要密码才能看这篇文章。
wrong_pass_message: 多试几次吧。
---
Kesioncms Version9.0通用版  asp.net/sqlserver
<!--more-->

<font size=5>1.后台登录界面爆破绕过验证码</font>  
后台地址为 manage/login.aspx  
当密码和验证码同时错误时提示  
![1.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/1.png)  
当仅密码错误时提示  
![2.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/2.png)  
也算一个比较常见的逻辑问题了，直接用bp爆破常见密码即可，但在实际测试过程中，部分版本存在登录次数限制。  

<font size=5>2.信息泄露</font>  
这个问题和服务器配置有关系  
在游客状态下可访问  
/UploadFiles/temp/  
/config/  

这个洞看具体情况，比如在某站点下发现了大量用户注册信息  
![3.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/3.png)  
<font size=5>3.后台任意文件上传</font>  
漏洞描述  
科汛cms v9通用版后台存在任意文件上传  
漏洞详情  
后台存在抓取远程图片功能  
![4.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/4.png)    
Admin/Include/SaveBetondfile.aspx中调用Button1_Click方法  
![5.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/5.png)    
在Button1_Click方法中未对文件后缀名做检查，造成任意文件上传  
![6.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/6.png)    
效果展示  
1.上传asp一句话成功  
![7.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/7.png)    
2.菜刀连接getshell  
![8.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/8.png)     

<font size=5>4.前台下载系统逻辑问题</font>  
漏洞描述  
科汛网络系统V9前台下载系统存在逻辑漏洞  
![9.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/9.png)     
当下载的文件需要收费时，使用余额支付时，系统仅检测用户账户金额是否大于0，账户金额大于0，可下载任意价格的文件。  
![10.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/10.png)     
使用余额支付跳转到指定预设的下载地址  
![11.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/11.png)     
![12.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/12.png)     
漏洞分析  
1.抓包分析访问/index.aspx?c=download&M_id=3&ID=3735&DownID=1  
跟进index.aspx  
![13.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/13.png)     
2.跟进Kesion.NET.WebSite.Index  Page_Load方法，调用Kesion.Controllers.downloadController Run方法  
![14.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/14.png)     
3.跟进Kesion.Controllers.downloadController，调用了Charge.MoneyInOrout()  
![15.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/15.png)    
4.Charge继承了ICharge接口，在Kesion.SqlDAL.Charge实现  
其中对结算后金额为负的情况验证存在逻辑错误，默认扣费时传入参数InorOutFlag为2，MustIn为1，当money-Money<0时，无法返回false  
![16.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/16.png)    
![17.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/17.png)    
继续执行，完成扣费操作  
![18.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/18.png)    
![19.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/19.png)    
SQL跟踪:  
![20.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/20.png)    
网页显示:  
![21.png](https://lockcy-github-io.vercel.app/2022/05/16/%E7%A7%91%E6%B1%9Bcms9-0%E9%80%9A%E7%94%A8%E7%89%88%E6%BA%90%E7%A0%81%E5%AE%A1%E8%AE%A1/21.png)    
