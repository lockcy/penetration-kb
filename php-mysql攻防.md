---
title: php mysql攻防
copyright: true
date: 2022-06-14 00:09:01
tags: [渗透测试,php]
---
^-^
<!--more-->
<h1> mysql （5.5之前）</h1>  

1、
php.ini 配置 magic_quotes_gpc = On  
开启魔术引号开关后，输入数据（ _GET _POST _COOKIE 等）中含单引号（ ' ）、双引号（ " ）、反斜线（ \ ）与 NULL（NULL 字符）等字符，都会被加上反斜线转义
addslashes 函数用于在上述符号前添加反斜杠转义，与 gpc 的作用大致相同  

在仅使用 gpc 或 addslashes 函数的情况下  
```
function check_input($value)
{
    if(!get_magic_quotes_gpc()){
        $value = addslashes($value);
    }
    return $value;
}
```
情况一： 数字型注入  
```
SELECT * FROM user WHERE id=-1 or 1=1 --
Array ( [0] => 1 [1] => test [2] => test@qq.com )
```
修复方法  
```
$id  = intval($_GET['id']);
```
情况二： 宽字节注入  
mysql 在使用 gbk 编码时，会将两字节认作一个汉字；当 %df%27 和 gpc/addslashes 相遇，%27 前会添加转义字符 %5c ，%df 和 %5c 组成了新的汉字 %df%5c，单引号 %27 则逃逸出来  
```
// 设置编码
mysql_query("SET NAMES 'gbk'");
```
```
username=admin1%df%27%20or%201=1%20--%20-
SELECT * FROM user WHERE username='admin1運' or 1=1 -- -'
Array ( [0] => 1 [1] => test [2] => test@qq.com )
```
修复方法  
```
使用过滤函数：
mysql_real_escape_string
设置以二进制形式连接：
mysql_query("SET character_set_connection=gbk, character_set_results=gbk,character_set_client=binary");
```

2、  
情况一： 宽字节注入  
mysql_real_escape_string（此方法在 php5.5 后不被建议使用，在 php7 中废除）  
同样可在编码设置有问题的情况下用宽字节绕过  
需要设置二进制连接避免该问题  
```
mysql_query("SET character_set_connection=gbk, character_set_results=gbk,character_set_client=binary");
```
情况二：二次注入  
……  

<h1> mysqli </h1>  
在 php5.5 后  mysql 等方法被废弃，更提倡使用 mysqli 来连接操作 mysql 数据库。  

1、  
转义方式的安全做法  
```
function check_input($conn, $value)
{
// 去除斜杠
    if (get_magic_quotes_gpc())
    {
        $value = stripslashes($value);
    }
// 如果不是数字则加引号
    if (!is_numeric($value))
    {
        $value = "'" . mysqli_real_escape_string($conn, $value) . "'";
    }
    return $value;
}
```
2、  
在 mysql 5.1 后，提供了类似于 jdbc 的预处理-参数化查询。它的查询方法是：    
a. 先预发送一个 sql 模板过去  
b. 再向 mysql 发送需要查询的参数  
就好像填空题一样，不管参数怎么注入， mysql 都能知道这是变量，不会做语义解析，起到防注入的效果，这是在 mysql 中完成的。  
 mysqli 默认在服务端预处理  
```
<?php

$host='localhost';
$dbName='testdb';
$user='root';
$pass='123456';
$mysqli = mysqli_connect($host,$user,$pass,$dbName);

if(mysqli_connect_errno())
{
    echo mysqli_connect_error();
}

$stmt = $mysqli->prepare("select * from user where username=?");

$stmt->bind_param("s",$_GET['username']);

$stmt->execute();

$res = $stmt->get_result();
$row = $res->fetch_assoc();
print_r($row);
$stmt->close();

mysqli_close($mysqli);
```
<h1> PDO </h1>  
```
<?php

$pdo = new PDO("mysql:host=localhost;dbname=testdb;charset=utf8",'root','123456');
$pdo->exec('set names utf8');
$id = $_GET['id'];
$sql = "select * from user where id = ?";
$statement = $pdo->prepare($sql);
$statement->bindParam(1, $id);
$statement->execute();


$row = $statement->fetch();
print_r($row);
```

PHP5.3.6 之前，使用 PDO 预处理过程如下：      
![1.png](https://lockcy-github-io.vercel.app/2022/06/14/php-mysql%E6%94%BB%E9%98%B2/1.png)    
以一种本地预处理的方式向远程服务器提交查询信息，信息还是在本地组装成字符串后向mysql服务器请求；使用 $pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false); 禁止本地预处理无效。  
查阅一些资料发现问题所在：有些驱动不支持或有限度地支持本地预处理。如果驱动不能成功预处理当前查询，它将总是回到模拟预处理语句上。在 win 下无法禁止本地预处理，在 linux 下可以禁止。  

在本地预处理生效且有编码问题的情况下，仍有受到宽字节注入的危险：  
```
<?php
$pdo = new PDO("mysql:host=127.0.0.1;dbname=testdb;charset=utf8",'root','123456');
// $pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES,false);

$sql = "select * from user1 where uid = ?";
$statement = $pdo->prepare($sql);
$pdo->exec('set names gbk');

$id = $_GET['id'];
$statement->bindParam(1, $id);

$statement->execute();
$row = $statement->fetch();
print_r($row);
```
```
id=%df%27%20or%201=1%20--%20-
```


php5.3.6 及之后的版本预处理过程如下  
默认还是在本地预处理，但如果禁用本地预处理的情况会先传送模板，再传送数据，如下图所示    
![2.png](https://lockcy-github-io.vercel.app/2022/06/14/php-mysql%E6%94%BB%E9%98%B2/2.png)  
禁用本地预处理的可以有效防止宽字节注入。  


<h1>小结</h1>  

**注：以上测试均在 windows 下 php5 环境下测试**  
**php版本小于 5.3.6(5.2.17) ：**  
**无法关闭本地预处理，易受宽字节注入影响， linux 下可关闭本地预处理（未测试）**  
**php版本大于 5.3.6(5.4.45) ：  
**可关闭本地预处理，在关闭本地预处理的情况下不受宽字节注入影响**  
测试结果与网上一些资料有点出入，有时间会深入再看看  


**最好设置 $pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES,false);  避免本地预处理**  
**若使用 php 5.3.6+ , 请在在 PDO 的 DSN 中指定 charset 属性**  
**宽字节注入危害还是很大的**   


https://segmentfault.com/a/1190000008117968  
https://www.iteye.com/blog/zhangxugg-163-com-1835721  
https://xz.aliyun.com/t/3950  
https://www.freebuf.com/articles/web/216336.html  
