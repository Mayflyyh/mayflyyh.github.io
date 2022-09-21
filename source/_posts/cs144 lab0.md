---
title: CS144 Lab0 warmup
date: 2022-09-09 11:55:19
categories:
- CS144
tags: 
- lab 
- network

---

热身

<!-- more -->

# CS144: Lab Checkpoint 0: networking warmup

## Fetch a Web page

run 

```bash
telnet cs144.keithw.org http
GET /hello HTTP/1.1
Host: cs144.keithw.org
Connection: close 

```

Remember to hit the Enter key one more time .This sends an empty line and tells the server that you are done with your HTTP request. 

## Send yourself an email

因为没有斯坦福的邮箱，所以用了QQ邮箱。

Run: 

```bash
telnet smtp.qq.com smtp
```

Type:`HELo qq.com`

```
250-newxmesmtplogicsvrsza8.qq.com-9.21.160.46-9278235
250-SIZE 73400320
250 OK
```

Type:`auth login`

```
334 VXNlcm5hbWU6
```

VXNlcm5hbWU6 是 Username:  的base64编码

将邮箱名转换为base64形式后输入，密码（授权码）亦然。

按照邮件报文格式（RFC:882）

```
MAIL FROM:<1273961325@qq.com>
RCPT TO:<yyh073@foxmail.com>
DATA
From: 1273961325@qq.com 
To: yyh073@foxmail.com 
Subject: Hello from CS144 Lab 0! 

Hello! This is Body.

.
```



##  Listening and connecting 

Run `netcat -v -l -p 9090 `

Then in another terminal windows, run `telnet localhost 9090`



##  Writing a network program using an OS stream socket 

###  Writing webget 

一定要好好阅读文档



##  An in-memory reliable byte stream 

利用内存传输流

