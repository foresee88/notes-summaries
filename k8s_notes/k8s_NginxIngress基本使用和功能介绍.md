
## Kubernetes Nginx Ingress基本使用和功能介绍

### 简介
Ingress是利用Nginx、Haproxy等负载均衡器暴露集群内服务的工具。
它可以提供七层负载均衡能力。在部署了相应的Ingress Controller 之后，你就可以通过创建Ingress对象把集群内的service服务暴露到集群外。

### 演示
下面我们在[Docker Desktop](https://www.docker.com/products/docker-desktop)环境下演示下nginx-ingress-controller部署和使用。

1. 首先要部署nginx-ingress-controller
```
//部署nginx-ingress-controller, 详细指导看 https://kubernetes.github.io/ingress-nginx/deploy/ 
//nginx-ingress-controller相关代码和yaml文件在 https://github.com/kubernetes/ingress-nginx
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml

//查看创建结果,流量转发发生在nginx-ingress-controller-86449c74bb-sfs8r这个pod上
foreseedeMacBook-Pro:~ foresee$ kubectl get deployments -n ingress-nginx
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
nginx-ingress-controller   1/1     1            1           19m
foreseedeMacBook-Pro:~ foresee$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-86449c74bb-sfs8r   1/1     Running   0          19m

//可以登陆到pod中查看nginx.conf,此时由于没有创建Ingress对象，因此nginx.conf中没有响应的转发规则
kubectl exec -n ingress-nginx -it nginx-ingress-controller-86449c74bb-sfs8r -- /bin/sh

//nginx-ingress-controller创建出来之后，需要在该namespace下创建service，把流量引入controller。
// ing-sev-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
    nodePort: 30080
  - name: https
    port: 443
    targetPort: 443
    protocol: TCP
    nodePort: 30443
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

//创建NodePort类型的service，并验证访问，可以看到符合nginx-ingress-controller-86449c74bb-sfs8r这个pod中nginx.conf定义的规则
foreseedeMacBook-Pro:~ foresee$ kubectl apply -f ing-sev-nodeport
service/ingress-nginx created
foreseedeMacBook-Pro:~ foresee$ curl http://localhost:30080
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>openresty/1.15.8.1</center>
</body>
</html>

//如果没创建该service，nginx-ingress-controller-86449c74bb-sfs8r会报“err services "ingress-nginx" not found”
foreseedeMacBook-Pro:~ foresee$ kubectl attach nginx-ingress-controller-86449c74bb-sfs8r -n ingress-nginx
Defaulting container name to nginx-ingress-controller.
Use 'kubectl describe pod/ -n ingress-nginx' to see all of the containers in this pod.
If you don't see a command prompt, try pressing enter.
W0809 12:14:43.681731 7 queue.go:130] requeuing &ObjectMeta{Name:sync status,GenerateName:,Namespace:,SelfLink:,UID:,ResourceVersion:,Generation:0,CreationTimestamp:0001-01-01 00:00:00 +0000 UTC,DeletionTimestamp:<nil>,DeletionGracePeriodSeconds:nil,Labels:map[string]string{},Annotations:map[string]string{},OwnerReferences:[],Finalizers:[],ClusterName:,Initializers:nil,ManagedFields:[],}, err services "ingress-nginx" not found


```
2. 部署完nginx-ingress-controller后，我们就可以通过创建ingress对象方式实现对nginx的配置，从而把流量引入到我们的应用集群中。

```
//以deployment部署一个两副本的nginx服务（模拟一个普通的应用服务）（参考上一篇文章）
//ng-ing.yaml
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

//部署一个ClusterIp类型的service        
apiVersion: v1
kind: Service
metadata:
  name: nginx-sev-clusterip
  labels:
   name: nginx-sev-clusterip
spec:
apiVersion: apps/v1beta1
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx

//部署一个Ingress对象，这个对象的部署会在nginx-ingress-controller-86449c74bb-sfs8r这个pod中nginx.conf添加转发规则
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ng-ing
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: myingress-test.info
    http:
      paths:
      - path:
        backend:
          serviceName: nginx-sev-clusterip
          servicePort: 80  
          
//可以登陆到pod中查看nginx.conf,此时由于没有创建Ingress对象，因此nginx.conf中没有响应的转发规则
kubectl exec -n ingress-nginx -it nginx-ingress-controller-86449c74bb-sfs8r -- /bin/sh
$ ls
geoip  lua  mime.types	modsecurity  modules  nginx.conf  opentracing.json  owasp-modsecurity-crs  template
$ cat nginx.conf

# Configuration checksum: 6106042106108846766
# setup custom paths that do not require root access
pid /tmp/nginx.pid;
daemon off;
worker_processes 2;
worker_rlimit_nofile 523264;
worker_shutdown_timeout 10s ;
events {
	multi_accept        on;
	worker_connections  16384;
	use                 epoll;
}
http {
    ...
	## start server myingress-test.info
	server {
		server_name myingress-test.info ;

		listen 80;

		set $proxy_upstream_name "-";
		set $pass_access_scheme $scheme;
		set $pass_server_port $server_port;
		set $best_http_host $http_host;
		set $pass_port $pass_server_port;
		
		location / {
			set $namespace      "default";
			set $ingress_name   "ng-ing";
			set $service_name   "nginx-sev-clusterip";
			set $service_port   "80";
			set $location_path  "/";
            ...
		}
	}
	## end server myingress-test.info
	...
}
        
```
3. 通过ingress对象定义的访问地址进行验证

```
# 验证方法：首先把流量引入到nginx-ingress-controller，有lb和nodeport两种方案 
# 1.nodeport类型service（这个service之前我们已经创建了，不需要重新创建）
// ing-sev-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
    nodePort: 30080
  - name: https
    port: 443
    targetPort: 443
    protocol: TCP
    nodePort: 30443
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

# 进行测试
foreseedeMacBook-Pro:~ foresee$ curl http://myingress-test.info:30080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</body>
</html>

# 2.LoadBalancer类型service
// ing-sev-lb.yaml
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  externalTrafficPolicy: Local
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: https
      port: 443
      targetPort: 443

# 进行测试
foreseedeMacBook-Pro:~ foresee$ curl http://myingress-test.info
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</body>
</html>
    
```

### 总结

nginx-ingress-controller是通过在集群内配置一个nginx pod把外部流量分发到内部各个clusterip类型的service上。但是传统nginx配置管理的方式，每增加一个service（一个新的应用服务pod集群）的时候，我们需要手工修改nginx配置，这就比较麻烦了。nginx-ingress-controller主要就是解决nginx手工配置的问题。nginx-ingress-controller通过与 Kubernetes API 交互，动态的去感知集群中 Ingress 规则变化，然后读取他，按照他自己模板生成一段 Nginx 配置，再写到 Nginx Pod 里，最后 reload 一下，完成配置变更和生效。

简单的理解就是原来需要手工改 Nginx 配置，改各种域名对应哪个 Service，现在把这个动作抽象出来，变成一个 Ingress 对象，你可以用 yaml 创建，每次不要去改 Nginx 了，直接改 yaml 然后创建/更新就行了。

工作流程如下图：

![alt-txt](https://share.nos-eastchina1.126.net/ingress.png)

### 参考资料

*1. https://kubernetes.github.io/ingress-nginx/deploy/*

*2. https://github.com/kubernetes/ingress-nginx*

*3. https://www.cnblogs.com/linuxk/p/9706720.html*

*4. http://blog.itpub.net/28916011/viewspace-2214747/*