# 简介
docker都是通过命令来操作容器，使用idea插件可以减少重复命令输入等。

# 使用步骤
> Idea内安装插件

打开`Idea,Preferences | Plugins`

进入插件安装界面，在搜索框中输入docker，可以看到Docker integration，点击右边的Install按钮进行安装,安装后重启Idea。
![image.png](https://raw.githubusercontent.com/purgeyao/TechnologyArticle/master/A-IMG/docker-plugins.png)

> 配置插件

重启后配置docker，连接到远程docker服务，打开配置界面。

路径: `Preferences | Build, Execution, Deployment | Docker`
![image.png](https://raw.githubusercontent.com/purgeyao/TechnologyArticle/master/A-IMG/docker-plugins-Set.png)

点击+号添加一个docker配置，输入Name和Engine API URL，URL是docker服务地址。

如果连接本机docker 选择`Docker for Mac`。

连接其他机器则选择`TCP socket`
```
tcp://<ip>:<端口>
# 例子
tcp://47.106.13.224:2375
```

> 开启docker远程连接

可能出现异常如下：
![image.png](https://raw.githubusercontent.com/purgeyao/TechnologyArticle/master/A-IMG/further%20information%2039.10%20375.png)

需要docker开启远程连接功能。CentOS中在docker启动参数里添加如下配置即可开启远程连接。

```
# 允许所有客户端连接
-H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375
```

docker 提供了远程控制API，采用的是restful风格。centos7开启方式： 
```
vim /lib/systemd/system/docker.service
```
找到 ExecStart行修改为:
```
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
```

重启docker
```
systemctl daemon-reload
systemctl restart docker
```

> 其他方式：

大家可以产考这个文章：[Docker 远程连接 -- dockerd 命令详解](https://cloud.tencent.com/developer/article/1047265)

在 /etc/docker/daemon.json （下文统一简称 daemon.json）中写入以下内容
```
{
  "hosts":[
    "unix:///var/run/docker.sock",
    "tcp://0.0.0.0:2375"
  ]
}
```
> idea docker 控制台

以上步骤完成就可以使用idea docker console了。

功能包含：日志查看，镜像启动停止等。
![image.png](https://raw.githubusercontent.com/purgeyao/TechnologyArticle/master/A-IMG/docker-run-p.png)
