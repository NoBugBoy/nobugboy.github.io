# k8s部署Dashboard（版本通用教程）
直接找对应的版本，不要乱搞各种问题（别问都是眼泪）

> 配置文件地址 https://github.com/kubernetes/dashboard/releases


![在这里插入图片描述](https://img-blog.csdnimg.cn/20200812110310485.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)

往下翻执行这个命令
```shell
#如果建议去上面的连接查看对应版本 不要无脑复制粘贴
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml
```
然后将clusterip改为nodeport
```shell
kubectl  edit svc kubernetes-dashboard -n kubernetes-dashboard 
```
改成这个样子
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200807171342715.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
ok，然后创建账号
```shell
kubectl create serviceaccount dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```
用火狐浏览器打开，其他浏览器不行，输入https://nodeip:nodeport,并且信任该链接

输入打印在终端的token即可,新版本默认为中文
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200812110528891.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200812110603132.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)

安装完成