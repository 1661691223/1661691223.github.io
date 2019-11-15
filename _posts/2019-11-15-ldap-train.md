---
layout: post
title: 内部ldap实践
categories: 内部ldap实践
description: 内部ldap实践
keywords: ldap
---

## 安装OPEN-LDAP

## yum安装（首先需要epel源）

```
yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel migrationtools
```

##  安装完了之后可以直接启动`OpenLDAP`服务，不需要做任何配置，

```
systemctl start slapd.service
systemctl enable slapd.service
```

## 目录结构

\[root@10-0-234-86 ~\]# cd /etc/openldap/
\[root@10-0-234-86 openldap\]# tree
.
├── certs
│   ├── cert8.db
│   ├── key3.db
│   ├── password
│   └── secmod.db
├── check_password.conf
├── ldap.conf
├── schema
│   ├── collective.ldif
│   ├── collective.schema
│   ├── corba.ldif
│   ├── corba.schema
│   ├── core.ldif
│   ├── core.schema
│   ├── cosine.ldif
│   ├── cosine.schema
│   ├── duaconf.ldif
│   ├── duaconf.schema
│   ├── dyngroup.ldif
│   ├── dyngroup.schema
│   ├── inetorgperson.ldif
│   ├── inetorgperson.schema
│   ├── java.ldif
│   ├── java.schema
│   ├── misc.ldif
│   ├── misc.schema
│   ├── nis.ldif
│   ├── nis.schema
│   ├── openldap.ldif
│   ├── openldap.schema
│   ├── pmi.ldif
│   ├── pmi.schema
│   ├── ppolicy.ldif
│   └── ppolicy.schema
└── slapd.d
    ├── cn=config
    │   ├── cn=schema
    │   │   ├── cn={0}core.ldif
    │   │   ├── cn={1}cosine.ldif
    │   │   ├── cn={2}nis.ldif
    │   │   └── cn={3}inetorgperson.ldif
    │   ├── cn=schema.ldif
    │   ├── olcDatabase={0}config.ldif
    │   ├── olcDatabase={-1}frontend.ldif
    │   ├── olcDatabase={1}monitor.ldif
    │   └── olcDatabase={2}hdb.ldif
    └── cn=config.ldif

5 directories, 42 files

　　/etc/openldap/slapd.conf：OpenLDAP的主配置文件，记录根域信息，管理员名称，密码，日志，权限等

　　/etc/openldap/slapd.d/*：这下面是/etc/openldap/slapd.conf配置信息生成的文件，每修改一次配置信息，这里的东西就要重新生成

　　/etc/openldap/schema/*：OpenLDAP的schema存放的地方

　　/var/lib/ldap/*：OpenLDAP的数据文件

　　/usr/share/openldap-servers/DB_CONFIG.example 模板数据库配置文件

　　/usr/share/openldap-servers/slapd.ldif 模板配置文件

　　OpenLDAP监听的端口：

　　默认监听端口：389（明文数据传输）

　　加密监听端口：636（密文数据传输）

## 初始化OpenLDAP的配置

这一块在最一开始是最麻烦的部分，网上所有教程讲的都不对。因为现在是`2019`年了，而很多教程还停留在`2008`年甚至`1998`年。配置`OpenLDAP`最正确的姿势是通过`ldapmodify`命令执行**一系列自己写好的**ldif文件，而**不要修改任何OpenLDAP装好的配置文件**。

初始化的步骤就是修改CN，DC，DC，添加管理密码

```
[root@10-0-234-86 openldap]#slappasswd
```

输入两次密码就能生成一个密文密码

此时编辑文件modify\\{2\\}hdb.ldif （这里文件名不固定，只是为了方便找自己的ldif文件,olcRootPW密码就是刚刚生成的加密密码，直接复制过来）

```
[root@10-0-234-86 ~]# cat modify\{2\}hdb.ldif
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=ZStack,dc=com
-
replace: olcSuffix
olcSuffix: dc=ZStack,dc=com
-
replace: olcRootPW
olcRootPW: {SSHA}//qyWNLsd0Uvbaefrt6locLTEbzyCNOX
```

编辑完成执行命令

```
ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f modify\{2\}hdb.ldif
```

这么长的命令是什么意思？`-Q`表示安静执行，`-Y`和后面的`EXTERNAL`表示，好吧，我也不知道什么意思（这个真没百度到），总之需要这样配合，然后`-H`表示地址，`-f`表示文件名。几乎所有的`ldapmodify`命令都这么执行就好了。

继续编辑文件modify{1}monitor.ldif

```
[root@10-0-234-86 ~]# cat modify{1}monitor.ldif
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=extern
 al,cn=auth" read by dn.base="cn=Manager,dc=ZStack,dc=com" read by * none
```

编辑完成执行命令

```
ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f modify\{1\}monitor.ldif
```

ldapmodify执行，就是操作了/etc/openldap/slapd.d/底下的文件，具体修改的文件就如同问加盟命名方式一样咯，至于为什么不能vim操作这个文件，我做了以下对比，vim会更改 olcDatabase={1}monitor.ldif 和 olcDatabase={2}hdb.ldif两个文件的文件校验值。

1.  `changetype`就是`modify`，表示我们要修改这个文件。第`3`行是`replace`，表示我们要替换里面的某个值，你可以把这个操作理解为`mysql`数据库的`update`操作，如果你把第`3`行改成`add`，那就是`mysql`的`insert`操作了。不过这里我们操作的只是配置文件本身，还牵涉不到添加用户或者更改用户，如果你以为事情就这么简单，那就是你太天真了。
2.  `RootDN`在这里就表示你整个`OpenLDAP`系统的管理员用户名是什么，不要奇怪，后面这一砣都是用户名`cn=Manager,dc=ZStack,dc=com。`

## 导入配置数据库

```
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown ldap:ldap /var/lib/ldap/*
```

## 导入一些基本schema

　　默认已经导入了core.schema

```
[root@10-0-234-86 ~]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=cosine,cn=schema,cn=config"

[root@10-0-234-86 ~]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=nis,cn=schema,cn=config"

[root@10-0-234-86 ~]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=inetorgperson,cn=schema,cn=config"
```

创建用户

```
[root@10-0-234-86 ~]# cat base.ldif
dn: dc=ZStack,dc=com
o: ZStack.io
dc: ZStack
objectClass: top
objectClass: dcObject
objectclass: organization

dn: cn=Manager,dc=ZStack,dc=com
cn: Manager
objectClass: organizationalRole
description: Directory Manager

dn: ou=People,dc=ZStack,dc=com
ou: People
objectClass: top
objectClass: organizationalUnit

dn: ou=Group,dc=ZStack,dc=com
ou: Group
objectClass: top
objectClass: organizationalUnit

[root@10-0-234-86 ~]# ldapadd -x -w "123456" -D "cn=Manager,dc=ZStack,dc=com" -f /root/base.ldif
adding new entry "dc=ZStack,dc=com"

adding new entry "cn=Manager,dc=ZStack,dc=com"

adding new entry "ou=People,dc=ZStack,dc=com"

adding new entry "ou=Group,dc=ZStack,dc=com"

[root@10-0-234-86 ~]# ldapadd -x -w "123456" -D "cn=Manager,dc=ZStack,dc=com" -f /root/user.ldif

[root@10-0-234-86 ~]# cat user.ldif
dn: uid=test,ou=People,dc=Zstack,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: test
sn: test
userPassword: 123456
loginShell: /bin/bash
uidNumber: 10007
gidNumber: 10002
homeDirectory: /home/test
mail: xxx@xxx.io
```

## 测试

```
[root@10-0-234-86 ~]# ldapsearch -H ldap://127.0.0.1 -x -b "ou=People,dc=ZStack,dc=com"
# extended LDIF
#
# LDAPv3
# base <ou=People,dc=ZStack,dc=com> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# People, ZStack.com
dn: ou=People,dc=ZStack,dc=com
ou: People
objectClass: top
objectClass: organizationalUnit

# test, People, ZStack.com
dn: uid=test,ou=People,dc=ZStack,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: test
sn: test
userPassword:: MTIzNDU2
loginShell: /bin/bash
uidNumber: 10007
gidNumber: 10002
homeDirectory: /home/test
mail: xxx@xxx.io
uid: test

# search result
search: 2
result: 0 Success

# numResponses: 3
```

##  查看结果

![](https://oscimg.oschina.net/oscnet/9fc57504ddccd895c93e9d20102181c7a2d.jpg)
