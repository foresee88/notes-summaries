## 用kubeadm在云主机上完成kubernetes集群的部署

### 资源准备
在网易云"https://c.163yun.com/dashboard#/m/nvm/"申请2台云主机，建议用CentOs镜像，下面演示也以CentOs为例。注意master节点需要2CPU或以上规格，否则master安装会报错。

### 节点初始化
2、通过下述命令初始化所有节点，安装docker并启动kubelet
```
yum install -y docker
systemctl enable docker && systemctl start docker

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet

```

### master节点安装

通过 kubeadm init 完成master节点核心组件的部署。

1. 如果人在海外，可以直接运行如下命令进行部署。
```
# --api-advertise-addresses <ip-address>
# for flannel, setup --pod-network-cidr 10.244.0.0/16
kubeadm init --pod-network-cidr 10.244.0.0/16 --kubernetes-version latest

# 设置KUBECONFIG环境变量，否则会报“The connection to the server localhost:8080 was refused - did you specify the right host or port?”的错误
export KUBECONFIG=/etc/kubernetes/admin.conf

# 允许工作负载调度到master节点上
kubectl taint nodes --all node-role.kubernetes.io/master:NoSchedule-
```
2. 如果在国内，需要通过如下方式安装

* 准备工作：
```
//在运行 kubeadm init 前可以通过 kubeadm config images pull 来测试与 "k8s.gcr.io",“k8s.io”, "gcr.io"等资源站点间的连接。
[root@k8s-master kubernetes]# kubeadm config images pull
W0816 18:12:13.288546   64208 version.go:98] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://dl.k8s.io/release/stable-1.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
W0816 18:12:13.288718   64208 version.go:99] falling back to the local client version: v1.15.2
...
//由于国内墙了"k8s.gcr.io"等资源站点，以上的命令大概率会失败。
//"https://dl.k8s.io/release/stable-1.txt"得到的是"v1.15.2",这是k8s社区当前的稳定版本（后续可能会变化）。
//通过命令"kubeadm config images list"可以查看k8s master节点需要的核心组件镜像及版本。
//这里为了不重新请求https://dl.k8s.io/release/stable-1.txt，指定了--kubernetes-version v1.15.2
[root@k8s-master-test ~]# kubeadm config images list --kubernetes-version v1.15.2
k8s.gcr.io/kube-apiserver:v1.15.2
k8s.gcr.io/kube-controller-manager:v1.15.2
k8s.gcr.io/kube-scheduler:v1.15.2
k8s.gcr.io/kube-proxy:v1.15.2
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```
kubeadm init 时很重要的一步就是拉取这些镜像，由于"k8s.gcr.io"无法访问（如果有vpn可以在这里找到所有官方镜像版本https://console.cloud.google.com/projectselector2/gcr?supportedpurview=project），不过我们可以通过dockerhub拉取到这些镜像（https://hub.docker.com）。

由于从dockerhub拉取镜像速度较慢，建议把拉取到的镜像推送到阿里云的容器镜像服务或者自建docker Registry上（可用harbor自建https://goharbor.io/），方便后续使用。镜像拉取和推送的方法如下。（这里已经替你完成了镜像准备，你也可以直接使用 "registry.cn-hangzhou.aliyuncs.com/k8s-v1152" 这个registry用于部署。）

```
//从dockerhub拉取镜像并推送到阿里云容器镜像服务的方法
//1.要开通阿里云的容器镜像仓库服务https://cr.console.aliyun.com/cn-hangzhou/instances/repositories
//2.通过以下这些命令把相关镜像下载到本地并推送到阿里云的容器镜像仓库中。
docker pull docker.io/mirrorgooglecontainers/kube-proxy:v1.15.2
docker tag docker.io/mirrorgooglecontainers/kube-proxy:v1.15.2 registry.cn-hangzhou.aliyuncs.com/k8s-v1152/kube-proxy:v1.15.2
docker push registry.cn-hangzhou.aliyuncs.com/k8s-v1152/kube-proxy:v1.15.2

docker pull docker.io/mirrorgooglecontainers/kube-apiserver:v1.15.2
docker tag docker.io/mirrorgooglecontainers/kube-apiserver:v1.15.2 registry.cn-hangzhou.aliyuncs.com/k8s-v1152/kube-apiserver:v1.15.2
docker push registry.cn-hangzhou.aliyuncs.com/k8s-v1152/kube-apiserver:v1.15.2

docker docker.io/mirrorgooglecontainers/kube-controller-manager:v1.15.2-beta.0
docker tag docker.io/mirrorgooglecontainers/kube-controller-manager:v1.15.2-beta.0 registry.cn-hangzhou.aliyuncs.com/k8s-v1152/kube-controller-manager:v1.15.2
docker push registry.cn-hangzhou.aliyuncs.com/k8s-v1152/kube-controller-manager:v1.15.2

docker pull docker.io/mirrorgooglecontainers/kube-scheduler:v1.15.2-beta.0
docker tag docker.io/mirrorgooglecontainers/kube-scheduler:v1.15.2-beta.0 registry.cn-hangzhou.aliyuncs.com/k8s-v1152/kube-scheduler:v1.15.2
docker push registry.cn-hangzhou.aliyuncs.com/k8s-v1152/kube-scheduler:v1.15.2

docker pull docker.io/coredns/coredns:1.3.1
docker tag docker.io/coredns/coredns:1.3.1 registry.cn-hangzhou.aliyuncs.com/k8s-v1152/coredns:1.3.1
docker push registry.cn-hangzhou.aliyuncs.com/k8s-v1152/coredns:1.3.1

docker pull docker.io/mirrorgooglecontainers/etcd:3.3.10
docker tag docker.io/mirrorgooglecontainers/etcd:3.3.10 registry.cn-hangzhou.aliyuncs.com/k8s-v1152/etcd:3.3.10
docker push registry.cn-hangzhou.aliyuncs.com/k8s-v1152/etcd:3.3.10

docker pull docker.io/mirrorgooglecontainers/pause:3.1
docker tag docker.io/mirrorgooglecontainers/pause:3.1 registry.cn-hangzhou.aliyuncs.com/k8s-v1152/pause:3.1
docker push registry.cn-hangzhou.aliyuncs.com/k8s-v1152/pause:3.1
```
* 两种部署方法

1）用命令行参数的方式
     
```
//--api-advertise-addresses 是master节点的地址
//使用flannel需要设置 --pod-network-cidr 10.244.0.0/16
//要绕开"k8s.gcr.io"，需要指定--kubernetes-version和--image-repository
kubeadm init --pod-network-cidr 10.244.0.0/16 --kubernetes-version v1.15.2 --apiserver-advertise-address 192.168.0.2 --image-repository registry.cn-hangzhou.aliyuncs.com/k8s-v1152 
```
2）用配置文件的方式

```
//生成配置文件
[root@k8s-master ~]# kubeadm config print init-defaults ClusterConfiguration > kubeadm.conf
[root@k8s-master ~]# cat kubeadm.conf
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master.localdomain
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.14.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}


vim kubeadm.conf
# 修改 imageRepository: k8s.gcr.io
imageRepository: registry.cn-hangzhou.aliyuncs.com/k8s-v1152
# 修改kubernetes版本kubernetesVersion: v1.14.0
kubernetesVersion: v1.15.2

# 执行如下命令
kubeadm init --config kubeadm.conf

```
master节点部署完成后，下面的打印要重点关注一下。
```
...
Your Kubernetes control-plane has initialized successfully!

# 执行下述命令后可正常使用kubectl工具，否则会报“The connection to the server localhost:8080 was refused - did you specify the right host or port?”
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 提示安装网络插件，详见“安装flannel”小节。
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

# 如何把node节点加入集群，详见“添加node节点”章节。
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.2:6443 --token mkxzf6.2d1lfi8pgxyk9kd4 \
    --discovery-token-ca-cert-hash sha256:6dcc27d6ff227bb007180c43beb0628f60dfec6b6ea647489d0b8b0515865ea3
```

### 安装flannel插件（node节点也需要安装）

```
mkdir -p /etc/cni/net.d
cat >/etc/cni/net.d/10-mynet.conf <<-EOF
{
    "cniVersion": "0.3.0",
    "name": "mynet",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.244.0.0/16",
        "routes": [
            {"dst": "0.0.0.0/0"}
        ]
    }
}
EOF

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml

```

### node节点加入集群

```
// 其中192.168.0.2:6443是master节点的地址和端口
// token一般24小时有效，可通过kubeadm token list查看，如果已失效，通过 kubeadm token create重新创建
// --discovery-token-ca-cert-hash通过命令 "openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'" 获取
kubeadm join 192.168.0.2:6443 --token mkxzf6.2d1lfi8pgxyk9kd4 \
    --discovery-token-ca-cert-hash sha256:6dcc27d6ff227bb007180c43beb0628f60dfec6b6ea647489d0b8b0515865ea3
    
  
// 在node节点执行kubectl会报“The connection to the server localhost:8080 was refused - did you specify the right host or port?”, 可以把master上 $HOME/.kube/config 拷贝到node上相同的位置,然后执行即可成功。
[root@k8s-node1 ~]# kubectl get nodes
NAME                     STATUS   ROLES    AGE     VERSION
k8s-master.localdomain   Ready    master   6h1m    v1.15.2
k8s-node1.localdomain    Ready    <none>   3h47m   v1.15.2
    
```
### 应用部署测试

```
// ng-dep.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
[root@k8s-master ~]# kubectl apply -f ng-dep
deployment.apps/nginx-deployment1 created
[root@k8s-master ~]# kubectl get pods -o wide
NAME                                 READY   STATUS    RESTARTS   AGE     IP            NODE                     NOMINATED NODE   READINESS GATES
nginx-deployment-68c7f5464c-gmnh9    1/1     Running   0          8s      10.244.0.11   k8s-master.localdomain   <none>           <none>
nginx-deployment-68c7f5464c-k96sf    1/1     Running   0          8s      10.244.1.8    k8s-node1.localdomain    <none>           <none>
```

### 测试遇到的两个问题
1、flannel安装的问题
创建pod时会报 "cni0" already has an IP address different from 10.244.0.1/24"的错误，导致pod创建不成功。

解决方法：这是因为daemonset无法重建cni网络配置，可以手动执行ip addr del 10.244.0.1/24 dev cni0 解决。
相关讨论 https://github.com/containernetworking/cni/issues/306

2、创建pod时会报 k8s.gcr.io/pause:3.1 镜像拉不下来，导致pod创建不成功

解决方法：之前运行 “kubeadm init --image-repository registry.cn-hangzhou.aliyuncs.com/k8s-v1152 ...” 时我们已经把
registry.cn-hangzhou.aliyuncs.com/k8s-v1152/pause:3.1 这个镜像拉到本地了，通过以下命令解决。
```
docker tag registry.cn-hangzhou.aliyuncs.com/k8s-v1152/pause:3.1 k8s.gcr.io/pause:3.1
```

### 参考资料
1. https://kubernetes.feisky.xyz/bu-shu-pei-zhi/cluster/kubeadm
2. https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/
2. https://blog.csdn.net/nklinsirui/article/details/80581286
3. https://my.oschina.net/Kanonpy/blog/3006129
