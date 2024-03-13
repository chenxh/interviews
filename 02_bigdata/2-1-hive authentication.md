## hive 的认证机制
hive 支持 authentication 认证机制，可通服务端参数
```
hive.server2.authentication
```
在实际应用中常用以下3种：
* NONE：即不做身份校验；系统默认是 NONE
* LDAP: 使用基于 LDAP/AD 的用户身份校验；
* KERBEROS: 使用 Kerberos/GSSAPI 做身份校验；

### 不认证场景

* 用户通过各种客户端如 cli/gui/java 登录时，可以不配置用户名和密码, 在服务端 Hive 会认为登录的是匿名用户 anonymous,如：如：beeline -u jdbc:hive2://xx.xx.xx.xx:10000/default
* 用户通过各种客户端如 cli/gui/java 登录时，也可以配置为任意用户名和任意密码,在服务端 Hive 会认为登录的是用户声明的任意用户（用户名可以是任意用户名，甚至是不存在的用户名；密码可以是任意密码，或不配置密码），如：beeline -u jdbc:hive2://xx.xx.xx.xx:10000/default -n xyz；beeline -u jdbc:hive2://xx.xx.xx.xx:10000/default -n xyz -p xxxx
* 可以通过 hiveserver2 webui，查看登录的用户身份；

### ldap 认证
```
hive.server2.authentication = ldap
hive.server2.authentication.ldap.url=url
```
客户端登录 ldap 认证的 hiveserver2 时，需要提供用户名和密码，hiveserver2 会到ldap中验证用户名和密码，只有验证通过后才能正常登录；
beeline 登录
```
beeline -u jdbc:hive2://xx.xx.xx.xx:10000/default -n hs_cic -p xyzabc;
```

### kerberos  认证
在开启了 kerberos 安全认证的大数据集群环境中，HIVE既可以配置使用 kerberos 认证机制，也可以配置使用 LDAP 认证机制；

其配置方式比较简单，配置参数 hive.server2.authentication = kerberos/ldap 即可；

不过在使用方式上，有不少容易犯错的细节，需要强调下。

4.1 kerberos 环境下，hive 的 kerberos 认证方式: hive.server2.authentication = kerberos
由于是在kerberos环境下，所以客户端在登录前，需要首先从 kdc 获取 ticket 并维护在 ticket cache中：a valid Kerberos ticket in the ticket cache before connecting：

如果是cli等客户端，一般会通过命令 kinit principal_name -kt key_tab_location，基于 keytab 文件来获取特定业务用户的 ticket，并存储在客户端的 ticket cache中；（如果缓存的 ticket 过期了，需要通过命令重新获取；如果不使用keytab, 也可以通过密码来获取 ticket: kinit principal_name）；

如果是程序代码，则一般通过 org.apache.hadoop.security.UserGroupInformation.loginUserFromKeytab(String user, String path) 的方式，基于keytab文件来获取特定业务用户的 ticket，并存储在客户端的 ticket cache中；（UserGroupInformation 在后台会自动基于keytab 文件来定时刷新ticket，确保不会过期）；

客户端在获取业务用户的 ticket 成功后，才可以通过 jdbc连接，登录到指定的 hiveserver2：

此时需要特别注意下 hiveserver2 的url的格式，其格式推荐使用：jdbc:hive2://xx.xx.xx.xx:10000/default;principal=hive/_HOST@CDH.COM：

这里的principal部分，推荐使用三段式来指定，包含pincipal, host 和 realm;

pincipal 必须指定为系统用户hive，而不能是业务用户如 dap,xyz等（本质上是因为，hive-site.xml 中配置的hive系统用户是hive）；

host部分，推荐指定为_HOST，此时在底层使用时会替换为 hiveserver2 节点的hostname （当然也可以直接指定为 hiveserver2 节点的具体的 hostname）；

realm 部分，需要根据实际配置情况进行指定（可以查看配置文件 /etc/krb5.conf）；

4.2 kerberos环境下，hive 的 LDAP 认证方式 : hive.server2.authentication = ldap
由于是在kerberos环境下，所以客户端在登录前，需要首先从 kdc 获取 ticket 并维护在 ticket cache中，这一点跟 kerberos 环境下，hive 的 kerberos 认证方式时一直的：a valid Kerberos ticket in the ticket cache before connecting：

如果是cli等客户端，一般会通过命令 kinit principal_name -kt key_tab_location，基于 keytab 文件来获取特定业务用户的 ticket，并存储在客户端的 ticket cache中；（如果缓存的 ticket 过期了，需要通过命令重新获取；如果不使用keytab, 也可以通过密码来获取 ticket: kinit principal_name）；

如果是程序代码，则一般通过 org.apache.hadoop.security.UserGroupInformation.loginUserFromKeytab(String user, String path) 的方式，基于keytab文件来获取特定业务用户的 ticket，并存储在客户端的 ticket cache中；（UserGroupInformation 在后台会自动基于keytab 文件来定时刷新ticket，确保不会过期）；

客户端在获取业务用户的 ticket 成功后，才可以通过 jdbc连接，登录到指定的 hiveserver2，此时登录格式，跟非 kerberos 环境下，hive 的 ldap认证方式，是一样的：

客户端登录 ldap 认证的 hiveserver2 时，需要提供用户名和密码，hiveserver2 会到ldap中验证用户名和密码，只有验证通过后才能正常登录；

以 beeline 登录为例，其命令格式如下：beeline -u jdbc:hive2://xx.xx.xx.xx:10000/default -n hs_cic -p xyzabc;
