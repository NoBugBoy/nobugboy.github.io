# Traefik2.x IngressController，流量是如何从外部流入kubernetes内部的？

> 引言

主要想了解，外部流量如何请求到内部，并且做反向代理

> 正文

### 1. k8s外部流量如何请求到内部？
1. 第一种方式比较容易想到，只要将service绑定好pod，并且将service的nodeport暴露出来，提供入口和负载均衡，流量就可以从外部请求到对应的服务，此时就可以按照以前的经验，在最外层部署nginx，由nginx代理这些暴露的nodeport，这样就可以达到反向代理的效果，但是这种方式是静态的，也就是说如果有新的服务需要修改配置重启生效

2. 第二种方式主要利用k8s提供的IngressController Kind，有很多种实现，这里就说一下Traefik IngressController，他本身提供了反向代理等功能，路由方式，各种插件，WEB页面，以及权限管理等，安装好之后，以后有变更只需要edit ingress即可，发布新的路由也只需发布一个新的ingress就可以动态生效，ingress是在k8s内部，所以他可以不代理nodeport，直接代理clusterIp对应的containerPort即可，也就是说可以不暴露服务到外网，只有ingressController暴露在了外网
### 2.如何部署并暴露IngressController？
这里依然拿Traefik2举例，他有很多端口，比如流量入口 web：80，管理页面端口 admin：8080等，我们只需要将80端口改为hostport占用宿主机的80端口，并以daemonset方式部署，每个节点都部署一个，配置相同的域名和子域名，这样流量就可以请求到任意一个节点都是代理端口80，并自动转发，最后将所有节点的外网ip与域名绑定好后，自动就有了DNS轮训功能，保证了多节点并发流量均衡。
### 3.如何部署Traefik2 IngressController？
1.  网上有很多教程可以参考一些基于yml部署的博客去实现（好不好用看运气）
2.  基于官网的Helm3部署，需要安装helm并且对其尽可能的熟悉，参考官方地址：[基于Helm安装教程](https://doc.traefik.io/traefik/getting-started/install-traefik/)
3.  基于官网的crd安装（有些小问题）

安装traefik的自定义kind资源和rbac，然后部署dp即可，[yum安装教程](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/)

这里有一些问题，部署yml的时候官网是dp的形式部署，可改为daemonset，然后需要加一个东西，否则ingressroute不生效
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201119180928468.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)
然后将web：80 流量入口改为hostport即可
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201119181057593.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)
这样写ingress或ingressRoute就可以不用端口号直接请求到配置perfix前缀路径了，最后再写一个ingressRoute去代理下traefik的admin端口：8080 就可以看到web页面了(不代理用nodeport也可以)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201119181305458.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)
最后需要注意一下的是ingressRoute的配置路径
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201119181430467.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)
比如我配置的是/nginx 会404，如果你用过nginx看一下我这篇文章你就能理解什么意思[路径问题](https://blog.csdn.net/Day_Day_No_Bug/article/details/103571592)，大概就是请求 /nginx 时确实会转发给nginx-svc，但是转发过去的路径依然会带上/nginx 但是nginx.conf并没有任何服务代理到/nginx 所以一定是404 ，只需要代理到 / 即可