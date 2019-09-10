# 简介nfs
nfs网络文件系统常用于共享音视频，图片等静态资源。将需要共享的资源放到NFS里的共享目录，通过服务器挂载实现访问。
# 服务端安装:

```
yum install -y nfs-utils rpcbind
```
或者
```
yum install -y nfs-utils
yum install -y rpcbind
```
# 客户端安装:
```
yum install -y nfs-utils
```
设置开机自启动

```
systemctl enable nfs
systemctl enable rpcbind
```

# 服务端配置:

## 1. 创建共享目录

```
mkdir -p /data/nfs-share
```

## 2. 安装完nfs服务一般会自动生成配置文件exports，如果没有就自己创建一个 /etc/exports

```
cat /etc/exports
```
## 3.编辑共享配置文件

```
vi /etc/exports
/data/nfs-share *(rw,sync,no_root_squash)


#/home/nfs *(rw,sync,no_root_squash)
#/data/nfs-share *
```
第一列：欲共享出去的目录，也就是想共享到网络中的文件系统；
```
第二列：可访问主机
192.168.152.13      指定IP地址的主机
nfsclient.test.com  指定域名的主机
192.168.1.0/24      指定网段中的所有主机
*.test.com          指定域下的所有主机
*                   所有主机
```

第三列：共享参数
下面是一些NFS共享的常用参数：
```
 ro                      只读访问
 rw                      读写访问
 sync                    所有数据在请求时写入共享
 async                   NFS在写入数据前可以相应请求
 secure                  NFS通过1024以下的安全TCP/IP端口发送
 insecure                NFS通过1024以上的端口发送
 wdelay                  如果多个用户要写入NFS目录，则归组写入（默认）
 no_wdelay               如果多个用户要写入NFS目录，则立即写入，当使用async时，无需此设置。
 Hide                    在NFS共享目录中不共享其子目录
 no_hide                 共享NFS目录的子目录
 subtree_check           如果共享/usr/bin之类的子目录时，强制NFS检查父目录的权限（默认）
 no_subtree_check        和上面相对，不检查父目录权限
 all_squash              共享文件的UID和GID映射匿名用户anonymous，适合公用目录。
 no_all_squash           保留共享文件的UID和GID（默认）
 root_squash             root用户的所有请求映射成如anonymous用户一样的权限（默认）
 no_root_squas           root用户具有根目录的完全管理访问权限
 anonuid=xxx             
```
指定NFS服务器/etc/passwd文件中匿名用户的UID
例如可以编辑/etc/exports为：
```
/tmp　　　　    　*(rw,no_root_squash)
/home/public    　192.168.0.*(rw)　　 *(ro)
/home/test　    　192.168.0.100(rw)
/home/linux　     *.the9.com(rw,all_squash,anonuid=40,anongid=40)
```


## 4. 启动nfs

```
service rpcbind start
# 提示: Redirecting to /bin/systemctl start rpcbind.service
service nfs start
# 提示: Redirecting to /bin/systemctl start nfs.service
```

## 5. 查看挂载

```
showmount -e 127.0.0.1
```
```
返回内容
# Export list for 127.0.0.1:
# /data/nfs-share *
```


# 客户端配置:
## 1. 创建

```
# /kubernetes 为本机挂载的目录
mkdir -p /kubernetes
```

## 2. 挂载
```
mount [服务端ip]:/data/nfs-share /kubernetes
# 例如
mount 10.1.1.99:/home/nfs /kubernetes
```
# 其他

配置生效
```
# 修改共享配置文件执行
exportfs -r
```