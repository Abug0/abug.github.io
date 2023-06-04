# minikube启动问题

## 1、dashboard外部访问

如下，开启dashboard后，只监听了本地地址：

```shell
[root@centos02-01-ssd ~]# minikube dashboard
* Enabling dashboard ...
  - Using image docker.io/kubernetesui/dashboard:v2.7.0
  - Using image docker.io/kubernetesui/metrics-scraper:v1.0.8
* Some dashboard features require the metrics-server addon. To enable all features please run:
	minikube addons enable metrics-server	
* Verifying dashboard health ...
* Launching proxy ...
* Verifying proxy health ...
http://127.0.0.1:38987/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```

此时可开启代理：

```shell
[root@centos02-01-ssd ~]# kubectl proxy --address='0.0.0.0' --disable-filter=true
W0410 19:37:57.675250  126941 proxy.go:175] Request filter disabled, your proxy is vulnerable to XSRF attacks, please be cautious
Starting to serve on [::]:8001
```

替换ip和端口后即可访问：

![image-20230411203903032](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/Typora20230411203910.png)

## 2、metrics-server问题

```shell
[root@centos02-01-ssd ~]# kubectl get pods
E0410 02:41:01.509557   41565 memcache.go:255] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0410 02:41:01.664696   41565 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0410 02:41:01.732327   41565 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0410 02:41:01.757328   41565 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
No resources found in default namespace.
```

执行get pod来查找命令：

```shell
kubectl get pods -n kube-system
```

结果如下：

```shell
E0410 11:32:34.128252   79846 memcache.go:255] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0410 11:32:34.129985   79846 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0410 11:32:34.132637   79846 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0410 11:32:34.134949   79846 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
NAME                               READY   STATUS         RESTARTS        AGE
coredns-787d4945fb-cmx6d           1/1     Running        6 (5h43m ago)   24d
etcd-minikube                      1/1     Running        4 (5h43m ago)   24d
kindnet-8bdq4                      1/1     Running        9 (113m ago)    24d
kube-apiserver-minikube            1/1     Running        7 (112m ago)    24d
kube-controller-manager-minikube   1/1     Running        9 (5h42m ago)   24d
kube-proxy-dtpnk                   1/1     Running        3 (8h ago)      24d
kube-scheduler-minikube            1/1     Running        6 (5h43m ago)   24d
metrics-server-5f8fcc9bb7-7cvll    0/1     ErrImagePull   0               15m
storage-provisioner                1/1     Running        15              24d
```

进一步describle命令：

```shell
kubectl describe pod metrics-server-5f8fcc9bb7-7cvll -n kube-system
```

结果如下：

```shell
E0410 11:33:11.602283   79887 memcache.go:255] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0410 11:33:11.629565   79887 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0410 11:33:11.633242   79887 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0410 11:33:11.635276   79887 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
Name:                 metrics-server-5f8fcc9bb7-7cvll
Namespace:            kube-system
Priority:             2000000000
Priority Class Name:  system-cluster-critical
Service Account:      metrics-server
Node:                 minikube/192.168.49.2
Start Time:           Mon, 10 Apr 2023 11:16:47 +0800
Labels:               k8s-app=metrics-server
                      pod-template-hash=5f8fcc9bb7
Annotations:          <none>
Status:               Pending
IP:                   10.244.0.10
IPs:
  IP:           10.244.0.10
Controlled By:  ReplicaSet/metrics-server-5f8fcc9bb7
Containers:
  metrics-server:
    Container ID:  
    Image:         registry.k8s.io/metrics-server/metrics-server:v0.6.2@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2
    Image ID:      
    Port:          4443/TCP
    Host Port:     0/TCP
    Args:
      --cert-dir=/tmp
      --secure-port=4443
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
      --kubelet-use-node-status-port
      --metric-resolution=60s
      --kubelet-insecure-tls
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Requests:
      cpu:        100m
      memory:     200Mi
    Liveness:     http-get https://:https/livez delay=0s timeout=1s period=10s #success=1 #failure=3
    Readiness:    http-get https://:https/readyz delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /tmp from tmp-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9rlrp (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  tmp-dir:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
  kube-api-access-9rlrp:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  16m                   default-scheduler  Successfully assigned kube-system/metrics-server-5f8fcc9bb7-7cvll to minikube
  Warning  Failed     15m                   kubelet            Failed to pull image "registry.k8s.io/metrics-server/metrics-server:v0.6.2@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": rpc error: code = Unknown desc = failed to pull and unpack image "registry.k8s.io/metrics-server/metrics-server@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": failed to resolve reference "registry.k8s.io/metrics-server/metrics-server@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": failed to do request: Head "https://registry.k8s.io/v2/metrics-server/metrics-server/manifests/sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": dial tcp: lookup registry.k8s.io on 192.168.49.1:53: read udp 192.168.49.2:45360->192.168.49.1:53: i/o timeout
  Warning  Failed     14m                   kubelet            Failed to pull image "registry.k8s.io/metrics-server/metrics-server:v0.6.2@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": rpc error: code = Unknown desc = failed to pull and unpack image "registry.k8s.io/metrics-server/metrics-server@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": failed to resolve reference "registry.k8s.io/metrics-server/metrics-server@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": failed to do request: Head "https://registry.k8s.io/v2/metrics-server/metrics-server/manifests/sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": dial tcp: lookup registry.k8s.io on 192.168.49.1:53: read udp 192.168.49.2:46766->192.168.49.1:53: i/o timeout
  Warning  Failed     13m                   kubelet            Failed to pull image "registry.k8s.io/metrics-server/metrics-server:v0.6.2@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": rpc error: code = Unknown desc = failed to pull and unpack image "registry.k8s.io/metrics-server/metrics-server@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": failed to resolve reference "registry.k8s.io/metrics-server/metrics-server@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": failed to do request: Head "https://registry.k8s.io/v2/metrics-server/metrics-server/manifests/sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": dial tcp: lookup registry.k8s.io on 192.168.49.1:53: read udp 192.168.49.2:36366->192.168.49.1:53: i/o timeout
  Normal   Pulling    12m (x4 over 16m)     kubelet            Pulling image "registry.k8s.io/metrics-server/metrics-server:v0.6.2@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2"
  Warning  Failed     12m (x4 over 15m)     kubelet            Error: ErrImagePull
  Warning  Failed     12m                   kubelet            Failed to pull image "registry.k8s.io/metrics-server/metrics-server:v0.6.2@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": rpc error: code = Unknown desc = failed to pull and unpack image "registry.k8s.io/metrics-server/metrics-server@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": failed to resolve reference "registry.k8s.io/metrics-server/metrics-server@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": failed to do request: Head "https://registry.k8s.io/v2/metrics-server/metrics-server/manifests/sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": dial tcp: lookup registry.k8s.io on 192.168.49.1:53: read udp 192.168.49.2:32960->192.168.49.1:53: i/o timeout
  Warning  Failed     11m (x6 over 15m)     kubelet            Error: ImagePullBackOff
  Normal   BackOff    6m13s (x25 over 15m)  kubelet            Back-off pulling image "registry.k8s.io/metrics-server/metrics-server:v0.6.2@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2"
  Warning  Failed     52s                   kubelet            Failed to pull image "registry.k8s.io/metrics-server/metrics-server:v0.6.2@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": rpc error: code = Unknown desc = failed to pull and unpack image "registry.k8s.io/metrics-server/metrics-server@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": failed to resolve reference "registry.k8s.io/metrics-server/metrics-server@sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": failed to do request: Head "https://registry.k8s.io/v2/metrics-server/metrics-server/manifests/sha256:f977ad859fb500c1302d9c3428c6271db031bb7431e7076213b676b345a88dc2": dial tcp: lookup registry.k8s.io on 192.168.49.1:53: read udp 192.168.49.2:39894->192.168.49.1:53: i/o timeout
```

此时在dashboard上也可看到ImagePullBackOff。

所以很明显是由于无法访问registry.k8s.io，从而拉取不到metrics-server:v0.6.2镜像导致。

### 解决方案

1、手动拉取镜像，见参考链接（实操由于minikube虚拟机无法访问外网故而放弃）；

2、VMWare虚拟机（minikube所在的宿主机）配置代理，通过物理机代理访问外网，删除minikube虚拟机，重新启动即可（这一步重新enable metrics-server应该就可以了...未证实过）。

完成后重新执行命令：

```shell
[root@centos02-01-ssd ~]# kubectl cluster-info
Kubernetes control plane is running at https://192.168.76.2:8443
CoreDNS is running at https://192.168.76.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

[root@centos02-01-ssd ~]# kubectl top pods
No resources found in default namespace.

[root@centos02-01-ssd ~]# kubectl get pods
No resources found in default namespace.
```

## 3、重启minikube出错

执行delete后，start出错：

```shell
[root@centos02-01-ssd ~]# minikube delete --all
[root@centos02-01-ssd ~]# minikube start --force
* minikube v1.29.0 on Centos 7.9.2009
! minikube skips various validations when --force is supplied; this may lead to unexpected behavior

* Automatically selected the docker driver. Other choices: ssh, none
* The "docker" driver should not be used with root privileges. If you wish to continue as root, use --force.
* If you are running minikube within a VM, consider using --driver=none:
*   https://minikube.sigs.k8s.io/docs/reference/drivers/none/
* Using Docker driver with root privileges
* Starting control plane node minikube in cluster minikube
* Pulling base image ...
    > gcr.io/k8s-minikube/kicbase...:  39.51 MiB / 407.19 MiB  9.70% 80.63 KiB 
    > gcr.io/k8s-minikube/kicbase...:  39.51 MiB / 407.19 MiB  9.70% 89.25 KiB 
    > gcr.io/k8s-minikube/kicbase...:  80.54 MiB / 407.19 MiB  19.78% 365.52 Ki
! minikube was unable to download gcr.io/k8s-minikube/kicbase:v0.0.37, but successfully downloaded docker.io/kicbase/stable:v0.0.37 as a fallback image
* Creating docker container (CPUs=2, Memory=2200MB) ...| E0410 19:30:18.021902  115247 network_create.go:112] error while trying to create docker network minikube 192.168.67.0/24: create docker network minikube 192.168.67.0/24 with gateway 192.168.67.1 and MTU of 1500: docker network create --driver=bridge --subnet=192.168.67.0/24 --gateway=192.168.67.1 -o --ip-masq -o --icc -o com.docker.network.driver.mtu=1500 --label=created_by.minikube.sigs.k8s.io=true --label=name.minikube.sigs.k8s.io=minikube minikube: exit status 1
stdout:

stderr:
Error response from daemon: Failed to Setup IP tables: Unable to enable SKIP DNAT rule:  (iptables failed: iptables --wait -t nat -I DOCKER -i br-1528d16c7c9d -j RETURN: iptables: No chain/target/match by that name.
 (exit status 1))                                                                                                                                                                                 
! Unable to create dedicated network, this might result in cluster IP change after restart: un-retryable: create docker network minikube 192.168.67.0/24 with gateway 192.168.67.1 and MTU of 1500: docker network create --driver=bridge --subnet=192.168.67.0/24 --gateway=192.168.67.1 -o --ip-masq -o --icc -o com.docker.network.driver.mtu=1500 --label=created_by.minikube.sigs.k8s.io=true --label=name.minikube.sigs.k8s.io=minikube minikube: exit status 1
stdout:

stderr:
Error response from daemon: Failed to Setup IP tables: Unable to enable SKIP DNAT rule:  (iptables failed: iptables --wait -t nat -I DOCKER -i br-1528d16c7c9d -j RETURN: iptables: No chain/target/match by that name.
 (exit status 1))

! StartHost failed, but will try again: creating host: create: creating: create kic node: create container: docker run -d -t --privileged --security-opt seccomp=unconfined --tmpfs /tmp --tmpfs /run -v /lib/modules:/lib/modules:ro --hostname minikube --name minikube --label created_by.minikube.sigs.k8s.io=true --label name.minikube.sigs.k8s.io=minikube --label role.minikube.sigs.k8s.io= --label mode.minikube.sigs.k8s.io=minikube --volume minikube:/var --security-opt apparmor=unconfined --memory=2200mb --cpus=2 -e container=docker --expose 8443 --publish=127.0.0.1::8443 --publish=127.0.0.1::22 --publish=127.0.0.1::2376 --publish=127.0.0.1::5000 --publish=127.0.0.1::32443 docker.io/kicbase/stable:v0.0.37@sha256:8bf7a0e8a062bc5e2b71d28b35bfa9cc862d9220e234e86176b3785f685d8b15: exit status 125
stdout:
79fd5fd76c6f6c51655dcb038239f1de6e48fd853db0db4584933faa71502979
stderr:
docker: Error response from daemon: driver failed programming external connectivity on endpoint minikube (222ce95cbdd7c0d070b9556fe085ff0795be9ab18424a265060638dbf0f9a0be):  (iptables failed: iptables --wait -t nat -A DOCKER -p tcp -d 127.0.0.1 --dport 32812 -j DNAT --to-destination 172.17.0.3:32443 ! -i docker0: iptables: No chain/target/match by that name.
 (exit status 1)).

* docker "minikube" container is missing, will recreate.
```

此时重启一下docker服务即可。

## 参考

[处理安装 metrics-server 时出现的 ImagePullBackOff 错误](https://www.mls-tech.info/microservice/k8s/minikube-use-metrics-server/)

[minikube安装k8s过程中ImagePullBackOff问题处理](https://juejin.cn/post/7166878805766701092)