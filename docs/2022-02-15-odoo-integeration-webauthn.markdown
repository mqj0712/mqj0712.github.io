---
layout: default
title:  "odoo与webauthn集成"
date:   2022-02-15 12:12:12 +0800
categories: odoo webauthn
---
# 背景

最近由于业务需要, 要求在odoo中支持FIDO2的认证方式, 即在用户账户绑定FIDO2兼容的**Authenticator**(通常为USB key), 需要用户在正常的帐号密码登录之后, 使用USB key做二次身份确认. 

# 调查研究

当前项目使用的odoo版本是[14.0 community 版本](https://nightly.odoo.com/14.0/nightly/), 而OCA 的[14.0 版本的server-auth 项目](https://github.com/OCA/server-auth/tree/14.0)并不包含FIDO2 或者U2F的认证支持, 不过在 [12.0 版本](https://github.com/OCA/server-auth/tree/12.0) 中发现有 [auth_u2f 插件](https://github.com/OCA/server-auth/tree/12.0/auth_u2f) 实现, 所以决定从改造 12.0 版本的`auth_u2f` 开始, 最终目标是**FIDO2** 与 **odoo14** 的集成

# 过程

整体的改造过程如下:

## 准备工作

由于FIDO2认证需要网站工作在**TLS/HTTPS** 下, 所以本地测试需要准备:

1. 通过openssl生成网站需要的证书和私钥:`certificate.pem` 以及 `key.pem`, 方法请*自行~~百~~(goo)~~度~~(gle)* , 在nginx配置中会用到  
2. 搭建一个nginx反向代理, 关键的配置如下, 其中的 `proxy_set_header X-Forwarded-HTTPS on` 是解决odoo redirect 操作导致浏览器使用回HTTP请求的问题

    ```nginx
        server {
            listen 443 ssl;
            proxy_set_header X-Forwarded-Proto $scheme;
            ssl_certificate /usr/local/nginx/cert/certificate.pem; #cert
            ssl_certificate_key /usr/local/nginx/cert/key.pem; #key
            ssl_session_timeout  5m;    #time out
            ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;    #supported algorithm
            ssl_protocols TLSv1 TLSv1.1 TLSv1.2;    #supported tls version
            ssl_prefer_server_ciphers on;   #use server prefered
            ...

                location / {
                proxy_pass http://localhost:8069;
                proxy_set_header X-Forwarded-HTTPS on;
            }
        
        }
    ```


3. 在odoo的系统参数中配置`web.base.url` 以及 `web.base.url.freeze`
    
    web.base.url -> https://localhost  
    web.base.url.freeze -> True  

完成之后可以测试新的https . 接下来就可以开始改造认证部分

## auth_u2f -> odoo14

这部分工作主要集中在 auth_u2f 到odoo 14  
1. 下载auth_u2f插件到本地的插件目录  
2. 按照插件的说明安装依赖包 `pip install python-u2flib-server`  
3. 去掉方法上不支持的装饰器 `@api.multi`  
4. 按照标准插件安装方式安装插件

不出其他意外的话, auth_u2f 应该可以在系统中运行起来, 唯一遇到的一个问题是, 给当前用户新绑定usb key的时候, 会导致当前session出现登录失败的情况, 可能导致设备绑定出现问题, 修改方式也很简单, 修改`controller` ,如果发现用户已经登录, 不强制验证密钥

{% highlight python %}
    @http.route('/web/u2f/login', type='http', auth='none', sitemap=False)
    def u2f_login(self, u2f_token_response=None, redirect=None, **kw):
        user = request.env['res.users'].browse(request.session.uid)
        if user:
            return http.redirect_with_hash(self._login_redirect(
                user.id, redirect=redirect))   

{% endhighlight %}

测试后发现在 Firefox中能工作正常, 不过在chrome 中发现js错误, 由**u2f-api.js** 代码引起, 报`extension: kmendfapggjehodndflmmgagdbamhnfd` 不存在, 这个扩展应该是chrome 内置的扩展, 调查后发现应该和chrome放弃U2F支持有关 [Deprecate U2F API (Cryptotoken)](https://developer.chrome.com/blog/deps-rems-95/#deprecate-u2f-api-cryptotoken), sighhhh! 还是Firefox耿直,所以接下来势必要转战 FIDO2  

## FIDO2 in odoo 14

和U2F的认证方式很类似, 包含注册部分

![webauthn_reg](/assets/images/2021/2/webauthn_reg.png)  

和认证部分

![webauthn_auth](/assets/images/2021/2/webauthn_auth.png)  

改造步骤如下:
1. 替换对应的后端和前端U2F支持库到FIDO2的支持库, 这里选用的是 [py_webauthn][py_webauthn] (server) 以及 [SimpleWebAuthn][SimpleWebAuthn] (client) , 安装方式请参考对应项目文档.  
2. 后端  
    1. device的模型中增加 `credential` (可选), `credential_id` , 以及 `credential_pub_key` 字段
    2. 在设备注册response verify通过后, 从对应的verify_registration_response 中解析出`credential_id` 和 `credential_pub_key` 并保存到对应的device record中
    3. 在登录时从device record中拿到creddential_id, 作为 allow_credentials 放到AuthenticationOption中
    4. 验证时从device记录中拿`credential_pub_key`做验证

3. 前端的改动就比较简单, 替换auth_u2f.js 和auth_uf_frontend.js中对应的u2f-api的register 和 sign 方法为SimpleWebAuthn的startRegistration 和 startAuthentication即可

至此chrome 和 firefox都可以支持

# 相关网站连接

Some useful links

- **awesome-webauthn**: [https://github.com/herrjemand/awesome-webauthn](https://github.com/herrjemand/awesome-webauthn) , 包含了很多关于WebAuthn的资源连接
- **py_webauthn**: [https://github.com/duo-labs/py_webauthn](https://github.com/duo-labs/py_webauthn) 本文最终使用的 **webauthn server** 库
- **SimpleWebAuthn**: [https://github.com/MasterKale/SimpleWebAuthn](https://github.com/MasterKale/SimpleWebAuthn) 本文最终使用的 **webauthn client** 库
- **Auth0**: [https://webauthn.me/](https://webauthn.me/) Demo网站, 可以直观了解FIDO2的认证流程
- **谈谈WebAuthn**: [https://flyhigher.top/develop/2160.html](https://flyhigher.top/develop/2160.html)


[py_webauthn]: https://github.com/duo-labs/py_webauthn
[SimpleWebAuthn]: https://github.com/MasterKale/SimpleWebAuthn