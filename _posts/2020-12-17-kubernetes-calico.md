# kubeadm部署calico网络插件部署
如果已经安装flannel请先卸载掉，并确保pod之间无法通信

```shell
下载yml
https://docs.projectcalico.org/manifests/calico.yaml
```
修改配置，首先找到CALICO_IPV4POOL_IPIP修改为Never使用BGP模式

```yaml
              #CALICO_IPV4POOL_IPIP:是否启用IPIP模式。启用IPIP模式
            - name: CALICO_IPV4POOL_IPIP
              value: "Never"
              #获取Node IP地址的方式， 默认使用第1个网络接口的IP地址，对于安装了多块网卡的Node，可以 使用正则表达式选择正确的网卡
            - name: IP_AUTODETECTION_METHOD
              value: "interface=ens.*"
```
配置好后执行执行就可以了, 我这里并没有配置etcd的ip以及证书位置，但是确实在etcd中能查到calico的信息
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201217161339426.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

```shell
kubectl apply -f calico.yaml
```

calico-node:Calico服务程序，用于设置Pod的网络资源，保证 Pod的网络与各Node互联互通。它还需要以hostNetwork模式运行，直接 使用宿主机网络。

calico-kube-controllers容器，用于对接Kubernetes集群中为Pod 设置的Network Policy,用户在Kubernetes集群中设置了Pod的Network Policy之后，calico- kube-controllers就会自动通知各Node上的calico-node服务，在宿主机上 设置相应的iptables规则，完成Pod间网络访问策略的设置。

安装完成之后会发现多了几个虚拟网卡
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201217162853961.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
然后看下路由规则
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020121716304670.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
可以看到,如果pod在其他节点的网段上，会通过ens33网卡发出，本节点的都是走的calico虚拟网卡