## Kubernetes Service基本使用和功能介绍
### 简介
在k8s中，pod很可能受到各种因素影响（宿主机、网络等故障,集群扩缩容等）被上层的控制器在未通知用户的情况下删除或重建，重建pod会导致它的虚拟ip发生改变，因此一组pods（通常由deployment管理）如何以一种稳定状态对外提供服务，对依赖这组pod所提供服务的其他服务来说至关重要。这个答案就是service。

### 演示
service有ClusterIP、NodePort、LoadBalancer、ExternalName四种类型。下面我们在[Docker Desktop](https://www.docker.com/products/docker-desktop)环境下演示一下通过不同类型service访问一个以deployment形态部署的双副本nginx服务的的例子，来了解一下这几种类型的基本使用和差异。

1. 以deployment部署一个两副本的nginx服务（模拟一个普通的应用服务）
```
//创建如下ng-dep.yaml文件
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

//使用如下命令部署
kubectl create -f ng-dep.yaml
//查看创建结果
foreseedeMacBook-Pro:~ foresee$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           19h
foreseedeMacBook-Pro:~ foresee$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-64fc4c755d-kxkx4   1/1     Running   0          19h
nginx-deployment-64fc4c755d-tpkg9   1/1     Running   0          19h
```

2. 通过spec.type:ClusterIP类型的service来暴露nginx(ClusterIP是默认类型，不需要显示指定)
```
//创建ng-sev.yaml文件
apiVersion: v1
kind: Service
metadata:
  name: nginx-sev-clusterip
  labels:
   name: nginx-sev-clusterip
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx

//使用如下命令部署
kubectl create -f ng-sev.yaml
// 查看创建结果
foreseedeMacBook-Pro:~ foresee$ kubectl get service
NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-sev-clusterip   ClusterIP   10.98.110.72   <none>        80/TCP    28m

//由于clusterip类型的service只能被集群内部的pod所访问到，因此要验证访问只能创建pod，通过pod来访问进行测试。
//下面命令创建一个带busybox工具的pod,命令说明详见https://kubernetes.io/blog/2015/10/some-things-you-didnt-know-about-kubectl_28/
kubectl run -i --tty busybox --image=busybox --restart=Never -- sh

//在创建出来的pod上测试clusterip类型的service访问
/ # wget -qO - 10.98.110.72
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</body>
</html>
/ #

//也可以通过kube-dns记录的service名称来访问，如果要跨namespace访问，需要带上目标service的ns，比如nginx-sev-clusterip.default
/ # wget -qO - nginx-sev-clusterip
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</body>
</html>
/ #
```

3. 通过spec.type:NodePort类型的service来暴露nginx
```
//创建ng-sev-nodeport.yaml文件
apiVersion: v1
kind: Service
metadata:
  name: nginx-sev-nodeport
  labels:
   name: nginx-sev-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30088
  selector:
    app: nginx

//使用如下命令部署
kubectl create -f ng-sev-nodeport.yaml

// 查看部署结果
foreseedeMacBook-Pro:~ foresee$ kubectl get service
NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-sev-nodeport   NodePort    10.102.56.80   <none>        80:30088/TCP   9s

//查看service的详细信息，比对nginx-sev-clusterip，发现多了“LoadBalancer Ingress: localhost”和“NodePort: <unset>  30088/TCP” 这两项
foreseedeMacBook-Pro:~ foresee$ kubectl describe service nginx-sev-nodeport
Name:                     nginx-sev-nodeport
Namespace:                default
Labels:                   name=nginx-sev-nodeport
Annotations:              <none>
Selector:                 app=nginx
Type:                     NodePort
IP:                       10.102.56.80
LoadBalancer Ingress:     localhost
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30088/TCP
Endpoints:                10.1.0.53:80,10.1.0.54:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

//在busybox pod上测试这个nodeport类型的service访问，发现同样可以正常访问到。10.102.56.80:80是这个service创建时默认创建的<ClusterIp>:<Port>
/ # wget -qO - 10.102.56.80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</body>
</html>
/ #

//跟clusterip区别在于通过nodeport类型的service，我们可以通过<NodeIP>:<NodePort>来访问nginx服务。这意味着外部服务可以通过node的ip访问pod提供的服务。
foreseedeMacBook-Pro:~ foresee$ curl localhost:30088
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</body>
</html>
```
4. 通过spec.type:LoadBalancer类型的service来暴露nginx服务
```
// 创建ng-sev-lb.yaml文件
apiVersion: v1
kind: Service
metadata:
  name: nginx-sev-lb
  labels:
   name: nginx-sev-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
     
//使用如下命令部署
kubectl create -f ng-sev-lb.yaml

// 查看部署结果    
foreseedeMacBook-Pro:~ foresee$ kubectl get service
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-sev-lb         LoadBalancer   10.109.86.84   localhost     80:31439/TCP   3s

//LoadBalancer的对象描述跟nodeport比较相似，可以通过“kubectl get service nginx-sev-lb -o yaml”看的更加清晰
foreseedeMacBook-Pro:~ foresee$ kubectl describe service nginx-sev-lb
Name:                     nginx-sev-lb
Namespace:                default
Labels:                   name=nginx-sev-lb
Annotations:              kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"name":"nginx-sev-lb"},"name":"nginx-sev-lb","namespace":"default"},"spec":{...
Selector:                 app=nginx
Type:                     LoadBalancer
IP:                       10.111.144.62
LoadBalancer Ingress:     localhost
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31952/TCP
Endpoints:                10.1.0.53:80,10.1.0.54:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

// 由于是本地测试，用curl localhost 替代负载均衡实例测试。
foreseedeMacBook-Pro:~ foresee$ curl localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</body>
</html>
```

### 总结
总结一下，kind:Service的spec.type取值和行为有四种：
- ClusterIP：通过集群的内部 IP暴露服务。选择该值，只能在集群内部访问该Service，这也是默认的 ServiceType。
- NodePort：通过Node的IP 和静态端口（NodePort）暴露服务。创建NodePort类型的service时，会自动创建CluterIp。NodePort会路由到 ClusterIP。通过 <NodeIP>:<NodePort>，可以从集群外部访问到该Service。
- LoadBalancer：使用云提供商的负载局衡器，可以向外部暴露服务。创建LoadBalancer类型的Service时，会自动创建NodePort和CluterIp。外部的负载均衡器会路由到 NodePort和ClusterIP。
- ExternalName：通过返回 CNAME 和它的值，可以将服务映射到 externalName 字段的内容（例如， foo.bar.example.com）。 没有任何类型代理被创建，这只有 Kubernetes 1.7 或更高版本的 kube-dns 才支持。

*注：在 Kubernetes v1.0 版本，Service 是 “4层”（TCP/UDP over IP）概念。 在 Kubernetes v1.1 版本，新增了 Ingress API（beta 版），用来表示 “7层”（HTTP）服务。关于Ingress，请看下一篇总结。*

### 参考资料

*1. http://docs.kubernetes.org.cn/703.html*

*2. https://kubernetes.feisky.xyz/he-xin-yuan-li/index-2/service*



