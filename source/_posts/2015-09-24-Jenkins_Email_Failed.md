---
title: Jenkins_email_failed
date: 2015-09-24
tags: Jenkins
---

### 报错信息：

> 

**Failed to send out e-mail**
com.sun.mail.smtp.SMTPSendFailedException: 501 mail from address must be same as authorization user; nested exception is: com.sun.mail.smtp.SMTPSenderFailedException:
**501 mail from address must be same as authorization user**

### 解决方法：

在设置Jenkins URL底下有一个文本框System Admin e-mail address， 这里要设置发送者的邮箱地址。

