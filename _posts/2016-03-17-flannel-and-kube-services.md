---
layout: post
title: flannel and kubernetes services network implementation
date: 2016-03-17 14:00:30
categories: Linux
tags: network nat iptables
excerpt: flannel and kubernetes services network implementation
---

# Environment

```
172.17.42.30 kube-master
172.17.42.31 kube-node1
172.17.42.32 kube-node2


/usr/bin/kube-apiserver --logtostderr=true --v=0 --etcd-servers=http://kube-master:2379 --insecure-bind-address=127.0.0.1 --secure-port=443 --allow-privileged=true --service-cluster-ip-range=10.254.0.0/16 --admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota --tls-cert-file=/etc/kubernetes/certs/server.crt --tls-private-key-file=/etc/kubernetes/certs/server.key --client-ca-file=/etc/kubernetes/certs/ca.crt --token-auth-file=/etc/kubernetes/tokens/known_tokens.csv --service-account-key-file=/etc/kubernetes/certs/server.crt
```

# flannel

## implementation

* node1

```sh
[root@kube-node1 ~]# ip route show
default via 172.17.42.1 dev eth0 
172.16.0.0/12 dev flannel.1  proto kernel  scope link  src 172.16.28.0 
172.16.28.0/24 dev docker0  proto kernel  scope link  src 172.16.28.1 
172.17.0.0/16 dev eth0  proto kernel  scope link  src 172.17.42.31

[root@kube-node1 ~]# iptables-save -t nat
# Generated by iptables-save v1.4.21 on Thu Mar 17 04:26:56 2016
*nat
-A POSTROUTING -s 172.16.28.0/24 ! -o docker0 -j MASQUERADE

[root@kube-node1 ~]# ip a
5: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN 
    link/ether b2:69:2c:67:63:b8 brd ff:ff:ff:ff:ff:ff
    inet 172.16.28.0/12 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::b069:2cff:fe67:63b8/64 scope link 
       valid_lft forever preferred_lft forever
6: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 02:42:fb:83:70:4d brd ff:ff:ff:ff:ff:ff
    inet 172.16.28.1/24 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:fbff:fe83:704d/64 scope link 
       valid_lft forever preferred_lft forever


### flannel.1 is vxlan device
[root@kube-node1 ~]# ip -d link show flannel.1
5: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT 
    link/ether b2:69:2c:67:63:b8 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    vxlan id 1 local 172.17.42.31 dev eth0 srcport 32768 61000 dstport 8472 nolearning ageing 300 
```

* node2

```sh
[root@kube-node2 ~]# ip route show
default via 172.17.42.1 dev eth0 
172.16.0.0/12 dev flannel.1  proto kernel  scope link  src 172.16.78.0 
172.16.78.0/24 dev docker0  proto kernel  scope link  src 172.16.78.1 
172.17.0.0/16 dev eth0  proto kernel  scope link  src 172.17.42.32

[root@kube-node2 ~]# iptables-save -t nat
# Generated by iptables-save v1.4.21 on Thu Mar 17 04:30:51 2016
*nat
-A POSTROUTING -s 172.16.78.0/24 ! -o docker0 -j MASQUERADE
```

* network topology

![](/assets/flannel/flanne-and-kube-service-00.jpg)


* packet flow

假设当sshd-2访问nginx-0：

当packet{172.16.28.5:port => 172.16.78.9:80} 到达docker0时，根据路由规则：

```
172.16.0.0/12 dev flannel.1  proto kernel  scope link  src 172.16.28.0 
```

packet将选择flannel.1作为出口，同时，根据iptables SNAT规则，将packet的源IP地址改为flannel.1的地址(172.16.28.0/12)。flannel.1是一个VXLAN设备，将packet进行隧道封包，然后发到node2。node2解包，然后根据路由规则：

```
172.16.78.0/24 dev docker0  proto kernel  scope link  src 172.16.78.1 
```
从接口docker0发出，再转给nginx-0。

在node2上对VXLAN port抓包：
![](/assets/flannel/flanne-and-kube-service-01.png)


* difference between flannel and docker overlay

flannel中连接容器的bridge（docker0）与vxlan设备（flannel.1）是相互独立的，而docker overlay是将vxlan设备直接作为bridge的端口，即flannel中docker0是NAT bridge，docker overlay是Host bridge。

这样的缺点是当跨节点的容器通信时，看不到对方的IP。比如nginx-0中看到源地IP地址是flannel.1的IP地址。

这样的优点是从node可以直接访问容器。比如可以直接从node1访问nginx-0。这一点是kubernetes中的services的基础。Docker overlay中的host是不能直接与容器通信的。


# kubernetes service

## test service

* create rc

```sh
# cat replication.yml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    app: my-nginx
  template:
    metadata:
      name: nginx
      labels:
        app: my-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.0
        ports:
        - containerPort: 80

# kubectl create -f ./replication.yml
# kubectl get rc                      
CONTROLLER   CONTAINER(S)   IMAGE(S)    SELECTOR       REPLICAS   AGE
nginx        nginx          nginx:1.0   app=my-nginx   2          3s

# kubectl describe replicationcontrollers/nginx
Name:           nginx
Namespace:      default
Image(s):       nginx:1.0
Selector:       app=my-nginx
Labels:         app=my-nginx
Replicas:       2 current / 2 desired
Pods Status:    2 Running / 0 Waiting / 0 Succeeded / 0 Failed
No volumes.
Events:
  FirstSeen     LastSeen        Count   From                            SubobjectPath   Type            Reason                  Message
  ---------     --------        -----   ----                            -------------   --------        ------                  -------
  5m            5m              1       {replication-controller }                       Normal          SuccessfulCreate        Created pod: nginx-x5qqd
  5m            5m              1       {replication-controller }                       Normal          SuccessfulCreate        Created pod: nginx-wzcbn
```


* create service

```sh
# cat service.yml 
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "my-nginx-service"
    },
    "spec": {
        "selector": {
            "app": "my-nginx"
        },
        "ports": [
            {
                "protocol": "TCP",
                "port": 8080,
                "targetPort": 80
            }
        ]
    }
}


# kubectl create -f ./service.yml 
service "my-nginx-service" created
# kubectl get services
NAME               CLUSTER-IP       EXTERNAL-IP   PORT(S)    SELECTOR       AGE
kubernetes         10.254.0.1       <none>        443/TCP    <none>         27d
my-nginx-service   10.254.247.121   <none>        8080/TCP   app=my-nginx   11s


# kubectl describe services/my-nginx-service   
Name:                   my-nginx-service
Namespace:              default
Labels:                 <none>
Selector:               app=my-nginx
Type:                   ClusterIP
IP:                     10.254.247.121
Port:                   <unnamed>       8080/TCP
Endpoints:              172.16.78.10:80,172.16.78.9:80
Session Affinity:       None
No events.
```

可以看到，service分配的VIP为10.254.247.121。

## service implementation

如果不了解service，可以先参考[Kubernetes解析：services](http://hustcat.github.io/kubernetes-services/)。

有两个注意点：

（1)kube会在host上创建2个nat iptables规则，其中一个在PREROUTING链，影响Host上的容器访问service；另外一个规则在OUTPUT链，影响Host本身访问service。

（2)kube-proxy会监听一个端口。当Host上的容器（或者Host本身）访问service时，iptables将请求转到本机的kube-proxy，然后kube-proxy再转给service对应的实际（本机或者其它Host的）容器（从本机的flannel.1接口发出）。

从这里可以看到，kube service要求Host能够访问所有的容器，上一节中已经介绍flannel满足了这种要求。

* node1 rule

```sh
[root@kube-node1 ~]# iptables-save -t nat
-A PREROUTING -m comment --comment "handle ClusterIPs; NOTE: this must be before the NodePort rules" -j KUBE-PORTALS-CONTAINER
-A OUTPUT -m comment --comment "handle ClusterIPs; NOTE: this must be before the NodePort rules" -j KUBE-PORTALS-HOST

## 影响从Host上的容器访问service
-A KUBE-PORTALS-CONTAINER -d 10.254.247.121/32 -p tcp -m comment --comment "default/my-nginx-service:" -m tcp --dport 8080 -j REDIRECT --to-ports 50948

## 影响从Host访问service
-A KUBE-PORTALS-HOST -d 10.254.247.121/32 -p tcp -m comment --comment "default/my-nginx-service:" -m tcp --dport 8080 -j DNAT --to-destination 172.17.42.31:50948


[root@kube-node1 ~]# netstat -ltnp
tcp6       0      0 :::50948                :::*                    LISTEN      2615/kube-proxy
```

* node2 rule

```sh
[root@kube-node2 ~]# iptables-save -t nat
-A PREROUTING -m comment --comment "handle ClusterIPs; NOTE: this must be before the NodePort rules" -j KUBE-PORTALS-CONTAINER
-A OUTPUT -m comment --comment "handle ClusterIPs; NOTE: this must be before the NodePort rules" -j KUBE-PORTALS-HOST

-A KUBE-PORTALS-CONTAINER -d 10.254.247.121/32 -p tcp -m comment --comment "default/my-nginx-service:" -m tcp --dport 8080 -j REDIRECT --to-ports 55231
-A KUBE-PORTALS-HOST -d 10.254.247.121/32 -p tcp -m comment --comment "default/my-nginx-service:" -m tcp --dport 8080 -j DNAT --to-destination 172.17.42.32:55231

[root@kube-node2 ~]# netstat -tlnp|grep kube-proxy
tcp6       0      0 :::55231                :::*                    LISTEN      1651/kube-proxy
```

## access test

### container access service

```sh
[root@sshd-2 ~]# ip a
13: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 02:42:ac:10:1c:05 brd ff:ff:ff:ff:ff:ff
    inet 172.16.28.5/24 scope global eth0
       valid_lft forever preferred_lft forever

[root@sshd-2 ~]# telnet 10.254.247.121 8080
Trying 10.254.247.121...
Connected to 10.254.247.121.
Escape character is '^]'.

[root@sshd-2 ~]# netstat -tnp
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address               Foreign Address             State       PID/Program name           
tcp        0      0 172.16.28.5:38380           10.254.247.121:8080         ESTABLISHED 48/telnet  
```

Host's socket：

```sh
[root@kube-node1 ~]# netstat -tnp|grep kube-proxy 
tcp        0      0 172.16.28.0:59023       172.16.78.10:80         ESTABLISHED 2615/kube-proxy     
tcp6       0      0 172.16.28.1:50948       172.16.28.5:38380       ESTABLISHED 2615/kube-proxy  
```
上面的第一个连接为kube-proxy与nginx-1连接。第二个为telnet与kube-proxy的连接。

packet flow:

```
{SPod_IP: SPod_port -> VIP:VPort } => {SPod_IP: SPod_port -> Docker0_IP:Proxy_port }  => {flannel.1_IP: flannel.1_port -> DPod_IP:DPod_port}
```

### node access service

* node1 -> service:

```sh
[root@kube-node1 ~]# telnet  10.254.247.121 8080
Trying 10.254.247.121...
Connected to 10.254.247.121.
Escape character is '^]'.

[root@kube-node1 ~]# netstat -ntp|grep telnet
tcp        0      0 172.17.42.31:51498      10.254.247.121:8080     ESTABLISHED 15524/telnet

[root@kube-node1 ~]# netstat -ntp|grep proxy
tcp6       0      0 172.17.42.31:50948      172.17.42.31:51498      ESTABLISHED 2615/kube-proxy
tcp        0      0 172.16.28.0:59012       172.16.78.10:80         ESTABLISHED 2615/kube-proxy
```

kube-prox的第一个连接为telnet与kube-proxy的连接，第二个为kube-proxy与nginx-1的连接。

* node2 -> service:

```sh
[root@kube-node2 ~]# telnet  10.254.247.121 8080
Trying 10.254.247.121...
Connected to 10.254.247.121.
Escape character is '^]'.

[root@kube-node2 ~]# netstat -tnp|grep telnet
tcp        0      0 172.17.42.32:46588      10.254.247.121:8080     ESTABLISHED 25878/telnet

[root@kube-node2 ~]# netstat -tnp|grep kube-proxy
tcp        0      0 172.16.78.1:56942       172.16.78.10:80         ESTABLISHED 1651/kube-proxy     
tcp6       0      0 172.17.42.32:55231      172.17.42.32:46588      ESTABLISHED 1651/kube-proxy
```

与node1的区别在于node2中，kube-proxy与nginx-1的连接的源IP为docker0的IP。node1中为flannel.1的IP。

# related posts
* [flannel](https://github.com/coreos/flannel)
* [user guide: Services](http://kubernetes.io/docs/user-guide/services/)
* [use iptables for proxying instead of userspace](https://github.com/kubernetes/kubernetes/issues/3760)
* [Enable iptables kube-proxy by default in master](https://github.com/kubernetes/kubernetes/pull/16344)