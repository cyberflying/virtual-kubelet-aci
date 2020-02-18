# Virtual-Kubelet-ACI
**Kubernetes Virtual Kubelet with ACI(Azure Container Instances), AKS Virtual Nodes with ACI** <!-- omit in toc -->
---
- [Virtual-Kubelet-ACI](#virtual-kubelet-aci)
  - [什么是Virtual Kubelet？](#%e4%bb%80%e4%b9%88%e6%98%afvirtual-kubelet)
  - [Virtual Kubelet with ACI](#virtual-kubelet-with-aci)
  - [AKS的Virtual Nodes](#aks%e7%9a%84virtual-nodes)
  - [在Virtual Nodes--ACI上部署应用](#%e5%9c%a8virtual-nodes--aci%e4%b8%8a%e9%83%a8%e7%bd%b2%e5%ba%94%e7%94%a8)

## 什么是Virtual Kubelet？
引用自[官网](https://virtual-kubelet.io/)的定义：
> Virtual Kubelet is an open-source Kubernetes kubelet implementation that masquerades as a kubelet.
> 
> This allows Kubernetes nodes to be backed by Virtual Kubelet providers such as serverless cloud container platforms.

再放张[官方Repo](https://github.com/virtual-kubelet/virtual-kubelet)的图，就方便理解了：
![](https://github.com/virtual-kubelet/virtual-kubelet/raw/master/website/static/img/diagram.svg?sanitize=true)


## Virtual Kubelet with ACI
在强大、智能、易用……等等的Azure上，当然也会有Virtual Kubelet的Provider提供了，而且还不止一个，有ACI(Azure Container Instances)， Azure Batch， Azure IoT Edge等。本文主要介绍ACI的部分。

ACI作为Virtual Kubelet的一个Provider，可以给您的K8S cluster提供的好处有哪些呢？
* **快速扩展**
* **快速扩展**
* **快速扩展**

最大的便利性就是可以非常快速的进行扩展，相比于在K8S cluster去scale out node来讲，ACI是秒级部署完成，node的vm至少是分钟级的启动时间。

当前疫情非常时期，很多线上服务，像在线教育、远程办公等场景，使用时的波峰波谷比较明显，常常因访问量突然大增急需对基础架构进行扩容。如果您的应用是部署在K8S cluster上面，需要扩展节点时，需要准备物理机或者虚拟机。**如果时间紧急，则将现有的workload直接扩展到ACI上运行，则可以快速进行扩展；当波峰访问之后，也可以很方便、快速的收回ACI的资源，节省成本，真正体现云端资源弹性伸缩的便利性。**

而从成本方面考虑，本文也拿一个具体的实例进行估算。ACI和VM的价格比较，每月费用的估算：
ACI|VM|
:---:|:---:|
82.944$/月|83.22$/月|
[价格连接](https://azure.microsoft.com/zh-cn/pricing/details/container-instances/)|[价格连接](https://azure.microsoft.com/zh-cn/pricing/details/virtual-machines/linux/)|
![](/img/aci_price.png)|![](/img/DS2v2_price.png)

当然，Azure上VM的价格如果采用保留实例（类似包年）的形式会更便宜，但ACI的好处是可以快速的帮你实现扩容，而且ACI的使用时间更灵活。

## AKS的Virtual Nodes
K8S cluster可以方便的使用ACI，可以参考：https://github.com/virtual-kubelet/azure-aci/

而作为Azure上托管的K8S，即AKS是如何使用ACI的呢？AKS的virtual nodes即是使用ACI作为virtual kubelet的。具体创建virtual nodes的方式可以参考官网文档：

https://docs.microsoft.com/azure/aks/virtual-nodes-cli

https://docs.microsoft.com/azure/aks/virtual-nodes-portal

这儿不再赘述了。启用virtual nodes后的状态：
```bash
kevin@kevin-laptop3:~$ kubectl get node -o wide
NAME                                STATUS   ROLES   AGE   VERSION                       INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-nodepool1-10604634-vmss000000   Ready    agent   6d    v1.15.7                       10.100.0.4    <none>        Ubuntu 16.04.6 LTS   4.15.0-1066-azure   docker://3.0.8
virtual-node-aci-linux              Ready    agent   6d    v1.14.3-vk-azure-aci-v1.2.1   10.100.0.11   <none>        <unknown>            <unknown>           <unknown>
kevin@kevin-laptop3:~$
kevin@kevin-laptop3:~$
kevin@kevin-laptop3:~$ kubectl get pod -o wide
NAME                              READY   STATUS    RESTARTS   AGE     IP           NODE                     NOMINATED NODE   READINESS GATES
aci-helloworld-6d4c6bf75f-5scmf   1/1     Running   0          5d23h   10.200.0.4   virtual-node-aci-linux   <none>           <none>
kevin@kevin-laptop3:~$
```

## 在Virtual Nodes--ACI上部署应用

我们以Nginx为例，测试一下部署在ACI上的效果。使用下面的virtual-node-nginx.yaml文件进行部署：
```Yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    run: nginx
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - name: nginx
    port: 8080
    targetPort: 80
    protocol: TCP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
      nodeSelector:
        kubernetes.io/role: agent
        beta.kubernetes.io/os: linux
        type: virtual-kubelet
      tolerations:
      - key: virtual-kubelet.io/provider
        operator: Exists
      - key: azure.com/aci
        effect: NoSchedule
```
可以看到，运行命令40秒之后，3个nginx的pod即在ACI开始运行了：
```bash
kevin@kevin-laptop3:~$ kubectl apply -f virtual-node-nginx.yaml
service/nginx created
deployment.apps/nginx created
kevin@kevin-laptop3:~$
kevin@kevin-laptop3:~$
kevin@kevin-laptop3:~$ kubectl get pod -o wide
NAME                              READY   STATUS     RESTARTS   AGE    IP           NODE                                NOMINATED NODE   READINESS GATES
aci-helloworld-6d4c6bf75f-5scmf   1/1     Running    0          6d1h   10.200.0.4   virtual-node-aci-linux              <none>           <none>
nginx-846d559fb7-dft9r            0/1     Creating   0          7s     <none>       virtual-node-aci-linux              <none>           <none>
nginx-846d559fb7-rjhvv            0/1     Pending    0          7s     <none>       virtual-node-aci-linux              <none>           <none>
nginx-846d559fb7-rmzss            0/1     Creating   0          7s     <none>       virtual-node-aci-linux              <none>           <none>
kevin@kevin-laptop3:~$
kevin@kevin-laptop3:~$ kubectl get pod -o wide
NAME                              READY   STATUS    RESTARTS   AGE    IP           NODE                                NOMINATED NODE   READINESS GATES
aci-helloworld-6d4c6bf75f-5scmf   1/1     Running   0          6d1h   10.200.0.4   virtual-node-aci-linux              <none>           <none>
nginx-846d559fb7-dft9r            1/1     Running   0          41s    10.200.0.6   virtual-node-aci-linux              <none>           <none>
nginx-846d559fb7-rjhvv            1/1     Running   0          41s    10.200.0.5   virtual-node-aci-linux              <none>           <none>
nginx-846d559fb7-rmzss            1/1     Running   0          41s    10.200.0.7   virtual-node-aci-linux              <none>           <none>
kevin@kevin-laptop3:~$
```
查看LB的IP:
```bash
kevin@kevin-laptop3:~$ kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)          AGE
kubernetes   ClusterIP      10.0.0.1       <none>         443/TCP          6d5h
nginx        LoadBalancer   10.0.153.110   52.167.68.95   8080:30037/TCP   3h22m
kevin@kevin-laptop3:~$
```
在Azure资源组中可以看到多出来的ACI：
![](/img/nginx0.png)

访问测试效果：
![](/img/nginx2.png)

我们再进一步修改这个3个pod中nginx:
```bash
kevin@kevin-laptop3:~$ kubectl exec nginx-846d559fb7-dft9r -c nginx -it -- /bin/bash
root@wk-caas-04d1bf6b7b7a41fe8e05ab9330980cfa-3c21f7c2dae9a83a33a62f:/# echo "Hello World from host 1" $HOSTNAME "!" > /usr/share/nginx/html/index.html
root@wk-caas-04d1bf6b7b7a41fe8e05ab9330980cfa-3c21f7c2dae9a83a33a62f:/#
root@wk-caas-04d1bf6b7b7a41fe8e05ab9330980cfa-3c21f7c2dae9a83a33a62f:/# ls -l  /usr/share/nginx/html/index.html
-rw-r--r-- 1 root root 90 Feb 18 13:13 /usr/share/nginx/html/index.html
root@wk-caas-04d1bf6b7b7a41fe8e05ab9330980cfa-3c21f7c2dae9a83a33a62f:/# kevin@kevin-laptop3:~$
kevin@kevin-laptop3:~$
kevin@kevin-laptop3:~$
kevin@kevin-laptop3:~$ kubectl exec nginx-846d559fb7-rjhvv -c nginx -it -- /bin/bash
root@wk-caas-3cb24cf18a0a408a9cf154a5ec55c818-949e2ebb19815ee19374f3:/# ls -l /usr/share/nginx/html/index.html
-rw-r--r-- 1 root root 612 Jan 21 13:36 /usr/share/nginx/html/index.html
root@wk-caas-3cb24cf18a0a408a9cf154a5ec55c818-949e2ebb19815ee19374f3:/#
root@wk-caas-3cb24cf18a0a408a9cf154a5ec55c818-949e2ebb19815ee19374f3:/# echo "Hello World from host 2 " $HOSTNAME "!" > /usr/share/nginx/html/index.html
root@wk-caas-3cb24cf18a0a408a9cf154a5ec55c818-949e2ebb19815ee19374f3:/# ls -l /usr/share/nginx/html/index.html
-rw-r--r-- 1 root root 91 Feb 18 13:15 /usr/share/nginx/html/index.html
root@wk-caas-3cb24cf18a0a408a9cf154a5ec55c818-949e2ebb19815ee19374f3:/#
root@wk-caas-3cb24cf18a0a408a9cf154a5ec55c818-949e2ebb19815ee19374f3:/# exit
exit
kevin@kevin-laptop3:~$
kevin@kevin-laptop3:~$ kubectl exec nginx-846d559fb7-rmzss -c nginx -it -- /bin/bash
root@wk-caas-697c2c9f8edf46fd9cf34f0e747e26e2-3cc90bbd5dcaa40b9da5ea:/# echo "Hello World from host 3 " $HOSTNAME "!" > /usr/share/nginx/html/index.html
root@wk-caas-697c2c9f8edf46fd9cf34f0e747e26e2-3cc90bbd5dcaa40b9da5ea:/# exit
exit
kevin@kevin-laptop3:~$
```
在浏览器中访问这个LB，多刷新几次，可以看到不同的nginx页面：
![](/img/nginx1.png)
![](/img/nginx3.png)