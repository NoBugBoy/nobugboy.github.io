# k8s基于kubeadm安装（虚拟机环境），顺带解决一些虚拟机的bug
## 0 如果你也是虚拟机搭建，重启挂起等操作将会遇到两个恶心的问题（解决每次重启都会导致k8s环境BUG）

1. 如果你挂起了虚拟机或重启或重启网络可能会导致两个问题
2. 第一个问题  flannel网卡消失，无法通过 master的ip调用 masterip:nodeport
3. 第二个问题 pod内部无法解析servicename

> 第一个问题解决办法
> kubectl get daemonsets.apps -n kube-system  | grep 'flannel'
> 删除所有flannel daemonset
> 重新执行flannel.yaml
> 如果依然ping不通
> 执行以下命令后重试
> setenforce 0
> iptables --flush
> iptables -tnat --flush
> service docker restart
> iptables -P FORWARD ACCEPT

在不行就reboot试一下可能就好了


> 第二个问题解决办法
> kubectl get svc -n kube-system  | grep 'dns'
> 记录CLUSTER-IP
> 删除kube-dns （如果没有记录的话，可以去pod内查看cat /etc/resolv.conf）
> 重新执行下面的配置即可

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.1.0.10     #修改为我们设定的cluster的IP,其它默认。
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```

## 1虚拟机安装（已经安装或不想看着内容直接目录跳到k8s安装）
该文章基于mac系统 + vmware虚拟机（由于需要多台机且配置要求不低，所以决定省钱用虚拟环境）
### 1.首先下载centos的镜像，我这里选择的centos7版本
[阿里云centos镜像下载地址](http://mirrors.aliyun.com/centos/7/isos/x86_64/)
### 2.安装vmware，这个去官网下载就好了免费30天，30天之后仁者见仁智者见智
### 3.Windows的用户可以去找win的配置教程，mac的用户可以继续看

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200806100244346.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

启动vmware之后，点击右上角小图标，创建一个虚拟机，选择从镜像安装
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200806100324657.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
### 4.配置master
1. 这台核心机的配置稍微高一些，根据自己电脑的配置尽量高（最低2g），否则会有问题（默认为1g，在安装cni网络的时候发现一直启动不开，调高内存后解决）
2. 网络的话使用nat![在这里插入图片描述](https://img-blog.csdnimg.cn/20200806102407856.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
这个时候应该是ping不通外网的，现在配置网络,在mac上打开终端（不是虚拟机哦）
```shell
cd /Library/Preferences/VMware\ Fusion/vmnet8
cat nat.conf
```
找到ip和掩码，记住一会会配置
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200806102853646.png)
```shell
cat dhcpd.conf 
```
静态ip的取值范围
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200806102955494.png)
然后进入虚拟机，开始配置网络
```shell
cd /etc/sysconfig/network-scripts
ls
```
配置第一个网卡，名字可能各不相同，我的是ifcfg-ens33
```shell
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
#改成静态ip，方便使用
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
#一会会复制一台centos虚拟机省着重新创建，需要修改uuid改成不一样的即可
UUID=b2423a95-b774-4fb8-886a-9740bae35ea6
DEVICE=ens33
ONBOOT=yes
#静态ip填写dhcpd.conf 打印的范围内即可
IPADDR=192.168.121.130
#填写nat.conf中的ip
GATEWAY=192.168.121.2
#填写nat.conf中的掩码
NETMASK=255.255.255.0
#首先dns
DNS1=8.8.8.8
```
```shell
service network restart
ping www.baidu.com
```

### 5. 复制一个node节点
配置可以放低，node不会跑一大堆k8s必要组件，打开面板右键选择在finder打开
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200806104223343.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
直接将文件复制一份，改一个名字,然后打开这个复制后的文件运行，如果提示打不开磁盘“xxxxxxx.vmdk”或它所依赖的某个快照磁盘,就进入这个目录然后删除所有.lck的文件即可，然后还是修改网络，换一个静态ip 和 uuid就可以了(**配置yum源之类的就不在这说了**)，虚拟机安装到此结束

## 2.K8S安装
首先在两台机器上分别安装docker
```shell
#依赖
yum install -y epel-release
yum install -y conntrack ntpdate ntp ipvsadm ipset jq iptables curl sysstat 
libseccomp wget
#docker 安装方式1
wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum -y install docker-ce
systemctl enable docker
systemctl start docker
#docker 安装方式2
 yum install -y yum-utils
 yum-config-manager \
       --add-repo \
       https://download.docker.com/linux/centos/docker-ce.repo
 yum install docker-ce docker-ce-cli containerd.io
 sercice docker start
```
配置国内镜像（如果找不到文件夹，先启动docker）
```shell
vi /etc/docker/daemon.json
```
添加如下内容,然后重启docker
```json
{
     "registry-mirrors": ["https://kieaqrx3.mirror.aliyuncs.com"]
   }
   systemctl restart docker 或 service docker restart
```
关闭防火墙，swap，selinux
```shell
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

修改主机（node）名称和时间
```shell
#都需要执行把master node 都添加上
hostnamectl set-hostname yourhostname
#同步时间
ntpdate time1.aliyun.com
modprobe ip_vs_rr
modprobe br_netfilter
```
添加hosts
```shell
vi /etc/hosts
# 静态ip（内网ip） 节点名称
192.168.121.131 node1
192.168.121.130 master
# 为了加速获取flannel网络否则会有获取不到的问题
151.101.76.133 raw.githubusercontent.com
```
将桥接的IPv4流量传递到iptables的链
```shell
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
添加好建议重启一下生效
添加阿里云的k8s yum源
```shell
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
安装kubeadm kubelet kubectl
```shell
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet
```
初始化安装master节点，这里会拉镜像速度会比较慢，多等一会
```
kubeadm init \
  --apiserver-advertise-address=静态ip \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.18.0 \
  --service-cidr=10.1.0.0/16 \
  --pod-network-cidr=10.244.0.0/16
```
成功之后会打印node join master的命令
然后配置kubectl命令
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
然后就ok了，接下来在node1的机器上也做如上操作，只是不需要执行初始化操作
然后复制刚刚初始化终端上的kubeadm join 命令执行（如果忘记了可以用kubeadm token create --print-join-command从新生成一条）

在master上执行
```shell
kubectl get nodes
```
这个时候列表应该有两个节点，一个master 一个 node1 并且节点状态为 notready

接下里就要安装cni网络，我们选择flannel网络，这里不需要再手动安装etcd了，kebeadm已经给安装好了

```shell
# 先下载下来，如果有国内的镜像地址需要替换，如果没有打开看一下镜像版本，然后找一个可以翻墙的机器下载再导
wget  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
# 下面这个可能有问题
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
```
如果是多网卡可能需要指定网卡，否则pod ip不在同一个网段无法相互访问
```shell
#修改flannel的daemonset
kubectl edit daemonset kube-flannel-ds-amd64 -n kube-system
```
找到这个位置
```yaml
spec:
      containers:
      - args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=ens33
```
加入 - --iface=网卡名称

等待安装完成（长时间padding需要看日志）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200806112843939.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)


在一次查看kubectl get nodes
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200812105203341.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)
为了使内部pod可以ping通servicename需要打开ipvs模式的dns
```shell
#编译
kubectl edit cm kube-proxy -n kube-system
#修改
找到mode："" 修改为 ipvs即可
#重启
kubectl  get pod -n kube-system | grep kube-proxy | awk '{print $1}' | xargs kubectl delete pod -n kube-system
```


为了确保成功，请尝试如下几种操作
1. 创建一个nginx或随意一个pod + service（nodeport模式），并通过 master的nodeip:nodeport访问和nodeip:nodeport访问，都能访问成功没问题否则肯定哪里出了问题
2. 使用kubectl exec 进入pod内部 ping servicename  ，如果能ping出clusterip并且得到回应则完成![](https://img-blog.csdnimg.cn/20200812105743287.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)



安装过程中如果出现问题多使用kubectl logs 查看日志查看具体原因
kubectl get pod -n kube-system

journalctl -f -u kubelet.service


 [虚拟机部分参考文章出处](https://www.jianshu.com/p/b42ed273ef6f)