---
layout:     post
title:      发送邮件
subtitle:   Send email util
date:       2018-06-12
author:     Static
header-img: img/bg/black.jpg
catalog: true
tags:
    - Java
---

### 利用Apache的commons-email-1.4.jar实现发送邮件的功能

#### 1. 首先需要依赖的jar包

```
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-email</artifactId>
    <version>1.4</version>
</dependency>
        
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.16.18</version>
</dependency>
```

#### 2. 添加发送邮件的工具类，简单的封装了下

```java
package com.github.whvixd.util;

import lombok.Getter;
import lombok.Setter;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.mail.EmailException;
import org.apache.commons.mail.SimpleEmail;

import java.text.SimpleDateFormat;
import java.util.Date;

/**
 * Created by wangzhx on 2018/6/11.
 */
@Slf4j
public class SendEmail {

    public static void toWho(String toEmail, String nickName, String title, String content) {
        toWho(toEmail, nickName, title, content, () -> FromUser.USER);
    }

    //通过listener可以设置发送邮件服务器
    public static void toWho(String toEmail, String nickName, String title, String content, IChangeListener listener) {
        FromUser fromUser = listener.change();
        if (StringUtils.isBlank(fromUser.getHost())) {
            fromUser.setHost(FromUser.USER.getHost());
        }

        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyy-MM-dd HH:mm:ss");
        StringBuilder msgInfo = new StringBuilder();
        msgInfo.append(content).append("\r\n\t");
        msgInfo.append("时间：").append(simpleDateFormat.format(new Date()));

        try {
            SimpleEmailSingleton.INSTANCE
                    .setHostName(fromUser.getHost())
                    .setAuthentication(fromUser.getUserName(), fromUser.getPassword())
                    .setCharset("UTF-8")
                    .addTo(toEmail)
                    .setFrom(fromUser.getUserName(), nickName)
                    .setSubject(title + "-" + simpleDateFormat.format(new Date()))
                    .setMsg(msgInfo.toString())
                    .send();
            log.info("发送人:{},收件人:{},发送邮箱成功", fromUser.getUserName(), toEmail);
        } catch (EmailException e) {
            log.error("发给{}邮件失败", toEmail);
            throw new RuntimeException("发送邮箱失败！");
        }

    }

    //构造者模式，封装发送邮件工具类
    enum SimpleEmailSingleton {
        INSTANCE;

        private static final SimpleEmail simpleEmail = new SimpleEmail();

        SimpleEmailSingleton setHostName(String hostName) {
            simpleEmail.setHostName(hostName);
            return this;
        }

        SimpleEmailSingleton setAuthentication(String userName, String password) {
            simpleEmail.setAuthentication(userName, password);
            return this;
        }

        SimpleEmailSingleton setCharset(String newCharset) {
            simpleEmail.setCharset(newCharset);
            return this;
        }

        SimpleEmailSingleton addTo(String email) throws EmailException {
            simpleEmail.addTo(email);
            return this;
        }

        SimpleEmailSingleton setFrom(String email, String name) throws EmailException {
            simpleEmail.setFrom(email, name);
            return this;
        }

        SimpleEmailSingleton setSubject(String aSubject) {
            simpleEmail.setSubject(aSubject);
            return this;
        }

        SimpleEmailSingleton setMsg(String msg) throws EmailException {
            simpleEmail.setMsg(msg);
            return this;
        }

        SimpleEmailSingleton send() throws EmailException {
            simpleEmail.send();
            return this;
        }
    }

    public enum FromUser {
        //主机服务,邮箱账号,授权码
        USER("smtp.163.com", "whvixd@163.com", "***");

        @Setter
        @Getter
        private String host;

        @Setter
        @Getter
        private String userName;

        @Setter
        @Getter
        private String password;


        FromUser(String host, String userName, String password) {
            this.host = host;
            this.userName = userName;
            this.password = password;
        }
    }

    @FunctionalInterface
    public interface IChangeListener {
        FromUser change();
    }

}


```

#### 3. 测试发送邮箱

```java
package com.github.whvixd;

import com.github.whvixd.util.SendEmail;

/**
 * Created by wangzhx on 2018/3/29.
 */
public class Test {

    @org.junit.Test
    public void sendEmail() {
        SendEmail.toWho("***@***.com",
                "七里香", "Title",
                "1. qilixiang test send email");
    }

}

```

**最后成功收到邮件**

![目录结构](/img/SendEamil/email1.png)

#### 4. 使用IChangeListener设置发送邮件服务器

```java
package com.github.whvixd;

import com.github.whvixd.util.SendEmail;

/**
 * Created by wangzhx on 2018/3/29.
 */
public class Test {

    @org.junit.Test
    public void sendEmail() {
        SendEmail.toWho("***@***.com", "七里香",
                "Title", "1. qilixiang test send email",
                () -> {
                    SendEmail.FromUser fromUser = SendEmail.FromUser.USER;
                    fromUser.setHost("smtp.qq.com");//不设置服务器，默认为163服务器
                    fromUser.setUserName("631791301@qq.com");
                    fromUser.setPassword("***");//邮箱授权码
                    return fromUser;
                });
    }

}
```


> 注意：若使用163邮箱作为发送人，要在163的个人首页中设置授权码(用于登录第三方邮件客户端的专用密码)

![目录结构](/img/SendEamil/163_1.jpg)

![目录结构](/img/SendEamil/163_2.jpg)