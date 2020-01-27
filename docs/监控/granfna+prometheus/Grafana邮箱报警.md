# Grafana邮箱报警

## 一 163 邮箱的设置

### 1.1 开启163邮箱的开启POP3/SMTP/IMAP服务

**网易邮箱已经默认开启**[**POP3/SMTP/IMAP**](http://help.163.com/07/0112/11/34KPH9KD007525G1.html)**服务，方便您可以通过电脑客户端软件更好地收发邮件，如果关闭可以通过以下方式打开。**

开启POP3/SMTP/IMAP服务方法

请登录[163邮箱](http://email.163.com/)，点击页面右上角的“设置”—在“高级”下，点“POP3/SMTP/IMAP”，勾选图中两个选项，点击确定。即可开启成功。开通后即可用[闪电邮](http://help.163.com/09/1218/14/5QQS7HIA00753VB8.html)、[Outlook](http://help.163.com/09/1222/17/5R5GPV6C00753VB8.html)等软件收发邮件了。

![1580046505086](D:\workspace\note\docs\监控\文档图片需要上传\1580046505086.png)

### 1.2 163邮箱报错: 535 Error: authentication failed
发件邮箱异常信息，SMTP功能已经开通，错误信息为：535 Error: authentication failed。经过网上查证，原来新的163邮箱代码传递的密码不再是登陆密码，更换为客户端授权密码。更换授权密码后，可正常发送。

### 1.3 设置客户端授权密码

![1580046597808](D:\workspace\note\docs\监控\文档图片需要上传\1580046597808.png)

我这里设置的客户端授权密码是fish939598604

## 二. 配置grafana

### 2.1 修改grafana配置文件

grafana的配置文件默认是在`/etc/grafana/grafana.ini`,修改配置文件如下

windows 的是grafana-6.5.2.windows-amd64\grafana-6.5.2\conf\defaults.ini

```
[smtp]
enabled = true
host = smtp.163.com:25
user = 15989287971@163.com
# If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
password = fish939598604
#cert_file =
#key_file =
skip_verify = true
from_address = 15989287971@163.com
from_name = Grafana
#ehlo_identity =

[emails]
#welcome_email_on_sign_up = false
#templates_pattern = emails/*.html

[alerting]
# Makes it possible to turn off alert rule execution.
execute_alerts = true
```

### 2.2 重启grafana服务      

```
sudo service grafana-server restart
```

### 2.3 web页面配置增加alert

![1580046597808](D:\workspace\note\docs\监控\文档图片需要上传\20171228113822523.png)

![1580046597808](D:\workspace\note\docs\监控\文档图片需要上传\20171228114045968.png)

![1580046597808](D:\workspace\note\docs\监控\文档图片需要上传\20200126222944.png)

send test测试，查看939598604@qq.com是否收到邮件

右上角发送成功提示，不成功请检查配置或网络

