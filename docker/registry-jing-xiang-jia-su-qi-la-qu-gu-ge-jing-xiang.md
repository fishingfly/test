# registry镜像加速器拉取谷歌镜像

中科大镜像地址：[http://mirrors.ustc.edu.cn/](http://mirrors.ustc.edu.cn/)

中科大github地址：

[https://github.com/ustclug/mirrorrequest](https://github.com/ustclug/mirrorrequest)

Azure中国镜像地址：[http://mirror.azure.cn/](http://mirror.azure.cn/)

Azure中国github地址：

[https://github.com/Azure/container-service-for-azure-china](https://github.com/Azure/container-service-for-azure-china)

## GCR Proxy Cache 帮助（这边走微软的可以直接拉取，那怎么做个自己的呢）

GCR Proxy Cache服务器相当于一台GCR镜像服务器，国内用户可以经由该服务器从[gcr.io](http://gcr.io/)下载镜像。

## 使用GCR Proxy Cache从gcr.io下载镜像

```text
docker pull gcr.azk8s.cn/google_containers/<imagename>:<version>
```

### 例子

```text
docker pull gcr.azk8s.cn/google_containers/pause-amd64:3.0
docker pull gcr.azk8s.cn/google_containers/kubedns-amd64:1.7
```

常用镜像仓库 DockerHub镜像仓库: [https://hub.docker.com/](https://hub.docker.com/) 阿里云镜像仓库： [https://cr.console.aliyun.com](https://cr.console.aliyun.com) google镜像仓库： [https://console.cloud.google.com/gcr/images/google-containers/GLOBAL](https://console.cloud.google.com/gcr/images/google-containers/GLOBAL) 如果你本地可以翻墙的话是可以连上去的 coreos镜像仓库： [https://quay.io/repository/](https://quay.io/repository/) RedHat镜像仓库： [https://access.redhat.com/containers](https://access.redhat.com/containers)

### 国内镜像源

部分国外镜像仓库无法访问，但国内有对应镜像源，可以从以下镜像源拉取到本地然后重改tag即可： Azure Container Registry\(ACR\)

```text
#dockerhub(docker.io)
#格式 
dockerhub.azk8s.cn/<repo-name>/<image-name>:<version>
#原镜像地址示例，我们可能平时拉dockerhub镜像是直接docker pull nginx:1.15.但是docker client会帮你翻译成#docker pull docker.io/library/nginx:1.15
docker.io/library/nginx:1.15
#国内拉取示例
dockerhub.azk8s.cn/library/nginx:1.15
​
#gcr.io 
#格式
gcr.azk8s.cn/<repo-name>/<image-name>:<version> 
#原镜像地址示例
gcr.io/google-containers/pause:3.1
#国内拉取示例
gcr.azk8s.cn/google_containers/pause:3.1
​
#quay.io
#格式
quay.azk8s.cn/<repo-name>/<image-name>:<version>
#原镜像地址示例
quay.io/coreos/etcd:v3.2.28
#国内拉取示例
quay.azk8s.cn/coreos/etcd:v3.2.28
​
#k8s.gcr.io
#格式
gcr.azk8s.cn/google_containers/<repo-name>/<image-name>:<version>
#原镜像地址示例
k8s.gcr.io/pause-amd64:3.1
#国内拉取示例
gcr.azk8s.cn/google_containers/pause:3.1
```

dockerhub

```text
#原镜像格式
k8s.gcr.io/pause:3.1
#改为以下格式
googlecontainersmirrors/pause:3.1
```

阿里云google镜像源

```text
#原镜像格式
k8s.gcr.io/pause:3.1
#改为以下格式
registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
```

或使用azk8spull，只有50行命令的小脚本，就可以从dockerhub、gcr.io、quay.io直接拉取镜像：

```text
#download azk8spull
curl -Lo /usr/local/bin/azk8spull https://github.com/xuxinkun/littleTools/releases/download/v1.0.0/azk8spull
chmod +x /usr/local/bin/azk8spull
​
#直接拉取镜像
azk8spull k8s.gcr.io/pause:3.1
azk8spull quay.io/coreos/etcd:v3.2.28
​
#查看拉取的镜像
# docker images
REPOSITORY                                                        TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/etcd                                                   v3.2.28             b2756210eeab        3 months ago        247MB
k8s.gcr.io/pause                                                  3.1
```

测试结果：

我配了一个Dockerhub的regsitry和谷歌镜像的registry。然后再客户机上配置registry mirrors参数，重启docker,centos下运行

systemctl daemon-reload

systemctl restart docker

然后你运行docker system info看下：

```text
Containers: 4
 Running: 2
 Paused: 0
 Stopped: 2
Images: 75
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
Name: WXJD-PSC-T-VM-CIS-8
ID: APXY:N3RV:QXW7:5MIV:PJGH:VDHQ:CRRG:4FTK:CGGC:26PY:UUP4:2BHN
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
Username: fishingfly
Registry: https://index.docker.io/v1/   ##要是mirror地址配的加速器地址全失效，那就走这拉吧
Labels:
Experimental: false
Insecure Registries:
 127.0.0.0/8
Registry Mirrors:
 http://10.154.12.120:7997/       ##mirror地址1
 http://10.154.12.120:7998/       ##mirror地址2
Live Restore Enabled: false
Product License: Community Engine
```

docker先向mirror地址1加速器请求，要是这个加速器可以拉到这个镜像，那就在这拉，要是这个加速器拉不到这个镜像，那就向mirror地址2拉镜像，要是第二个镜像加速器也拉不到，那就往 Registry: [https://index.docker.io/v1/](https://index.docker.io/v1/) 拉.

