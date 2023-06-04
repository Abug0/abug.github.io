---
title: "k8s集群安装"
isCJKLanguage: true
date: 2023-06-04 22:08:11
updated: 2023-06-04 22:05:21
categories: 
- IT
- kubernetes
tags: 
- kubernetes
---

# VMWare虚拟机安装K8s集群

## 一、安装kubeadm、kubectl、kubelet

{%spoiler 示例代码%}
```shell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```
{%endspoiler%}
## 二、容器运行时安装和配置

此处使用的是containerd

{%spoiler 示例代码%}
```shell
# 使用ali源
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

sed -i 's/download.docker.com/mirrors.aliyun.com\/docker-ce/g' /etc/yum.repos.d/docker-ce.repo

# 官方源
# $ sudo yum-config-manager \
#     --add-repo \
#     https://download.docker.com/linux/centos/docker-ce.repo

yum install containerd.io

# 启动containerd
systemctl start containerd
```
{%endspoiler%}
## 三、创建集群

{%spoiler 示例代码%}
```shell
# 关闭swap
swapoff -a

# 关闭防火墙(不推荐，推荐只放行必要端口)
systemctl stop firewalld

# 创建集群
kubeadm init --pod-network-cidr 10.244.0.0/16
```
{%endspoiler%}
报错信息如下：

{%spoiler 示例代码%}
```shell
[root@centos02-01-ssd k8s]# kubeadm init
[init] Using Kubernetes version: v1.27.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W0513 21:13:11.833251    6481 images.go:80] could not find officially supported version of etcd for Kubernetes v1.27.1, falling back to the nearest etcd version (3.5.7-0)


W0513 21:18:32.812024    6481 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR ImagePull]: failed to pull image registry.k8s.io/kube-controller-manager:v1.27.1: output: E0513 21:14:58.719147    6545 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-controller-manager:v1.27.1\": failed to resolve reference \"registry.k8s.io/kube-controller-manager:v1.27.1\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-controller-manager/manifests/v1.27.1\": dial tcp 64.233.187.82:443: connect: connection refused" image="registry.k8s.io/kube-controller-manager:v1.27.1"
time="2023-05-13T21:14:58+08:00" level=fatal msg="pulling image: rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-controller-manager:v1.27.1\": failed to resolve reference \"registry.k8s.io/kube-controller-manager:v1.27.1\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-controller-manager/manifests/v1.27.1\": dial tcp 64.233.187.82:443: connect: connection refused"
, error: exit status 1
	[ERROR ImagePull]: failed to pull image registry.k8s.io/kube-scheduler:v1.27.1: output: E0513 21:16:46.060474    6581 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-scheduler:v1.27.1\": failed to resolve reference \"registry.k8s.io/kube-scheduler:v1.27.1\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-scheduler/manifests/v1.27.1\": dial tcp 64.233.187.82:443: connect: connection refused" image="registry.k8s.io/kube-scheduler:v1.27.1"
time="2023-05-13T21:16:46+08:00" level=fatal msg="pulling image: rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-scheduler:v1.27.1\": failed to resolve reference \"registry.k8s.io/kube-scheduler:v1.27.1\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-scheduler/manifests/v1.27.1\": dial tcp 64.233.187.82:443: connect: connection refused"
, error: exit status 1
	[ERROR ImagePull]: failed to pull image registry.k8s.io/kube-proxy:v1.27.1: output: E0513 21:18:32.760371    6618 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-proxy:v1.27.1\": failed to resolve reference \"registry.k8s.io/kube-proxy:v1.27.1\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-proxy/manifests/v1.27.1\": dial tcp 64.233.187.82:443: connect: connection refused" image="registry.k8s.io/kube-proxy:v1.27.1"
time="2023-05-13T21:18:32+08:00" level=fatal msg="pulling image: rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/kube-proxy:v1.27.1\": failed to resolve reference \"registry.k8s.io/kube-proxy:v1.27.1\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/kube-proxy/manifests/v1.27.1\": dial tcp 64.233.187.82:443: connect: connection refused"
, error: exit status 1
	[ERROR ImagePull]: failed to pull image registry.k8s.io/pause:3.9: output: E0513 21:20:19.493222    6660 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/pause:3.9\": failed to resolve reference \"registry.k8s.io/pause:3.9\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/pause/manifests/3.9\": dial tcp 64.233.187.82:443: connect: connection refused" image="registry.k8s.io/pause:3.9"
time="2023-05-13T21:20:19+08:00" level=fatal msg="pulling image: rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/pause:3.9\": failed to resolve reference \"registry.k8s.io/pause:3.9\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/pause/manifests/3.9\": dial tcp 64.233.187.82:443: connect: connection refused"
, error: exit status 1
	[ERROR ImagePull]: failed to pull image registry.k8s.io/etcd:3.5.7-0: output: E0513 21:22:06.129274    6704 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/etcd:3.5.7-0\": failed to resolve reference \"registry.k8s.io/etcd:3.5.7-0\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/etcd/manifests/3.5.7-0\": dial tcp 64.233.187.82:443: connect: connection refused" image="registry.k8s.io/etcd:3.5.7-0"
time="2023-05-13T21:22:06+08:00" level=fatal msg="pulling image: rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/etcd:3.5.7-0\": failed to resolve reference \"registry.k8s.io/etcd:3.5.7-0\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/etcd/manifests/3.5.7-0\": dial tcp 64.233.187.82:443: connect: connection refused"
, error: exit status 1
	[ERROR ImagePull]: failed to pull image registry.k8s.io/coredns/coredns:v1.10.1: output: E0513 21:23:52.854222    6741 remote_image.go:171] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/coredns/coredns:v1.10.1\": failed to resolve reference \"registry.k8s.io/coredns/coredns:v1.10.1\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/coredns/coredns/manifests/v1.10.1\": dial tcp 64.233.188.82:443: connect: connection refused" image="registry.k8s.io/coredns/coredns:v1.10.1"
time="2023-05-13T21:23:52+08:00" level=fatal msg="pulling image: rpc error: code = Unknown desc = failed to pull and unpack image \"registry.k8s.io/coredns/coredns:v1.10.1\": failed to resolve reference \"registry.k8s.io/coredns/coredns:v1.10.1\": failed to do request: Head \"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/coredns/coredns/manifests/v1.10.1\": dial tcp 64.233.188.82:443: connect: connection refused"
, error: exit status 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```
{%endspoiler%}
由于国内网络原因，提示image拉取失败，此时有两种解决方法：

* 使用代理

* 切换到国内的源

### 1、使用代理

**在环境变量中设置代理是无效的**

需要为containerd服务设置代理才能生效，参考：

> 我们发现下载镜像报错，这是因为国内没办法访问 `k8s.gcr.io`，而且无论是在环境变量中设置代理，还是为 Docker Daemon 设置代理，都不起作用。后来才意识到，`kubeadm config images pull` 命令貌似不走 docker 服务，而是直接请求 containerd 服务，所以我们为 containerd 服务设置代理。

设置containerd代理：

{%spoiler 示例代码%}
```shell
vi /usr/lib/systemd/system/containerd.service.d/http_proxy.conf
```
{%endspoiler%}
文件内容如下：

{%spoiler 示例代码%}
```ini
[Service]
Environment="HTTP_PROXY=socks5://192.168.93.1:51837"
Environment="HTTPS_PROXY=socks5://192.168.93.1:51837"
```
{%endspoiler%}
重启服务：

{%spoiler 示例代码%}
```shell
systemctl daemon-reload
systemctl restart containerd
```
{%endspoiler%}
再次执行kubeadm init：

{%spoiler 示例代码%}
```shell
[root@centos02 ~]# kubeadm init

[init] Using Kubernetes version: v1.27.1
[preflight] Running pre-flight checks
	[WARNING Hostname]: hostname "centos02" could not be reached
	[WARNING Hostname]: hostname "centos02": lookup centos02 on 192.168.93.2:53: no such host
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W0513 22:42:39.814401   12142 images.go:80] could not find officially supported version of etcd for Kubernetes v1.27.1, falling back to the nearest etcd version (3.5.7-0)

W0513 22:42:40.005436   12142 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [centos02 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.93.136]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [centos02 localhost] and IPs [192.168.93.136 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [centos02 localhost] and IPs [192.168.93.136 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
W0513 22:42:42.904764   12142 images.go:80] could not find officially supported version of etcd for Kubernetes v1.27.1, falling back to the nearest etcd version (3.5.7-0)
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[apiclient] All control plane components are healthy after 61.504880 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node centos02 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node centos02 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: f3n3t3.smh43r6dltn85ugz
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.93.136:6443 --token f3n3t3.smh43r6dltn85ugz \
	--discovery-token-ca-cert-hash sha256:673622b61260cf5f5a3482a51da8192aef264c8ab4305684b4d317601b3b420e
```
{%endspoiler%}
### 2、使用国内的源

**需要重载pause镜像, 但是第二次创建集群时重载没有成功**

这里使用的是ali的源:

{%spoiler 示例代码%}
```shell
kubeadm init --image-repository=registry.aliyuncs.com/google_containers --kubernetes-version v1.27.1
```
{%endspoiler%}
但是此时依然存在问题，输出如下：

{%spoiler 示例代码%}
```shell
[root@centos02 k8s]# kubeadm init --image-repository=registry.aliyuncs.com/google_containers --kubernetes-version v1.27.1

[init] Using Kubernetes version: v1.27.1
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]: /etc/kubernetes/manifests/kube-apiserver.yaml already exists
	[ERROR FileAvailable--etc-kubernetes-manifests-kube-controller-manager.yaml]: /etc/kubernetes/manifests/kube-controller-manager.yaml already exists
	[ERROR FileAvailable--etc-kubernetes-manifests-kube-scheduler.yaml]: /etc/kubernetes/manifests/kube-scheduler.yaml already exists
	[ERROR FileAvailable--etc-kubernetes-manifests-etcd.yaml]: /etc/kubernetes/manifests/etcd.yaml already exists
	[ERROR Port-10250]: Port 10250 is in use
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
[root@centos02-01-ssd k8s]# kubeadm reset --kubeconfig kubeadm.yaml
W0513 22:07:21.208438    8046 preflight.go:56] [reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
[reset] Are you sure you want to proceed? [y/N]: y
[preflight] Running pre-flight checks
W0513 22:07:22.278340    8046 removeetcdmember.go:106] [reset] No kubeadm config, using etcd pod spec to get data directory
[reset] Stopping the kubelet service
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Deleting contents of directories: [/etc/kubernetes/manifests /var/lib/kubelet /etc/kubernetes/pki]
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]

The reset process does not clean CNI configuration. To do so, you must remove /etc/cni/net.d

The reset process does not reset or clean up iptables rules or IPVS tables.
If you wish to reset iptables, you must do so manually by using the "iptables" command.

If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
to reset your system's IPVS tables.

The reset process does not clean your kubeconfig files and you must remove them manually.
Please, check the contents of the $HOME/.kube/config file.
[root@centos02-01-ssd k8s]# kubeadm init --image-repository=registry.aliyuncs.com/google_containers --kubernetes-version v1.27.1
[init] Using Kubernetes version: v1.27.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W0513 22:07:24.088826    8061 images.go:80] could not find officially supported version of etcd for Kubernetes v1.27.1, falling back to the nearest etcd version (3.5.7-0)
W0513 22:07:24.273125    8061 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.aliyuncs.com/google_containers/pause:3.9" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [centos02-01-ssd kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.93.129]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [centos02-01-ssd localhost] and IPs [192.168.93.129 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [centos02-01-ssd localhost] and IPs [192.168.93.129 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
W0513 22:07:26.736169    8061 images.go:80] could not find officially supported version of etcd for Kubernetes v1.27.1, falling back to the nearest etcd version (3.5.7-0)
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s

[kubelet-check] Initial timeout of 40s passed.
Unfortunately, an error has occurred:
	timed out waiting for the condition
This error is likely caused by:
	- The kubelet is not running
	- The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)
If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
	- 'systemctl status kubelet'
	- 'journalctl -xeu kubelet'

Additionally, a control plane component may have crashed or exited when started by the container runtime.
To troubleshoot, list all containers using your preferred container runtimes CLI.
Here is one example how you may list all running Kubernetes containers by using crictl:
	- 'crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps -a | grep kube | grep -v pause'
	Once you have found the failing container, you can inspect its logs with:
	- 'crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock logs CONTAINERID'
error execution phase wait-control-plane: couldn't initialize a Kubernetes cluster
To see the stack trace of this error execute with --v=5 or higher
```
{%endspoiler%}
查看系统日志/var/log/message，发现有如下报错：

{%spoiler 示例代码%}
```ini
May 13 03:52:00 centos02 containerd: time="2023-05-13T03:52:00.916348843+08:00" level=info msg="trying next host" error="failed to do request: Head \"https://us-west2-docker.pkg.dev/v2/k8s-artifacts-prod/images/pause/manifests/3.9\": dial tcp 142.251.8.82:443: connect: connection refused" host=registry.k8s.io
May 13 03:52:00 centos02 containerd: time="2023-05-13T03:52:00.921119344+08:00" level=error msg="PullImage \"registry.k8s.io/pause:3.9\" failed" error="failed to pull and unpack image \"registry.k8s.io/pause:3.9\": failed to resolve reference \"registry.k8s.io/pause:3.9\": failed to do request: Head \"https://us-west2-docker.pkg.dev/v2/k8s-artifacts-prod/images/pause/manifests/3.9\": dial tcp 142.251.8.82:443: connect: connection refused"
May 13 03:52:00 centos02 containerd: time="2023-05-13T03:52:00.958651647+08:00" level=info msg="PullImage \"registry.k8s.io/pause:3.9\""
```
{%endspoiler%}
还是需要registry.k8s.io/pause:3.9的镜像，此时可以在containerd服务中重载pause镜像：

{%spoiler 示例代码%}
```shell
vi /etc/containerd/config.toml
```
{%endspoiler%}
{%spoiler 示例代码%}
```ini
#disabled_plugins = ["cri"]

# 重载pause镜像
[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.aliyuncs.com/google_containers/pause/pause:3.9"

#root = "/var/lib/containerd"
#state = "/run/containerd"
#subreaper = true
#oom_score = 0

[grpc]
address = "/run/containerd/containerd.sock"
uid = 0
gid = 0

#[debug]
#  address = "/run/containerd/debug.sock"
#  uid = 0
#  gid = 0
#  level = "info"
```
{%endspoiler%}
修改后重启containerd服务：

{%spoiler 示例代码%}
```shell
systemctl restart containerd
```
{%endspoiler%}
重新执行kubeadm init，执行成功。

## 四、继续配置集群

按照kubeadm init输出显示：

{%spoiler 示例代码%}
```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.10:6443 --token 34umd7.vitcuquy5egbslde \
	--discovery-token-ca-cert-hash sha256:e59f67eea49c49750a8d4ef0c1e0a56518fc7e57522f3fe4ff898461dafaab43
```
{%endspoiler%}
按照提示执行(一开始只执行了export KUBECONFIG=/etc/kubernetes/admin.conf，结果后面报错了)：

{%spoiler 示例代码%}
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
{%endspoiler%}
然后安装cni插件，这里使用的是flannel：

{%spoiler 示例代码%}
```shell
[root@centos02 k8s]# curl -LO https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
[root@centos02 k8s]# kubectl apply -f kube-flannel.yml
namespace/kube-flannel created
serviceaccount/flannel created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```
{%endspoiler%}
查看kube-flannel.yml，其中有一段：

{%spoiler 示例代码%}
```ini
apiVersion: v1
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
kind: ConfigMap

```
{%endspoiler%}
注意这里的network要与kubeadm init的--pod-network-cidr保持一致。

最后，在其他机器执行init输出的命令，加入集群：

{%spoiler 示例代码%}
```shell
kubeadm join 192.168.1.10:6443 --token 34umd7.vitcuquy5egbslde --discovery-token-ca-cert-hash sha256:e59f67eea49c49750a8d4ef0c1e0a56518fc7e57522f3fe4ff898461dafaab43
```
{%endspoiler%}


## 五、配置dashboard

执行如下命令部署dashboard:

{%spoiler 示例代码%}
```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```
{%endspoiler%}
执行：

{%spoiler 示例代码%}
```shell
kubectl proxy
```
{%endspoiler%}
现在可以通过http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/ 访问。

**note: 此时在本机外部还是无法访问，原因是可选的访问方式只有三种：**

> If your login view displays below error, this means that you are trying to log in over HTTP and it has been disabled for the security reasons.
>
> Logging in is available only if URL used to access Dashboard starts with:
>
> - `http://localhost/...`
> - `http://127.0.0.1/...`
> - `https://<domain_name>/...`

![Login disabled](https://github.com/kubernetes/dashboard/raw/master/docs/user/images/dashboard-login-disabled.png)

此时可以通过NodePort方式来开启访问（仅适用于开发环境）：

{%spoiler 示例代码%}
```shell
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
```
{%endspoiler%}
将type: ClusterIP修改为NodePort即可。

登录Token的获取方式参考[Creating sample user](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)，先创建用户，然后获取Token。



## 六、遇到的问题

### 1、kubectl提示connection refused

{%spoiler 示例代码%}
```shell
[root@centos02-01-ssd ~]# kubectl get pods -A
E0514 01:29:41.294053   22921 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0514 01:29:41.297024   22921 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0514 01:29:41.298929   22921 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0514 01:29:41.303931   22921 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0514 01:29:41.305052   22921 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```
{%endspoiler%}
这个时候通过crictl查看容器和pod，状态是正常的：

{%spoiler 示例代码%}
```shell
[root@centos02 ~]# crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD
da656f9d75bc2       c6b5118178229       14 minutes ago      Running             kube-controller-manager   2                   88762c1d1922e       kube-controller-manager-centos02-01-ssd
25e0f0e1725b2       6f6e73fa8162b       14 minutes ago      Running             kube-apiserver            3                   7b4e181301b82       kube-apiserver-centos02-01-ssd
6b817f57147d9       ead0a4a53df89       15 minutes ago      Running             coredns                   2                   84d90e5449d90       coredns-5d78c9869d-p6p7z
5a271a66e9bc1       ead0a4a53df89       15 minutes ago      Running             coredns                   2                   a630dd3d8e46e       coredns-5d78c9869d-nkg6k
191c9f52e7937       86b6af7dd652c       15 minutes ago      Running             etcd                      3                   4dcd2a8ae6bb8       etcd-centos02-01-ssd
14ee5f211ec8b       6468fa8f98696       15 minutes ago      Running             kube-scheduler            5                   db3f6fe4a762c       kube-scheduler-centos02-01-ssd
fe67ff6d7c456       fbe39e5d66b6a       2 hours ago         Running             kube-proxy                0                   4d2f71e24c9d8       kube-proxy-n2d2m


[root@centos02 ~]# crictl --runtime-endpoint unix:///run/containerd/containerd.sock pods
POD ID              CREATED             STATE               NAME                                      NAMESPACE           ATTEMPT             RUNTIME
84d90e5449d90       2 hours ago         Ready               coredns-5d78c9869d-p6p7z                  kube-system         0                   (default)
4d2f71e24c9d8       2 hours ago         Ready               kube-proxy-n2d2m                          kube-system         0                   (default)
a630dd3d8e46e       2 hours ago         Ready               coredns-5d78c9869d-nkg6k                  kube-system         0                   (default)
88762c1d1922e       2 hours ago         Ready               kube-controller-manager-centos02-01-ssd   kube-system         0                   (default)
7b4e181301b82       2 hours ago         Ready               kube-apiserver-centos02-01-ssd            kube-system         0                   (default)
4dcd2a8ae6bb8       2 hours ago         Ready               etcd-centos02-01-ssd                      kube-system         0                   (default)
db3f6fe4a762c       2 hours ago         Ready               kube-scheduler-centos02-01-ssd            kube-system         0                   (default)
```
{%endspoiler%}
最后查出是证书和配置问题，执行如下命令解决：

{%spoiler 示例代码%}
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
{%endspoiler%}
之所以遇到这个问题，是因为我在前面步骤中只执行了：

{%spoiler 示例代码%}
```shell
export KUBECONFIG=/etc/kubernetes/admin.conf
```
{%endspoiler%}
### 2、flannel处于CrashLoopBackOff:  open /run/flannel/subnet.env: no such file or directory

新建 /run/flannel/subnet.env文件，写入：

{%spoiler 示例代码%}
```ini
# 此处网络信息需与kubeadm init时指定的pod-network-cidr一致
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```
{%endspoiler%}
### 3、flannel处于CrashLoopBackOff: Error registering network

查看pods发现，flannel服务异常：

{%spoiler 示例代码%}
```shell
[root@centos02 ~]# kubectl get pods -o wide -A

NAMESPACE      NAME                           READY   STATUS             RESTARTS         AGE     IP               NODE   NOMINATED NODE   READINESS GATES
kube-flannel   kube-flannel-ds-9f7zf          0/1     CrashLoopBackOff   61 (4m14s ago)   6h13m   192.168.93.129   node   <none>           <none>
kube-system    coredns-5d78c9869d-4g4j7       1/1     Running            1 (4h3m ago)     6h16m   10.244.0.6       node   <none>           <none>
kube-system    coredns-5d78c9869d-r8c62       1/1     Running            1 (4h5m ago)     6h16m   10.244.0.7       node   <none>           <none>
kube-system    etcd-node                      1/1     Running            1 (3h37m ago)    6h16m   192.168.1.10     node   <none>           <none>
kube-system    kube-apiserver-node            1/1     Running            1 (3h52m ago)    6h16m   192.168.1.10     node   <none>           <none>
kube-system    kube-controller-manager-node   1/1     Running            7                6h16m   192.168.1.10     node   <none>           <none>
kube-system    kube-proxy-glmbw               1/1     Running            0                6h16m   192.168.1.10     node   <none>           <none>
kube-system    kube-scheduler-node            1/1     Running            8                6h16m   192.168.1.10     node   <none>           <none>

```
{%endspoiler%}
flannel服务一直处于CrashLoopBackOff状态，使用kubectl logs检查日志：

{%spoiler 示例代码%}
```shell
[root@centos02 ~]# kubectl logs kube-flannel-ds-9f7zf -n kube-flannel

Defaulted container "kube-flannel" out of: kube-flannel, install-cni-plugin (init), install-cni (init)
I0514 02:31:58.024055       1 main.go:211] CLI flags config: {etcdEndpoints:http://127.0.0.1:4001,http://127.0.0.1:2379 etcdPrefix:/coreos.com/network etcdKeyfile: etcdCertfile: etcdCAFile: etcdUsername: etcdPassword: version:false kubeSubnetMgr:true kubeApiUrl: kubeAnnotationPrefix:flannel.alpha.coreos.com kubeConfigFile: iface:[] ifaceRegex:[] ipMasq:true ifaceCanReach: subnetFile:/run/flannel/subnet.env publicIP: publicIPv6: subnetLeaseRenewMargin:60 healthzIP:0.0.0.0 healthzPort:0 iptablesResyncSeconds:5 iptablesForwardRules:true netConfPath:/etc/kube-flannel/net-conf.json setNodeNetworkUnavailable:true useMultiClusterCidr:false}
W0514 02:31:58.024235       1 client_config.go:617] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0514 02:31:58.230778       1 kube.go:485] Starting kube subnet manager
I0514 02:31:58.221851       1 kube.go:144] Waiting 10m0s for node controller to sync
I0514 02:31:59.233777       1 kube.go:151] Node controller sync successful
I0514 02:31:59.233820       1 main.go:231] Created subnet manager: Kubernetes Subnet Manager - node
I0514 02:31:59.233828       1 main.go:234] Installing signal handlers
I0514 02:31:59.235799       1 main.go:542] Found network config - Backend type: vxlan
I0514 02:31:59.236830       1 match.go:206] Determining IP address of default interface
I0514 02:31:59.289824       1 match.go:259] Using interface with name ens33 and address 192.168.93.129
I0514 02:31:59.290780       1 match.go:281] Defaulting external address to interface address (192.168.93.129)
I0514 02:31:59.290845       1 vxlan.go:140] VXLAN config: VNI=1 Port=0 GBP=false Learning=false DirectRouting=false
E0514 02:31:59.296851       1 main.go:334] Error registering network: failed to acquire lease: node "node" pod cidr not assigned
I0514 02:31:59.304807       1 main.go:522] Stopping shutdownHandler...
```
{%endspoiler%}
可以看到有一行错误信息：Error registering network: failed to acquire lease: node "node" pod cidr not assigned。

查阅资料后得知，还是pod-netword-cidr设置问题，重新创建集群，指定--pod-network-cidr即可（**需与flannel配置文件里的值保持一致**）。

### 4、节点加入集群失败：couldn't validate the identity of the API Server:

{%spoiler 示例代码%}
```shell
[root@centos02 ~]# kubeadm join 192.168.93.129:6443 --token 1db1oa.mkk4vqbqej1ahedi --discovery-token-ca-cert-hash sha256:26437963304d98baf542998c5de640886a81637825f90da4f5aa56aab76032bc

[preflight] Running pre-flight checks
	[WARNING Hostname]: hostname "centos02" could not be reached
	[WARNING Hostname]: hostname "centos02": lookup centos02 on 192.168.93.2:53: no such host
error execution phase preflight: couldn't validate the identity of the API Server: could not find a JWS signature in the cluster-info ConfigMap for token ID "1db1oa"
To see the stack trace of this error execute with --v=5 or higher
```
{%endspoiler%}
报错说明token过期，在master节点执行如下命令：

{%spoiler 示例代码%}
```shell
[root@centos0 k8s]# kubeadm token generate
1db1oa.mkk4vqbqej1ahedi
[root@centos02 k8s]# kubeadm token create 1db1oa.mkk4vqbqej1ahedi --print-join-command
kubeadm join 192.168.93.129:6443 --token 1db1oa.mkk4vqbqej1ahedi --discovery-token-ca-cert-hash sha256:26437963304d98baf542998c5de640886a81637825f90da4f5aa56aab76032bc
```
{%endspoiler%}
节点机器重新执行：

{%spoiler 示例代码%}
```shell
[root@centos02 ~]# kubeadm join 192.168.93.129:6443 --token 1db1oa.mkk4vqbqej1ahedi --discovery-token-ca-cert-hash sha256:26437963304d98baf542998c5de640886a81637825f90da4f5aa56aab76032bc 
[preflight] Running pre-flight checks
	[WARNING Hostname]: hostname "centos02" could not be reached
	[WARNING Hostname]: hostname "centos02": lookup centos02 on 192.168.93.2:53: no such host
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

```
{%endspoiler%}
### 5、在其他节点上使用kubectl命令

在工作节点上使用kubectl查看集群情况时报错：

{%spoiler 示例代码%}
```shell
[root@centos02 ~]# kubectl get nodes
E0516 08:14:53.832935   46677 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0516 08:14:53.835782   46677 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0516 08:14:53.836993   46677 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0516 08:14:53.838610   46677 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0516 08:14:53.839873   46677 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
```
{%endspoiler%}
可以看出，kubectl请求的默认地址是localhost:8080，但我们的kube-apiserver运行在控制节点上，所以地址肯定是不对的，于是通过-s指定地址：

{%spoiler 示例代码%}
```shell
[root@centos02 ~]# kubectl get nodes -s 192.168.93.129:6443
E0516 08:19:41.879611   47920 memcache.go:265] couldn't get current server API group list: the server rejected our request for an unknown reason
E0516 08:19:41.886542   47920 memcache.go:265] couldn't get current server API group list: the server rejected our request for an unknown reason
E0516 08:19:41.895951   47920 memcache.go:265] couldn't get current server API group list: the server rejected our request for an unknown reason
E0516 08:19:41.901974   47920 memcache.go:265] couldn't get current server API group list: the server rejected our request for an unknown reason
E0516 08:19:41.908230   47920 memcache.go:265] couldn't get current server API group list: the server rejected our request for an unknown reason
Error from server (BadRequest): the server rejected our request for an unknown reason
```
{%endspoiler%}
还是报错了，这里是需要加上https，如下:

{%spoiler 示例代码%}
```shell
[root@centos02 ~]# kubectl  get nodes -s https://192.168.93.129:6443
Please enter Username: kubernetes-admin
Please enter Password:
```
{%endspoiler%}
这次可以了，但是要求输入用户和密码，这里可以拷贝控制节点的$HOME/.kube/config文件到工作节点同路径下，拷贝后再次执行kubectl：

{%spoiler 示例代码%}
```shell
[root@centos02 ~]# kubectl get nodes -A
NAME       STATUS   ROLES           AGE   VERSION
centos02   Ready    <none>          19h   v1.27.1
node       Ready    control-plane   44h   v1.27.1
```
{%endspoiler%}
kubectl配置文件：

> 首先清楚走了那个config文件，按照下面顺序查找：
>
> 1. –kubeconfig 指定的
>
> 2. $KUBECONFIG环境变量中指定
>
> 3. ${HOME}/.kube/config



### 6、metric-server报错： remote error: tls: bad certificate

部署metrics-server后查看pods状态，发现metrics服务一直处于NotReady状态(node是控制节点，centos02是工作节点)：

{%spoiler 示例代码%}
```shell
[root@centos02-01 k8s]# kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
[root@centos02-01 k8s]# kubectl get pods -A -o wide
NAMESPACE      NAME                               READY   STATUS    RESTARTS         AGE    IP               NODE       NOMINATED NODE   READINESS GATES
default        nginx-deployment-cbdccf466-pxzn8   1/1     Running   0                31m    10.244.1.6       centos02   <none>           <none>
kube-flannel   kube-flannel-ds-b65m4              1/1     Running   0                2d1h   192.168.93.129   node       <none>           <none>
kube-flannel   kube-flannel-ds-qz478              1/1     Running   0                24h    192.168.93.136   centos02   <none>           <none>
kube-system    coredns-5d78c9869d-2f5lw           1/1     Running   3 (9h ago)       2d1h   10.244.0.9       node       <none>           <none>
kube-system    coredns-5d78c9869d-l6n8h           1/1     Running   2 (9h ago)       2d1h   10.244.0.8       node       <none>           <none>
kube-system    etcd-node                          1/1     Running   4 (9h ago)       2d1h   192.168.1.4      node       <none>           <none>
kube-system    kube-apiserver-node                1/1     Running   4 (9h ago)       2d1h   192.168.1.4      node       <none>           <none>
kube-system    kube-controller-manager-node       1/1     Running   7 (9h ago)       2d1h   192.168.1.4      node       <none>           <none>
kube-system    kube-proxy-c9h6f                   1/1     Running   0                24h    192.168.93.136   centos02   <none>           <none>
kube-system    kube-proxy-nzq26                   1/1     Running   0                2d1h   192.168.93.129   node       <none>           <none>
kube-system    kube-scheduler-node                1/1     Running   21 (10h ago)     2d1h   192.168.1.4      node       <none>           <none>
kube-system    metrics-server-7b4c4d4bfd-pdrqj    0/1     Running   11 (6h21m ago)   16h    10.244.1.5       centos02   <none>           <none>
```
{%endspoiler%}
查看两个节点的日志（/var/log/messages），发现两个节点都显示：

{%spoiler 示例代码%}
```shell
http: TLS handshake error from xx.xx.xx.xx:xxxx: remote error: tls: bad certificate
```
{%endspoiler%}
从日志可以看出是证书问题，进一步通过describe命令看下：

{%spoiler 示例代码%}
```shell
[root@centos02 ~]# kubectl logs metrics-server-7b4c4d4bfd-pdrqj -n kube-system

I0516 04:31:32.143977       1 server.go:187] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
I0516 04:31:42.153389       1 server.go:187] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0516 04:31:44.853185       1 scraper.go:140] "Failed to scrape node" err="Get \"https://192.168.93.136:10250/metrics/resource\": x509: cannot validate certificate for 192.168.93.136 because it doesn't contain any IP SANs" node="centos02"
E0516 04:31:44.855160       1 scraper.go:140] "Failed to scrape node" err="Get \"https://192.168.1.10:10250/metrics/resource\": x509: cannot validate certificate for 192.168.1.10 because it doesn't contain any IP SANs" node="node"
I0516 04:31:52.165282       1 server.go:187] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
```
{%endspoiler%}
进一步显示了证书问题的细节。

这里可以通过忽略处理，操作方法是：

{%spoiler 示例代码%}
```shell
[root@centos02-01 k8s]# kubectl edit deployment  metrics-server -n kube-system
```
{%endspoiler%}
{%spoiler 示例代码%}
```yaml
- args:
    - --cert-dir=/tmp
    - --secure-port=4443
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --kubelet-use-node-status-port
    - --metric-resolution=15s
    - --kubelet-insecure-tls # 添加这一行来忽略tls验证
```
{%endspoiler%}
保存后，过会儿再看，metrics-server状态已经正常：

{%spoiler 示例代码%}
```shell
[root@centos02-01 k8s]# kubectl get pods -A -o wide
NAMESPACE      NAME                               READY   STATUS    RESTARTS       AGE    IP               NODE       NOMINATED NODE   READINESS GATES
default        nginx-deployment-cbdccf466-pxzn8   1/1     Running   0              33m    10.244.1.6       centos02   <none>           <none>
kube-flannel   kube-flannel-ds-b65m4              1/1     Running   0              2d1h   192.168.93.129   node       <none>           <none>
kube-flannel   kube-flannel-ds-qz478              1/1     Running   0              24h    192.168.93.136   centos02   <none>           <none>
kube-system    coredns-5d78c9869d-2f5lw           1/1     Running   3 (9h ago)     2d1h   10.244.0.9       node       <none>           <none>
kube-system    coredns-5d78c9869d-l6n8h           1/1     Running   2 (9h ago)     2d1h   10.244.0.8       node       <none>           <none>
kube-system    etcd-node                          1/1     Running   4 (9h ago)     2d1h   192.168.1.4      node       <none>           <none>
kube-system    kube-apiserver-node                1/1     Running   4 (9h ago)     2d1h   192.168.1.4      node       <none>           <none>
kube-system    kube-controller-manager-node       1/1     Running   7 (9h ago)     2d1h   192.168.1.4      node       <none>           <none>
kube-system    kube-proxy-c9h6f                   1/1     Running   0              24h    192.168.93.136   centos02   <none>           <none>
kube-system    kube-proxy-nzq26                   1/1     Running   0              2d1h   192.168.93.129   node       <none>           <none>
kube-system    kube-scheduler-node                1/1     Running   21 (10h ago)   2d1h   192.168.1.4      node       <none>           <none>
kube-system    metrics-server-7db4fb59f9-c6b4l    1/1     Running   0              47s    10.244.1.7       centos02   <none>           <none>
```
{%endspoiler%}


## 参考

[安装 kubeadm](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

[容器运行时](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/)

[Kubernetes 安装小记](https://www.aneasystone.com/archives/2022/05/install-kubernetes.html)

[CentOS安装docker](https://yeasy.gitbook.io/docker_practice/install/centos)

[Kubernetes 报错："open /run/flannel/subnet.env: no such file or directory"](https://www.jianshu.com/p/9819a9f5dda0)

[connect: connection refused](https://zhuanlan.zhihu.com/p/490327659)

[通过kubeadm join 为k8s集群增加节点出错 couldn‘t validate the identity of the API Server](https://blog.csdn.net/marlinlm/article/details/122524416)

[kubernets网络插件](https://kubernetes.io/docs/concepts/cluster-administration/addons/)

[flannel文档](https://github.com/flannel-io/flannel#deploying-flannel-manually)

[kubectl:命令执行出错，Error from server (BadRequest)...](https://blog.csdn.net/u014686399/article/details/127869848)

[命令行工具 (kubectl)](https://kubernetes.io/zh-cn/docs/reference/kubectl/)

[Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server)

[k8s 监控程序Metric-server pod运行异常报：it doesn't contain any IP SANs](https://developer.aliyun.com/article/793487)

[k8s 监控程序Metric-server pod运行异常报：it doesn‘t contain any IP SANs](https://blog.csdn.net/pop_xiaohao/article/details/120699030)

[部署和访问 Kubernetes 仪表板（Dashboard）](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/web-ui-dashboard/)

[creating-sample-user](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)

[Accessing Dashboard](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md#login-not-available)

[Access control](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/README.md)

[部署并访问Dashboard](https://skyao.io/learning-kubernetes/docs/installation/kubeadm/dashboard.html)