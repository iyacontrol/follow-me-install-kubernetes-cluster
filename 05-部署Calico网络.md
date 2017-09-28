<!-- toc -->

tags: calicoctl

# 部署 Calico 网络

kubernetes 要求集群内各节点能通过 Pod 网段互联互通，本文档介绍使用 Calico 在**所有节点** (Master、Node) 上创建互联互通的 Pod 网段的步骤。

## 使用的变量

本文档用到的变量定义如下：

``` bash
$ export NODE_NAME=kube-node2  # 替换为当前部署的节点名
$ # 导入用到的其它全局变量：ETCD_ENDPOINTS、CALICO_ETCD_PREFIX、CLUSTER_CIDR
$ source /usr/local/bin/environment.sh
$
```

## 安装和配置 calico

Kubernetes master and 每一个 node 需要部署 calico/node container。 同时，每个 node 必须被记录在 Calico datastore（ETCD集群）。

### 下载calicoctl

``` bash
$ # Download and install `calicoctl`
$ wget https://github.com/projectcalico/calico-containers/releases/download/v1.5.0/calicoctl 
$ sudo chmod +x calicoctl


$ # Run the calico/node container
$ # sudo ETCD_ENDPOINTS=https://10.64.3.7:2379,https://10.64.3.8:2379,https://10.64.3.86:2379 ETCD_CA_CERT_FILE=/etc/kubernetes/ssl/ca.pem ETCD_CERT_FILE=/etc/etcd/ssl/etcd.pem ETCD_KEY_FILE=/etc/etcd/ssl/etcd-key.pem ./calicoctl node run --node-image=quay.io/calico/node:v2.5.1
$ sudo ETCD_ENDPOINTS=${ETCD_ENDPOINTS} ETCD_CA_CERT_FILE=/etc/kubernetes/ssl/ca.pem ETCD_CERT_FILE=/etc/etcd/ssl/etcd.pem ETCD_KEY_FILE=/etc/etcd/ssl/etcd-key.pem ./calicoctl node run --node-image=quay.io/calico/node:v2.5.1


$ sudo cp calicoctl /usr/local/bin
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
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run --net=host --privileged --name=calico-node \
  -e ETCD_ENDPOINTS=https://10.64.3.7:2379,https://10.64.3.8:2379,https://10.64.3.86:2379 \
  -e ETCD_CA_CERT_FILE=/etc/calico/certs/ca_cert.crt \
  -e ETCD_CERT_FILE=/etc/calico/certs/cert.crt \
  -e ETCD_KEY_FILE=/etc/calico/certs/key.pem \
  -e NODENAME=kube-node1 \
  -e IP= \
  -e CALICO_IPV4POOL_CIDR=172.30.0.0/16 \
  -e CALICO_IPV4POOL_IPIP=always \
  -e CALICO_LIBNETWORK_ENABLED=true \
  -e CALICO_NETWORKING_BACKEND=bird \
  -v /var/log/calico:/var/log/calico \
  -v /var/run/calico:/var/run/calico \
  -v /lib/modules:/lib/modules \
  -v /run:/run \
  -v /run/docker/plugins:/run/docker/plugins \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /etc/kubernetes/ssl/ca.pem:/etc/calico/certs/ca_cert.crt \
  -v /etc/etcd/ssl/etcd-key.pem:/etc/calico/certs/key.pem \
  -v /etc/etcd/ssl/etcd.pem:/etc/calico/certs/cert.crt \
  quay.io/calico/node:v2.5.1
  
ExecStop=/usr/bin/docker rm -f calico-node
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

安装并设置自启动

``` bash 
$ sudo cp calico-node.service /etc/systemd/system/
$ sudo systemctl enable /etc/systemd/system/calico-node.service
$ sudo systemctl start calico-node.service
$ systemctl status calico-node.service # 查看运行状态
```

其实这个systemd unit有点复杂，我们已经把calicoctl放置到/usr/local/bin下面了，calicoctl本身封装了docker的操作。所以可以提供如下：

``` bash
$ cat > calico-node.service << EOF
[Unit]
Description=calicoctl node
After=docker.service
Requires=docker.service

[Service]
User=root
PermissionsStartOnly=true
Environment="ETCD_ENDPOINTS=https://10.64.3.7:2379,https://10.64.3.8:2379,https://10.64.3.86:2379 ETCD_CA_CERT_FILE=/etc/kubernetes/ssl/ca.pem ETCD_CERT_FILE=/etc/etcd/ssl/etcd.pem ETCD_KEY_FILE=/etc/etcd/ssl/etcd-key.pem"
ExecStart=/usr/local/bin/calicoctl node run --node-image=quay.io/calico/node:v2.5.1
  
ExecStop=/usr/bin/docker rm -f calico-node
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
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
    "etcd_endpoints": ${ETCD_ENDPOINTS}",
    "etcd_key_file": "/etc/etcd/ssl/etcd-key.pem",
    "etcd_cert_file": "/etc/etcd/ssl/etcd.pem",
    "etcd_ca_cert_file": "/etc/kubernetes/ssl/ca.pem", 
    "mtu": 1440,   
    "log_level": "info",
    "ipam": {
        "type": "calico-ipam",
        "ipv4_pools": ["172.30.0.0/16"]
    },
    "policy": {
        "type": "k8s"
    },
    "kubernetes": {
        "kubeconfig": "/etc/kubernetes/kubelet.kubeconfig"
    }
}
EOF  
```

安装标准的CNI loopback 插件

Kubernetes 要求 标准的 CNI loopback 插件。

``` bash 
wget https://github.com/containernetworking/cni/releases/download/v0.3.0/cni-v0.3.0.tgz
tar -zxvf cni-v0.3.0.tgz
sudo cp loopback /opt/cni/bin/
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
calico-policy-controller                 3/3       Running   0          1m
$
```

### 注意事项

对于kubelet启动参数需要添加：

 --network-plugin-dir=/etc/cni/net.d \
 --network-plugin=cni \
 --cni-bin-dir=/opt/cni/bin \
 --cgroup-driver=systemd \

 其中设置cgroup-driver参数，为了让kubelet和docker的这两个参数一致，否则启动不起来
 apiVersion: v1
 kind: ipPool
 metadata:
    cidr: 172.30.0.0/16
 spec:
    nat-outgoing: true