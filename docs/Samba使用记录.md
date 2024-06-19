 参考资料链接

* ==[安装教程](https://ubuntu.com/tutorials/install-and-configure-samba#1-overview)==
* ==[推荐博客](https://blog.csdn.net/l1593572468/article/details/121444812)==
* ==[更换用户](https://blog.csdn.net/m0_38112165/article/details/110474609)==
* ==[权限管理](https://blog.51cto.com/u_14349334/3485557)==

注意：

Since Samba doesn’t use the system account password, we need to set up a Samba password for our user account:

sudo smbpasswd -a username

Note:

Username used must belong to a system account, else it won’t save.

Samba账号一定要是系统拥有的账号。

***共享配置：***

```
[temp]        #共享资源名称，也就是浏览时显示的名字

comment = Temporary file space  #简单的解释，内容无关紧要

path = /tmp    #实际的共享目录

writable = yes  #设置为可写入

browseable = yes #可以被所有用户浏览到资源名称，

guest ok = yes  #可以让用户随意登录

public = yes           #允许匿名查看

valid users = 用户名   #设置访问用户

valid users = @组名   #设置访问组

readonly = yes      #只读

readonly = no       #读写

hosts deny = 192.168.0.0    #表示禁止所有来自192.168.0.0/24 网段的IP 地址访问

hosts allow = 192.168.0.24 #表示允许192.168.0.24 这个IP 地址访问
```

***用户管理：***

```
pdbedit -a username    #新建Samba账户。

pdbedit -x username    #删除Samba账户。

pdbedit -L                     #列出Samba用户列表，读取passdb.tdb数据库文件。

pdbedit -Lv                   #列出Samba用户列表详细信息。

pdbedit -c “[D]” -u username   #暂停该Samba用户账号。

pdbedit -c “[]” -u username     #恢复该Samba用户账号。

sudo service smbd restart    #重启smb

sudo ufw allow samba           

\\ip-address\sambashare
```