### 1、安装NFS服务
`sudo apt-get install nfs-kernel-server`

### 2、新建目录及修改权限
`sudo mkdir /home/xx/linux/nfs`

`sudo chmod 777 /home/xx/linux/nfs/`

### 3、配置NFS服务
sudo vi /etc/exports

在最后添加如下内容

```
/home/xx/linux/nfs   *(rw,sync,no_root_squash)
```

/home/xx/linux/nfs 表示 NFS 共享的目录

*表示允许所有的网络段访问

rw 表示访问者具有可读写权限

sync 表示将缓存写入设备中，可以说是同步缓存的意思

no_root_squash 表示访问者具有 root 权限。

### 4、重启 NFS 服务器
`sudo /etc/init.d/nfs-kernel-server restart`

### 5、查看 NFS 共享目录

`showmount -e`