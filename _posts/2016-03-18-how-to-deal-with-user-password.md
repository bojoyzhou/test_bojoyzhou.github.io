---
layout: default
title: 该如何处理用户的密码
category: tech
tags: [user, password, 用户登录, 用户密码, 密码存储]
---

作为一名软件工程师, 一定对用户登录再熟悉不过了, 但是大家真的熟悉吗? 大家的用户密码是如何存储的呢? 下面我分享一下我的做法.

### 阶段1(明文存储)

第一次想要做一个用户登录的时候, 我想的很简单, 把用户名和密码存到数据库里面就可以了. 很快的, 这第一个版本也就完成了.

> 如果你是一个用户, 当你知道你所登录的网站的密码是这样存储的时候, 你如何想? 

### 阶段2(混淆存储)

介于阶段1的弊端, 阶段2必然要对存储的密码做一些混淆, 这个时候还不知道什么是散列算法, 只能对用户输入的密码做一些混淆, 然后验证密码的时候把混淆的密码还原, 然后做比较. 这样可以避免网站管理员直接看到密码

> 一段时间我都觉得是没问题的, 知道有人告诉我, 这个混淆不安全, 因为是可逆的

### 阶段3(数字签名)

这一阶段主要对上一阶段的加强, 系统学习了MD5, SHA1, SHA2等签名算法, 本质是算法过程不可逆(或者说极难可逆)

> 后来听说了一个东西叫做[彩虹表](http://baike.baidu.com/link?url=V4yOa3EzwZhaD2R0UMaT8vlTm7B7XEFqLInshZ4_nKsREGpwoHdV0xbaCYeimR58dwyliMooDw0k3IQva7L_r_), 它可以交快速的碰撞出加密后签名的源串

### 阶段4(密码加盐)

黑客可以预先把可能出现的字符序列进行MD5计算,然后把计算结果存储在一个密码表里面. 破解的时候把数字签名传进去, 然后查表即可得到源串. 彩虹表的原理比这个稍复杂些, 主要是优化了存储空间, 使得上面的表变小. 介于这个原因, 之前做的散列其实就不那么有效了. 解决方案是先将用户密码进行混淆,再进行签名. 这样黑客的查表就无效了, 因为查到的是我们混淆之后的密码.

_盐可以认为是一个字符串, 把这个字符串加到用户密码后面, 或者插到中间, 或者其他破坏源串的操作都称之为加盐, 一般的做法是md5(source+salt), 其中source+salt就是一个加盐操作_

> 由于我们对用户密码混淆的方式都是一致的, 一旦黑客了解了这种混淆方式, 那么他就可以在构建表的时候先进行混淆, 然后再构建表. 那么一个用户被破解也就代表着全栈的用户都被破解了.

### 阶段5(动态加盐)

既然如此, 那么我就让每个用户的密码混淆方式不同. 因为构建密码表是一件非常费时的工作, 黑客基本不可能针对每个用户进行构建密码表. 这里涉及到另一个算法[HMAC](http://baike.baidu.com/view/1136366.htm), 从学术的角度上对算法强度做了解释.

> 这个基本就是最终的方案了

### 其他

除此之外在看mysql手册的时候发现了另外一种做法, 也非常奇特. 我们上面的方案中都是把用户的密码作为值来进行散列然后存储, 这里的这个方案相反. 这里是把一个固定的值作为值, 把用户的密码作为密钥来处理. 下面是伪码(其实是sql).

加密过程

        //password是用户密码
        @password = 'My secret passphrase';
        //先把password进行SHA2签名
        @key_str = SHA2(@password,512);
        //init\_vector可以认为是一个盐
        @init_vector = RANDOM_BYTES(16);
        //对text进行加密
        @crypt_str = AES_ENCRYPT('text',@key_str,@init_vector);

验证过程

        //如果解密之后能够获得text, 那么认为密码是正确的, 如果得到的是空串(即解密失败), 则认为密码是错误的
        AES_DECRYPT(@crypt_str,@key_str,@init_vector);
