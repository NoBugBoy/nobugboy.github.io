# kubeadm1.19高可用kubernetes，高可用Etcd部署

> 预准备

1. 3台2核2G服务器（虚拟机）2G的只能保证搭建成功，测试的话内存可能不足
2. 前置的配置，源，docker安装等请参考之前的博客 -> [安装教程](https://blog.csdn.net/Day_Day_No_Bug/article/details/107832508)
3. 配置好后不要执行kubeadm init 就可回到这篇文章继续看

> 开始安装

### 1.Etcd集群安装

etcd是一个高可用的分布式键值(key-value)数据库，kubernetes将服务和数据信息保存在etcd中，如果etcd挂掉集群不可用，数据如果丢失集群将变为初始状态，所以etcd的高可用必须要保证，这里将使用外置etcd集群，不在使用kubeadm初始化时自动生成的容器化的单机etcd

---
首先选择一个服务器，用来生成etcd所需要的加密证书,这里使用cfssl方式，可以以json的格式生成，不需要写一长串命令
```shell
# 安装并配置环境变量
yum install wget -y
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
chmod +x cfssljson_linux-amd64
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
chmod +x cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
export PATH=/usr/local/bin:$PATH
```
生成证书的json文件
```json
mkdir /root/ssl
cd /root/ssl
cat >  ca-config.json <<EOF
{
"signing": {
"default": {
  "expiry": "8760h"
},
"profiles": {
  "kubernetes-Soulmate": {
    "usages": [
        "signing",
        "key encipherment",
        "server auth",
        "client auth"
    ],
    "expiry": "8760h"
  }
}
}
}
EOF
```
```json
cat >  ca-csr.json <<EOF
{
"CN": "kubernetes-Soulmate",
"key": {
"algo": "rsa",
"size": 2048
},
"names": [
{
  "C": "CN",
  "ST": "shanghai",
  "L": "shanghai",
  "O": "k8s",
  "OU": "System"
}
]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```
```json
#hosts需要填所有的节点ip
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "172.16.3.130",
    "172.16.3.131",
    "172.16.3.132"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "shanghai",
      "L": "shanghai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
#签名
cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes-Soulmate etcd-csr.json | cfssljson -bare etcd
```
接下来我们需要将证书分发到每一个需要安装etcd的服务上,由于我是在130服务器上生成的证书所以只需要将证书发送给另外两台服务器即可
```shell
mkdir -p /etc/etcd/ssl
cp /root/ssl/etcd.pem /root/ssl/etcd-key.pem /root/ssl/ca.pem /etc/etcd/ssl/
scp -r /etc/etcd/ 172.16.3.131:/etc/
scp -r /etc/etcd/ 172.16.3.132:/etc/
```
接下来在**每台**需要安装etcd服务的节点上安装etcd服务
```shell
yum install etcd -y   
# 创建数据保存目录，备份点
mkdir -p /var/lib/etcd
```
然后配置etcd的配置文件，将其修改为集群模式,需要注意的几个点
1. 需要修改--name
2. 需要将ip修改为对应服务器的ip
3. 如果出现request sent was ignored (cluster ID mismatch: peer[f73f6335fab3c75e]=903824bb6a071282问题，可以将--initial-cluster-state修改为existing
4. --initial-cluster  后面的名字一定要和--name对应上
5. --data-dir 要提前创建好
```shell
cat <<EOF >/usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name k8s01 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://172.16.3.130:2380 \
  --listen-peer-urls https://172.16.3.130:2380 \
  --listen-client-urls https://172.16.3.130:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://172.16.3.130:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster k8s01=https://172.16.3.130:2380,k8s02=https://172.16.3.131:2380,k8s03=https://172.16.3.132:2380 --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
每台etcd都配置完成后，启动,如果出现异常请检查配置文件
```shell
 systemctl daemon-reload
 systemctl enable etcd
 systemctl start etcd
 systemctl status etcd
```
找一台安装了etcd服务的机器来查看集群状态
```shell
#配置3.0的api 否则有些命令没有
echo "export ETCDCTL_API=3" >>/etc/profile  && source /etc/profile
#修改为自己的ip
etcdctl --endpoints=https://172.16.3.130:2379,https://172.16.3.131:2379,https://172.16.3.132:2379 --cacert=/etc/etcd/ssl/ca.pem   --cert=/etc/etcd/ssl/etcd.pem   --key=/etc/etcd/ssl/etcd-key.pem   endpoint health
```
如果如下图全部successfully就是成功了，如果有一个失败的请检查systemctl status etcd，查看错误信息，如果没有特殊情况就是配置文件出错了
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020112411051749.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)
### 2.安装keepalived
这里我模拟2台master节点，1个node节点，两个master节点通过keepalived漂移ip保活，以下配置需要在两台master节点都修改
```shell
#启动kubelet
systemctl daemon-reload
systemctl enable kubelet
```
安装keepalived
```shell
yum install -y keepalived
systemctl enable keepalived
```
```shell
cat <<EOF > /etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}
#心跳脚本
vrrp_script CheckK8sMaster {
    script "curl -k https://172.16.3.110:6443"  # 虚ip
    interval 3
    timeout 9
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33       #云服务器一般eth0，虚拟机需要自己ifconfig查看下
    virtual_router_id 61
    priority 120          
    advert_int 1
    mcast_src_ip 172.16.3.130  #写当前服务器的ip
    nopreempt
    authentication {
        auth_type PASS
        auth_pass sqP05dQgMSlzrxHj
    }
    unicast_peer {
        172.16.3.131    #另外一台masterIP，例如在131服务器上这里需要写130，也就是除了mcast_src_ip的ip
    }
    virtual_ipaddress {
        172.16.3.110/24    # VIP
    }
    track_script {
        CheckK8sMaster
    }

}
EOF
```
启动keepalived并测试下ip能否正常漂移
```
systemctl start keepalived
systemctl status keepalived
```
首先查看vip在哪台master上
```shell
#找到对应的vip
ip a
```
如下图，我的vip目前在172.16.3.131这台机器上，这时候关闭131上的keepalived看看vip能否正常漂移到另一台master也就是130上，如果可以正常漂移过去就说明生效了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201124112325112.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)
### 3.整理初始化配置文件
这个步骤由于每个版本的配置文件模板不一样所以需要动态调整,需要在自己的版本配置文件的基础上来修改，我这里的版本是kubeadm.k8s.io/v1beta2，以下有几个注意的点

1. apiServer.certSANs需要把集群所有ip添加，并且还要吧对应的hosts的名称添加进来
2. localAPIEndpoint.advertiseAddress是当前的节点ip
3. controlPlaneEndpoint是 vip的地址
4. imageRepository需要改为国内镜像源
5. etcd需要把刚才的ip都配置一遍

```shell
 #先导出一份基础配置
 kubeadm config print init-defaults >init-config.yaml
```
这个是基于1.19版本的修改为高可用后的配置文件
```yaml
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
  advertiseAddress: 172.16.3.130
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: master
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
  certSANs:
  - "node0"
  - "node1"
  - "node2"
  - "172.16.3.110"
  - "172.16.3.130"
  - "172.16.3.131"
  - "172.16.3.132"
  - "127.0.0.1"
  extraArgs: 
           etcd-cafile: /etc/etcd/ssl/ca.pem
           etcd-certfile: /etc/etcd/ssl/etcd.pem
           etcd-keyfile: /etc/etcd/ssl/etcd-key.pem
controlPlaneEndpoint: "172.16.3.130:6443"
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
   external:
        caFile: /etc/etcd/ssl/ca.pem
        certFile: /etc/etcd/ssl/etcd.pem
        keyFile: /etc/etcd/ssl/etcd-key.pem
        endpoints:
        - https://172.16.3.130:2379
        - https://172.16.3.131:2379
        - https://172.16.3.132:2379
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.19.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: "10.244.0.0/16"
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs" 
```
### 4.开始配置集群
需要注意的是，我们可以ping通vip的地址，但是有一个问题，集群的初始化必须要在当前vip所在的机器上，否则会疯狂的超时
```shell
GET https://172.16.3.110:6443/healthz?timeout=10s in 0 milliseconds
```
找到vip所在的master节点，开始初始化集群
```shell
kubeadm init --config init-config.yaml
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201124113719107.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)
安装完成后，会有两个join的命令，带有 --control-plane 是加入master集群的，执行之前需要先将证书发送过去
```shell
scp -r /etc/kubernetes/pki  172.16.3.130:/etc/kubernetes/
#然后执行对应的join命令即可
kubeadm join 172.16.3.110:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:c546b1e5c5cf3e587752bbd862db332c183607b6f9c48b6514e9197f25cdbe39 \
    --control-plane 
#加入成功后配置命令
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config    
```
然后查看下节点即可
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201124114003359.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)
再查看下各组件状态，如果是1.19版本的话默认是会连接失败
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201125131830163.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201124114023303.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70#pic_center)
如果连接失败只需将/etc/kubernetes/manifests下面的配置文件修改下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201124114212631.png#pic_center)
将这个两个配置文件上的--port=0这行删除即可，到此整个高可用集群搭建完成

如果服务器是2g的内存基本上搭建完就不剩余什么空间了.
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201125131921939.png#pic_center)