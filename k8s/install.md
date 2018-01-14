# install demo k8s cluster

#### vm servers

| server name | ip           | role          |
| ----------- | ------------ | ------------- |
| k8s01       | 192.168.1.76 | master & node |
| k8s02       | 192.168.1.75 | node          |
| k8s03       | 192.168.1.74 | node          |

#### disable all server's firewall

```
systemctl disable firewalld.service
systemctl stop firewalld.service
```

#### configure all server's hostname

```
hostnamectl --static set-hostname  k8s01 #k8s02 k8s03
```

#### configure all server's  /etc/hosts

```
echo '192.168.1.76   k8s-master
192.168.1.76   etcd
192.168.1.76   registry
192.168.1.76   k8s01
192.168.1.75   k8s02
192.168.1.74   k8s03' >> /etc/hosts
```



### Configuring and Installing Master Server

#### install package

```
yum install -y docker etcd kubernetes flannel
```

#### configure etcd service

```
vi /etc/etcd/etcd.conf
[root@localhost ~]# vi /etc/etcd/etcd.conf

# [member]
ETCD_NAME=master
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_WAL_DIR=""
#ETCD_SNAPSHOT_COUNT="10000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
#ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
#ETCD_CORS=""
#
#[cluster]
#ETCD_INITIAL_ADVERTISE_PEER_URLS="http://localhost:2380"
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
#ETCD_INITIAL_CLUSTER="default=http://localhost:2380"
#ETCD_INITIAL_CLUSTER_STATE="new"
#ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://etcd:2379,http://etcd:4001"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_SRV=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
```

#### enable and restart etcd service

```
systemctl enable etcd.service
systemctl restart etcd.service
```

#### check etcd service

```
etcdctl set /testcheck ok
etcdctl get /testcheck
etcdctl rm /testcheck
etcdctl -C http://etcd:4001 cluster-health
etcdctl -C http://etcd:2379 cluster-health
```

#### configure docker

```
vi /etc/sysconfig/docker

OPTIONS='--insecure-registry registry:5000'
ADD_REGISTRY='--add-registry registry:5000'
```

#### remove docker-created bridge and iptables rules

```
iptables -t nat -F
ip link set docker0 down
ip link delete docker0
```

#### enable and restart docker

```HTML
systemctl enable docker.service
systemctl restart docker.service
```

#### configure apiserver

```
vi /etc/kubernetes/apiserver

KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"
KUBE_API_PORT="--port=8080"
KUBE_ETCD_SERVERS="--etcd-servers=http://etcd:2379"
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
```

#### configure kubernetes config

```
vi /etc/kubernetes/config

KUBE_MASTER="--master=http://k8s-master:8080"
```

#### enable and restart kubernetes services

```
systemctl enable kube-apiserver.service
systemctl restart kube-apiserver.service
systemctl enable kube-controller-manager.service
systemctl restart kube-controller-manager.service
systemctl enable kube-scheduler.service
systemctl restart kube-scheduler.service
```

#### configure kubelet

```
vi /etc/kubernetes/kubelet

KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_HOSTNAME="--hostname-override=k8s01"
KUBELET_API_SERVER="--api-servers=http://k8s-master:8080"
```

#### enable and restart kubelet services

```
systemctl enable kubelet.service
systemctl restart kubelet.service
systemctl enable kube-proxy.service
systemctl restart kube-proxy.service
```

#### check k8s status

```
kubectl get node
#or
kubectl -s http://k8s-master:8080 get node 
```

#### configure Flannel

```
vi /etc/sysconfig/flanneld

FLANNEL_ETCD_ENDPOINTS="http://etcd:2379"
```

#### create flannel  key in etcd

```
#k8s01 server
etcdctl mk /atomic.io/network/config '{ "Network": "10.0.0.0/16" }'
```

#### enable and restart Flannel，then restart docker、kubernete。

```
systemctl enable flanneld.service 
systemctl restart flanneld.service 
systemctl restart docker.service
systemctl restart kube-apiserver.service
systemctl restart kube-controller-manager.service
systemctl restart kube-scheduler.service
```

## Configuring and Installing Node Server

#### install package

```
yum install -y docker kubernetes-node flannel
```

#### configure docker

```
vi /etc/sysconfig/docker

OPTIONS='--insecure-registry registry:5000'
ADD_REGISTRY='--add-registry registry:5000'
```

#### remove docker-created bridge and iptables rules

```
iptables -t nat -F
ip link set docker0 down
ip link delete docker0
```

#### enable and restart docker

```html
systemctl enable docker
systemctl restart docker
```

#### configure kubelet

```
vi /etc/kubernetes/kubelet

KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_HOSTNAME="--hostname-override=k8s02"   #k8s03
KUBELET_API_SERVER="--api-servers=http://k8s-master:8080"
```

#### enable and restart kubelet services

```
systemctl enable kubelet.service
systemctl restart kubelet.service
systemctl enable kube-proxy.service
systemctl restart kube-proxy.service
```

#### configure Flannel

```
vi /etc/sysconfig/flanneld

FLANNEL_ETCD_ENDPOINTS="http://etcd:2379"
```

#### set and restart Flannel，then restart docker、kubelet。

```
systemctl enable flanneld.service 
systemctl start flanneld.service 
systemctl restart docker.service
systemctl restart kubelet.service
systemctl restart kube-proxy.service
```



# check k8s service

```
kubectl get nodes
kubectl get pods
kubectl run my-nginx --image=nginx --replicas=2 --port=80
kubectl describe pod
kubectl expose deployment my-nginx --port=80 --target-port=80 --type=LoadBalancer
```

#### problem:

image pull failed for registry.access.redhat.com/rhel7/pod-infrastructure:latest, this may be because there are no credentials on this request.  details: (open /etc/docker/certs.d/registry.access.redhat.com/redhat-ca.crt: no such file or directory)

```
yum install *rhsm* -y
docker pull registry.access.redhat.com/rhel7/pod-infrastructure:latest
```

