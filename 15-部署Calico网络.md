<!-- toc -->

tags: calicoctl

# 部署 Calico 网络

kubernetes 要求集群内各节点能通过 Pod 网段互联互通，本文档介绍使用 Calico 在**所有节点** (Master、Node) 上创建互联互通的 Pod 网段的步骤。

## 使用的变量

本文档用到的变量定义如下：

``` bash
$ # 导入用到的其它全局变量：ETCD_ENDPOINTS、CALICO_ETCD_PREFIX、CLUSTER_CIDR
$ source /root/local/bin/environment.sh
$
```

## 安装和配置 calico

Kubernetes master and 每一个 Kubernetes node 需要部署 calico/node container。 同时，每个 node 必须被记录在 Calico datastore。

### 下载calicoctl

``` bash
$ # Download and install `calicoctl`
$ wget https://github.com/projectcalico/calico-containers/releases/download/v1.5.0/calicoctl 
$ sudo chmod +x calicoctl
$ sudo cp calicoctl /usr/bin
$
```

### 创建 calico-node 的 systemd unit 文件

``` bash
$ cat > calico-node.service << EOF
[Unit]
Description=calicoctl node
After=docker.service
Requires=docker.service

[Service]
User=root
Environment=ETCD_ENDPOINTS=${ETCD_ENDPOINTS}
PermissionsStartOnly=true
ExecStartPre=/usr/bin/wget -N -P /opt/bin https://github.com/projectcalico/calico-containers/releases/download/v1.5.0/calicoctl
ExecStartPre=/usr/bin/chmod +x /opt/bin/calicoctl
ExecStart=/usr/bin/calicoctl node --detach=false
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

安装并设置自启动

``` bash 
$ sudo systemctl enable /etc/systemd/system/calico-node.service
$ sudo systemctl start calico-node.service
```

### 下载并设置 Calico CNI plugins

k8s kubelet组件调用calico和calico-ipam插件。

``` bash
$ wget -N -P /opt/cni/bin https://github.com/projectcalico/calico-cni/releases/download/v1.10.0/calico
$ wget -N -P /opt/cni/bin https://github.com/projectcalico/calico-cni/releases/download/v1.10.0/calico-ipam
$ chmod +x /opt/cni/bin/calico /opt/cni/bin/calico-ipam
$
```

calico cni插件需要一个标准的cni配置文件，policy部分只有当你为networkpolicy部署calico/kube-policy-controller时被需要

``` bash
mkdir -p /etc/cni/net.d  
cat >/etc/cni/net.d/10-calico.conf <<EOF  
{  
    "name": "calico-k8s-network",  
    "type": "calico",  
    "etcd_endpoints": ${ETCD_ENDPOINTS},  
    "log_level": "info",  
    "ipam": {  
        "type": "calico-ipam"  
    },  
    "policy": {  
        "type": "k8s"  
    }  
}  
EOF  
```

### 部署calico网络策略控制器

calico/kube-policy-controller实现了k8s networkpolicy api，通过k8s api监听pod，namespace,networkpolicy事件从而做出对应的时间相应，通过rs运行为一个单独的pod

其中policy-controller.yaml 已经下载到mainfests/calico下
直接执行：

``` bash 
$ kubectl create -f policy-controller.yaml 
$
```

片刻之后，你应该可以看到网络策略控制器的运行状态

``` bash
$ kubectl get pods --namespace=kube-system
NAME                                     READY     STATUS    RESTARTS   AGE
calico-policy-controller                 2/2       Running   0          1m
$
```