---
title: "kubelet启动问题"
isCJKLanguage: true
date: 2023-05-13 17:53:12
updated: 2023-06-04 22:01:45
categories: 
- IT
- kubernetes
tags: 
- kubernetes
---

# Kubelet启动问题

环境:



![image-20230317211717739](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/Typora20230317211717.png)



Kubelet启动失败：

![image-20230316205038267](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/Typora20230316205038.png)

## 问题一: failed to validate kubelet flags: the container runtime endpoint address was not specified or empty, use --container-runtime-endpoint to set

查看系统日志发现：

![image-20230316205009102](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/Typora20230316205016.png)

于是使用--container-runtime-endpoint指定containerd作为CRI，命令行启动kubelet：

![image-20230316205640497](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/Typora20230316205640.png)

出现报错，查看一下containerd配置文件:

![image-20230316210245953](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/Typora20230316210246.png)

{%spoiler 示例代码%}
```yaml
disabled_plugins = ["cri"]
```
{%endspoiler%}

将这一句注释掉，重启containerd，并运行kubelet:

{%spoiler 示例代码%}
```shell
systemctl restart containerd

/usr/bin/kubelet --container-runtime-endpoint=unix:///run//containerd/containerd.sock
```
{%endspoiler%}

测试通过后，修改service文件，添加--container-runtime-endpoint参数，kubelet需要修改/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf:

{%spoiler 示例代码%}
```yaml
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --container-runtime-endpoint=unix:///run//containerd/containerd.sock

CPUAccounting=true
MemoryAccounting=true
```
{%endspoiler%}



## 问题二：failed to load kubelet config file

需要执行下面命令生成相应的配置文件：

{%spoiler 示例代码%}
```shell
kubeadm init
```
{%endspoiler%}

但是由于国内网络问题，可能会报错：

{%spoiler 示例代码%}
```shell
[root@centos02-01-ssd ~]# kubeadm init
[init] Using Kubernetes version: v1.26.2
[preflight] Running pre-flight checks
	[WARNING Hostname]: hostname "centos02-01-ssd" could not be reached
	[WARNING Hostname]: hostname "centos02-01-ssd": lookup centos02-01-ssd on 192.168.93.2:53: no such host
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR ImagePull]: failed to pull image registry.k8s.io/kube-apiserver:v1.26.2: output: E0316 07:35:48.281423  242221 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-apiserver:v1.26.2\": failed to resolve reference \"registry.k8s.io/kube-apiserver:v1.26.2\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-apiserver/manifests/v1.26.2\": dial tcp 64.233.188.82:443: connect: connection refused" image="registry.k8s.io/kube-apiserver:v1.26.2"
time="2023-03-16T07:35:48+08:00" level=fatal msg="pulling image: rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-apiserver:v1.26.2\": failed to resolve reference \"registry.k8s.io/kube-apiserver:v1.26.2\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-apiserver/manifests/v1.26.2\": dial tcp 64.233.188.82:443: connect: connection refused"
, error: exit status 1
	[ERROR ImagePull]: failed to pull image registry.k8s.io/kube-controller-manager:v1.26.2: output: E0316 07:37:35.233280  242802 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-controller-manager:v1.26.2\": failed to resolve reference \"registry.k8s.io/kube-controller-manager:v1.26.2\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-controller-manager/manifests/v1.26.2\": dial tcp 142.251.8.82:443: connect: connection refused" image="registry.k8s.io/kube-controller-manager:v1.26.2"
time="2023-03-16T07:37:35+08:00" level=fatal msg="pulling image: rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-controller-manager:v1.26.2\": failed to resolve reference \"registry.k8s.io/kube-controller-manager:v1.26.2\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-controller-manager/manifests/v1.26.2\": dial tcp 142.251.8.82:443: connect: connection refused"
, error: exit status 1
	[ERROR ImagePull]: failed to pull image registry.k8s.io/kube-scheduler:v1.26.2: output: E0316 07:39:22.339290  243388 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-scheduler:v1.26.2\": failed to resolve reference \"registry.k8s.io/kube-scheduler:v1.26.2\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-scheduler/manifests/v1.26.2\": dial tcp 142.250.157.82:443: connect: connection refused" image="registry.k8s.io/kube-scheduler:v1.26.2"
time="2023-03-16T07:39:22+08:00" level=fatal msg="pulling image: rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-scheduler:v1.26.2\": failed to resolve reference \"registry.k8s.io/kube-scheduler:v1.26.2\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-scheduler/manifests/v1.26.2\": dial tcp 142.250.157.82:443: connect: connection refused"
, error: exit status 1
	[ERROR ImagePull]: failed to pull image registry.k8s.io/kube-proxy:v1.26.2: output: E0316 07:41:11.695464  244012 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-proxy:v1.26.2\": failed to resolve reference \"registry.k8s.io/kube-proxy:v1.26.2\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-proxy/manifests/v1.26.2\": dial tcp 74.125.204.82:443: connect: connection refused" image="registry.k8s.io/kube-proxy:v1.26.2"
time="2023-03-16T07:41:11+08:00" level=fatal msg="pulling image: rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-proxy:v1.26.2\": failed to resolve reference \"registry.k8s.io/kube-proxy:v1.26.2\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-proxy/manifests/v1.26.2\": dial tcp 74.125.204.82:443: connect: connection refused"
, error: exit status 1
...
```
{%endspoiler%}

此时可以指定使用ali源：

{%spoiler 示例代码%}
```shell
kubeadm init --image-repository=registry.aliyuncs.com/google_containers
```
{%endspoiler%}



## crictl报错

执行crictl报错runtime.v1.ImageService，如下：

{%spoiler 示例代码%}
```shell
[root@centos01 ~]# crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock images
FATA[0000] validate service connection: CRI v1 image API is not implemented for endpoint "unix:///var/run/containerd/containerd.sock": rpc error: code = Unimplemented desc = unknown service runtime.v1.ImageService 
```
{%endspoiler%}

尝试各种办法，如更换docker版本等，均无效，最后更改containerd配置文件（路径：/etc/containerd/config.toml）：

{%spoiler 示例代码%}
```ini

#   Copyright 2018-2022 Docker Inc.

#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at

#       http://www.apache.org/licenses/LICENSE-2.0

#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

# 注释掉这一行
#disabled_plugins = ["cri"]

#root = "/var/lib/containerd"
#state = "/run/containerd"
#subreaper = true
#oom_score = 0

#[grpc]
#  address = "/run/containerd/containerd.sock"
#  uid = 0
#  gid = 0

#[debug]
#  address = "/run/containerd/debug.sock"
#  uid = 0
#  gid = 0
#  level = "info"
```
{%endspoiler%}

如上，注释掉disabled_plugins = ["cri"]后，重启congtainerd服务：

{%spoiler 示例代码%}
```shell
systemctl restart containerd
```
{%endspoiler%}

crictl现在可以正常使用。

## minikube启动失败

尝试如下命令(仅限于cri使用containerd)：

{%spoiler 示例代码%}
```shell
minikube delete

minikube start --container-runtime=containerd
```
{%endspoiler%}



## 参考

[【K8S 八】使用containerd作为CRI](https://blog.csdn.net/avatar_2009/article/details/126020671)

[浅析Docker、Containerd、RunC分别是什么](https://www.51cto.com/article/687502.html)

[k8s安装遇到错误： failed to load Kubelet config file /var/lib/kubelet/config.yaml, error failed to read kub](https://blog.csdn.net/yuxuan89814/article/details/118220640)

[kubeadm init [ERROR ImagePull]: failed to pull image registry.k8s.io 解决方法](https://blog.51cto.com/wanghaoyu/6022105)

[快速解决Kubernetes从k8s.gcr.io仓库拉取镜像失败问题](https://www.toutiao.com/article/7010635949466042919/)

[kubeadm init初始化时报错failed to pull image "k8s.gcr.io/kube-apiserver:v1.20.4": output: Error response from daemon: Get ht](https://www.cnblogs.com/qiaoer1993/p/14504615.html)

[i can't start minikube](https://stackoverflow.com/questions/66173362/i-cant-start-minikube)

[unknown service runtime.v1alpha2.ImageService](https://github.com/kubernetes-sigs/cri-tools/issues/710)