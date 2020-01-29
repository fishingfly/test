# registry拉取dockerhub私有镜像



### 准备阶段

目标是制作registry可用于加速dockerhub私有镜像的加速器。

需要准备的工具：

1、registry镜像，最新的就行

2、两台可通信的虚拟机（pc机就行）

测试项：

1、是否可以正常拉取dockerhub私有镜像

2、私有镜像保存在这样的仓库中是否安全（其他用户是否也可以不经过认证获取到私有镜像）

基于以上的情况，开始搭建测试环境：

config.yml配置文件：

```text
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
proxy:
  remoteurl: https://registry-1.docker.io
  username: ##填你Dockerhub的账号##
  password: ##填你自己dockerhub密码，没有就去注册一个##
```

Dockerfile

```text
FROM registry:latest
LABEL maintainer="zhounanjun <zhounanjun@126.com>"
COPY entrypoint.sh /entrypoint.sh
COPY config.yml /etc/docker/registry/config.yml
```

entrypoint.sh

```text
#!/bin/sh
​
set -e
​
CONFIG_YML=/etc/docker/registry/config.yml
​
if [ -n "$PROXY_REMOTE_URL" -a `grep -c "$PROXY_REMOTE_URL" $CONFIG_YML` -eq 0 ]; then
    echo "proxy:" >> $CONFIG_YML
    echo "  remoteurl: $PROXY_REMOTE_URL" >> $CONFIG_YML
    echo "------ Enabled proxy to remote: $PROXY_REMOTE_URL ------"
elif [ $DELETE_ENABLED = true -a `grep -c "delete:" $CONFIG_YML` -eq 0 ]; then
    sed -i '/rootdirectory/a\  delete:' $CONFIG_YML
    sed -i '/delete/a\    enabled: true' $CONFIG_YML
    echo "------ Enabled local storage delete -----"
fi
case "$1" in
    *.yaml|*.yml) set -- registry serve "$@" ;;
    serve|garbage-collect|help|-*) set -- registry "$@" ;;
esac
​
exec "$@"
​
```

Makefile

```text
VERSION ?= v1.0
​
image:
        docker build -t ecloud-cis/registry-mirror:${VERSION} .
run-test-registry:
        docker run -itd -p 7990:5000 -e PROXY_REMOTE_URL=https://registry-1.docker.io  --restart=always  --name registry-mirror-test-2 ecloud-cis/registry-mirror:${VERSION}
​
```

### 搭建测试环境

在你准备上面这些文件时，你就运行以下命令：

```text
make image
make run-test-registry
```

就可以运行期一个用于缓存的registry

docker ps 看下运行起的容器：

![image-20200120181322303](file://C:/Users/fishing/AppData/Roaming/Typora/typora-user-images/image-20200120181322303.png?lastModify=1580206302)

docker logs -f registry-mirror-test-2 //查看刚起的regsitry的日志

看到如下listen:5000字样，那就是启动成功了

![image-20200120181443599](file://C:/Users/fishing/AppData/Roaming/Typora/typora-user-images/image-20200120181443599.png?lastModify=1580206302)

到测试机上更改daemon.json,daemon.json文件一般在/etc/docker下，要是没有就新建一个

改成如下：里面的地址是你起的registry加速器的地址，

```text
{
        "registry-mirrors": ["http://10.154.12.120:7990"]
}
```

运行： systemctl daemon-reload

systemctl restart docker

systemctl status docker 看下Dockers的状态是不是running

然后运行docker system info //看下docker的信息：

```text
Containers: 5
 Running: 4
 Paused: 0
 Stopped: 1
Images: 18
Server Version: 18.09.5
Storage Driver: overlay2
 Backing Filesystem: extfs
 Supports d_type: true
 Native Overlay Diff: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
 Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Init Binary: docker-init
containerd version: bb71b10fd8f58240ca47fbb579b9d1028eea7c84
runc version: 2b18fe1d885ee5083ef9f0838fee39b62d653e30
init version: fec3683
Security Options:
 seccomp
  Profile: default
Kernel Version: 4.14.78-300.el7.bclinux.x86_64
Operating System: BigCloud Enterprise Linux For LDK 7 (Core)
OSType: linux
Architecture: x86_64
CPUs: 8
Total Memory: 30.91GiB
Name: WXJD-PSC-T-VM-CIS-2
ID: APXY:N3RV:QXW7:5MIV:PJGH:VDHQ:CRRG:4FTK:CGGC:26PY:UUP4:2BHN
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
Username: turningfish
Registry: https://index.docker.io/v1/ ####要是加速器地址拉不到镜像，就直接去dockerhub官方镜像
Labels:
Experimental: false
Insecure Registries:
 127.0.0.0/8
Registry Mirrors:
 http://10.154.12.120:7990/   ####这里是加速器地址，要是加速器地址拉不到镜像它会自动去上面的registry拉镜像
Live Restore Enabled: false
Product License: Community Engine
```

### 开始测试

上面顺利部署完之后，开始测试：

docker pull fishingflt/private2:v1 //这是我的私有镜像，你需要拉你的，拉我的你是拉不下来的

配置可以拉取私有镜像的registry参考文章：[https://docs.docker.com/registry/recipes/mirror/](https://docs.docker.com/registry/recipes/mirror/)。

如果remoteurl配置的是阿里云或者azure的加速地址，那么配置上去的username,password会被送去阿里云和微软云认证。因为账号和密码是Dockerhub的，所以认证失败，你也拉不到你想要的镜像。

![image-20200120175619079](file://C:/Users/fishing/AppData/Roaming/Typora/typora-user-images/image-20200120175619079.png?lastModify=1580206302)

将remoteurl改为[https://registry-1.docker.io](https://registry-1.docker.io)后，即可正常拉取对象

![image-20200120180046226](file://C:/Users/fishing/AppData/Roaming/Typora/typora-user-images/image-20200120180046226.png?lastModify=1580206302)

看见没，我们其实在拉取私有镜像fishingfly/private2:v1时没有要求我们去登录dockerhub。因为这一步是由registry帮你去做的，registry配置了dockerhub的用户名和密码，每次拉取私有镜像时，registry会用配置好的用户名和密码进行登录然后拉取私有镜像。所以只要知道registry镜像加速器地址的机器都能拉取fishingfly这个用户的私有镜像，这样是不安全的。

顺便测试下能不能正常拉取Dockerhub公共镜像：

![image-20200120183133198](file://C:/Users/fishing/AppData/Roaming/Typora/typora-user-images/image-20200120183133198.png?lastModify=1580206302)

### 总结

1. 配置缓存拉取私有镜像，参见官方文档：[https://docs.docker.com/registry/recipes/mirror/](https://docs.docker.com/registry/recipes/mirror/)
2. remoteUrl当拉取私有镜像时只能填中心仓库地址，不能填阿里云或者微软加速器的地址。（测试所得结论）
3. 使用dockerhub的url,配上username,pwd。（用户A的密码）。用户A能从加速器拉dockerhub私有镜像。用户B不能从加速器拉自己的私有镜像。同时用户A的私有镜像会被缓存在加速器本地，而且用户A的私有镜像能被所有能连到这个加速器的docker client所获取，所以我猜测是regsitry直接读取配置文件中的密码去dockerhub验证，验证完通过后才能拉取对象，这就不管你docker login哪个用户了，只要能连接加速器地址都能拉取用户A的私有镜像：
4. ![1578564186613](file://C:/Users/fishing/AppData/Roaming/Typora/typora-user-images/1578564186613.png?lastModify=1580206302)
5. 圈出来就是fishingfly用户的私有镜像，被存在镜像加速器里，下次Pull可以很快被其他机器拉下来

