---
layout: post
title: 'Kubernetes'
date: 2022-06-01
excerpt: "使用haproxy+keepalived实现k8s高可用架构集群，包括kubedam与二进制安装"
tags: ['haproxy+keepalived','kubeadm','二进制']
comments: false
<!-- project: true -->
---

## <a href='#a'>* 基本环境配置</a> 
## * <a href='#b'>使用ipvs模块</a>  
## * <a href='#c'>配置k8s集群内核参数</a>   
## * <a href='#d'>安装containerd</a>    
## * <a href='#e'>安装k8s组件</a>     
## * <a href='#f'>高可用组件安装与配置</a>     
## * <a href='#g'>集群初始化</a>    
## * <a href='#h'>污点</a>    
## * <a href='#i'>配置calico网络</a>   
## * <a href='#j'>Metrics部署</a>     
## * <a href='#k'>Dashboard部署</a>     
## * <a href='#l'>命令自动补全</a>    


```
阿里云国内仓库：registry.aliyuncs.com/google_containers/  

Kubeadm安装与二进制安装：  
kubeadm安装的集群，证书有效期默认是一年。master节点的kube-apiserver、kube-scheduler、kube-controller-manager、etcd都是以容器运行的。可以通过kubectl get po -n kube-system查看 

启动和二进制不同的是：  
kubelet的配置文件在/etc/sysconfig/kubelet和/var/lib/kubelet/config.yaml，修改后需要重启kubelet进程  
其他组件的配置文件在/etc/kubernetes/manifests目录下，比如kube-apiserver.yaml，该yaml文件更改后，kubelet会自动刷新配置，也就是会重启pod，不能再次创建该文件  

kube-proxy的配置在kube-system命名空间下的configmap中，可以通过  
kubectl edit cm kube-proxy -n kube-system  
进行更改，更改完成后，可以通过patch重启kube-proxy  
kubectl patch daemonset kube-proxy -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" -n kube-system  

生产环境要用二进制安装  
```

### <span id='a'>基本环境配置  </span>
```
Kubeadm使用keepalived+haproxy安装高可用集群，containerd做为容器引擎  
环境：  
192.168.1.10----k8s-master01  
192.168.1.15----k8s-master02  
192.168.1.20----k8s-master03  
192.168.1.111-vip，公有云搭建的VIP是负载均衡的IP，比如阿里云的是内网SLB地址，腾讯云的是内网ELB的地址。不需要再搭建keepalived+haproxy  
Docker 20.10.x  
K8s 1.24.x  
Pod网段 172.16.0.0/12  
Service网段 192.168.0.0/16  

以下所有操作未注明单台节点的即为三台节点统一命令操作	  
vim /etc/hosts  
![image](https://user-images.githubusercontent.com/80735002/175618513-94414bfe-d989-4725-acc9-c4decb87421f.png)
单台修改主机名称  
vim /etc/hostname  
hostname k8s-master{01,02,03}  
![image](https://user-images.githubusercontent.com/80735002/175618772-26975425-8816-4704-8721-1e64381af6c6.png)
![image](https://user-images.githubusercontent.com/80735002/175618783-f187cd4e-1077-43ff-b363-8e715e6087bd.png)
![image](https://user-images.githubusercontent.com/80735002/175618797-f5c48530-c473-40b3-9e00-3a0e6830f5a7.png)
下载aliyun的yum源  
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo  
安装必需依赖  
yum -y install yum-utils device-mapper-persistent-data lvm2 vim net-tools jq psmisc telnet git  
添加阿里云的docker源  
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo  
添加阿里云的k8s源  
cat <<EOF > /etc/yum.repos.d/kubernetes.repo  
[kubernetes]  
name=Kubernetes  
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/  
enabled=1  
gpgcheck=0  
repo_gpgcheck=0  
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg  
EOF  

关闭防火墙、seliuux、NetworkManager  
systemctl stop firewalld && systemctl disable firewalld  
systemctl stop NetworkManager && systemctl disable NetworkManager  
sed -i 's#SELINUX=enforcing#SELINUX=disable#g' /etc/sysconfig/selinux  
sed -i 's#SELINUX=enforcing#SELINUX=disable#g' /etc/selinux/config  

关闭swap分区  
swapoff -a && sysctl -w vm.swappiness=0  
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab  

安装ntpdate  
yum -y install ntpdate  
同步时间，修改时区为上海，并加入任务计划  
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime  
echo 'Asia/Shanghai' > /etc/timezone  
ntpdate time2.aliyun.com  

配置任务计划
crontab -e  
![image](https://user-images.githubusercontent.com/80735002/175621963-62846057-0118-49c2-9e0d-bd39a177adf8.png)
systemctl restart crond  

配置文件句柄数  
ulimit -SHn 65535  
vim /etc/security/limits.conf  
# 添加到文件末尾  
* soft nofile 65536  
* hard nofile 131072  
* soft nproc 65536  
* hard nproc 655360  
* soft memlock unlimited  
* hard memlock unlimited  

配置免密登录，生成配置文件和证书均在master01节点上操作，集群管理也只在master01上操作，阿里云或AWS上需要一台单的的kucectl服务器，密钥生成：  
ssh-keygen -t rsa  
![image](https://user-images.githubusercontent.com/80735002/175623482-05de1f4e-2b93-45d3-9133-8bb0a246864e.png)
将公钥传到master02、master03上  
for i in k8s-master02 k8s-master03;do ssh-copy-id -i .ssh/id_rsa.pub $i;done  
![image](https://user-images.githubusercontent.com/80735002/175626847-1416a0aa-085d-47f6-ad75-bf5f5c6680f1.png)

升级内核  
别问，内核4.18+会很舒服，低于的用docker以及k8s会有一些棘手的bug，多年生产经验总结  
升级系统  
yum update -y --exclude=kernel*  
下载内核  
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm  
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm  
yum localinstall -y kernel-ml*  
更改内核启动顺序  
grub2-set-default 0 && grub2-mkconfig -o /etc/grub2.cfg  
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)  
检查内核版本  
grubby --default-kernel  
这里重启之后会变成4.19······  
reboot  
![image](https://user-images.githubusercontent.com/80735002/175627383-0314f00b-a70d-49cd-b4c2-6d4ccb0ddacb.png)
```

### <span id='b'>使用ipvs模块  </span>
```
安装ipvsadm，生产环境推荐使用ipvs而不是iptables  
yum -y install ipvsadm ipset sysstat conntrack libseccomp  
加载ipvs  
内核4.19+，nf_conntrack_ipv4改为nf_conntrack  
modprobe -- ip_vs  
modprobe -- ip_vs_rr  
modprobe -- ip_vs_wrr  
modprobe -- ip_vs_sh  
modprobe -- nf_conntrack  

vim /etc/modules-load.d/ipvs.conf  
ip_vs  
ip_vs_lc  
ip_vs_wlc  
ip_vs_rr  
ip_vs_wrr  
ip_vs_lblc  
ip_vs_lblcr  
ip_vs_dh  
ip_vs_sh  
ip_vs_fo  
ip_vs_nq  
ip_vs_sed  
ip_vs_ftp  
ip_vs_sh  
nf_conntrack  
ip_tables  
ip_set  
xt_set  
ipt_set  
ipt_rpfilter  
ipt_REJECT  
ipip  

systemctl enable --now systemd-modules-load.service  
修改kube-proxy为ipvs模式  
kubectl edit cm kube-proxy -n kube-system  
![image](https://user-images.githubusercontent.com/80735002/175647675-9106c34b-bece-4359-96e0-69c1419b518c.png)
更新pod  
kubectl patch daemonset kube-proxy -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" -n kube-system  
验证  
![image](https://user-images.githubusercontent.com/80735002/175647717-e7c5c402-d0da-4af6-9123-03240f938335.png) 
```

### <span id='c'>配置k8s集群内核参数    </span>
```
cat <<EOF > /etc/sysctl.d/k8s.conf  
net.ipv4.ip_forward = 1  
net.bridge.bridge-nf-call-iptables = 1  
net.bridge.bridge-nf-call-ip6tables = 1  
fs.may_detach_mounts = 1  
net.ipv4.conf.all.route_localnet = 1  
vm.overcommit_memory=1  
vm.panic_on_oom=0  
fs.inotify.max_user_watches=89100  
fs.file-max=52706963  
fs.nr_open=52706963  
net.netfilter.nf_conntrack_max=2310720  
net.ipv4.tcp_keepalive_time = 600  
net.ipv4.tcp_keepalive_probes = 3  
net.ipv4.tcp_keepalive_intvl =15  
net.ipv4.tcp_max_tw_buckets = 36000  
net.ipv4.tcp_tw_reuse = 1  
net.ipv4.tcp_max_orphans = 327680  
net.ipv4.tcp_orphan_retries = 3  
net.ipv4.tcp_syncookies = 1  
net.ipv4.tcp_max_syn_backlog = 16384  
net.ipv4.ip_conntrack_max = 65536  
net.ipv4.tcp_max_syn_backlog = 16384  
net.ipv4.tcp_timestamps = 0  
net.core.somaxconn = 16384  
EOF  

sysctl --system  
![image](https://user-images.githubusercontent.com/80735002/175630273-701e5919-934a-4469-b8d4-0b15e27f4fde.png)
重启，保证重启后内核依然加载  
reboot  
lsmod |grep --color=auto -e ip_vs -e nf_conntrack  
![image](https://user-images.githubusercontent.com/80735002/175630867-d05db368-9a26-4851-88fb-61af676e935f.png)
```

### <span id='d'>安装containerd    </span>
```
yum -y install docker-ce-20.10.* docker-ce-cli-20.10.*  
不使用dockerfile则无需启动docker，配置并启动containerd即可  
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf  
overlay  
br_netfilter  
EOF   

加载模块  
modprobe -- overlay  
modprobe -- br_netfilter  

配置containerd所需内核  
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf  
net.bridge.bridge-nf-call-iptables  = 1  
net.ipv4.ip_forward                 = 1  
net.bridge.bridge-nf-call-ip6tables = 1  
EOF  
sysctl --system   

创建containerd的配置文件  
mkdir /etc/containerd -p  
containerd config default | tee /etc/containerd/config.toml  
vim /etc/containerd/config.toml  

cgroup改为systemd  
![image](https://user-images.githubusercontent.com/80735002/175633172-6957a45c-eca6-4af9-929a-119d03dd123f.png)
Pause镜像修改为国内阿里云地址  

启动containerd并设置开机自启  
systemctl daemon-reload && systemctl enable --now containerd  

配置crictl客户端连接的runtime位置，sock  
cat > /etc/crictl.yaml <<EOF  
runtime-endpoint: unix:///run/containerd/containerd.sock  
image-endpoint: unix:///run/containerd/containerd.sock  
timeout: 10  
debug: false  
EOF  
```

### <span id='e'>安装k8s组件  </span>
```
查看最新版本  
yum list kubeadm.x86_64 --showduplicates | sort -r  
生产一定使用1.2x.5+，比较稳定  
yum install kubeadm-1.24* kubelet-1.24* kubectl-1.24* -y  

更改runtime为containerd  
cat >/etc/sysconfig/kubelet<<EOF  
KUBELET_KUBEADM_ARGS="--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"  
EOF  

systemctl daemon-reload && systemctl enable --now kubelet  
```
  
### <span id='f'>高可用组件安装与配置  </span>
```
yum install keepalived haproxy -y  

配置haproxy  
vim /etc/haproxy/haproxy.cfg  
global  
  maxconn  2000  
  ulimit-n  16384  
  log  127.0.0.1 local0 err  
  stats timeout 30s    
defaults  
  log global  
  mode  http  
  option  httplog  
  timeout connect 5000  
  timeout client  50000  
  timeout server  50000  
  timeout http-request 15s  
  timeout http-keep-alive 15s    
frontend monitor-in  
  bind *:33305  
  mode http  
  option httplog  
  monitor-uri /monitor  
frontend k8s-master  
  bind 0.0.0.0:16443  
  bind 127.0.0.1:16443  
  mode tcp  
  option tcplog  
  tcp-request inspect-delay 5s  
  default_backend k8s-master    
backend k8s-master  
  mode tcp  
  option tcplog  
  option tcp-check  
  balance roundrobin  
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100  
  server k8s-master01   192.168.1.10:6443  check  
  server k8s-master02   192.168.1.15:6443  check  
  server k8s-master03   192.168.1.20:6443  check      
  
配置keepalived，master配置不相同  
vim /etc/keepalived/keepalived.conf  
! Configuration File for keepalived  
global_defs {  
    router_id LVS_DEVEL  
    script_user root  
    enable_script_security  
}  
vrrp_script chk_apiserver {  
    script "/etc/keepalived/check_apiserver.sh"  
    interval 5  
    weight -5  
    fall 2    
    rise 1  
}  
vrrp_instance VI_1 {  
    state MASTER  
    interface ens33  
    mcast_src_ip 192.168.1.10  
    virtual_router_id 51  
    priority 101  
    advert_int 2  
    authentication {  
        auth_type PASS  
        auth_pass K8SHA_KA_AUTH  
    }  
    virtual_ipaddress {  
        192.168.1.111  
    }  
    track_script {  
       chk_apiserver  
    }  
}  
![image](https://user-images.githubusercontent.com/80735002/175639425-3820847e-d77a-4914-b067-fb94493f43d3.png)

配置keepalived健康检查脚本  
vim /etc/keepalived/check_apiserver.sh  
#!/bin/bash  
err=0  
for k in $(seq 1 3)  
do  
    check_code=$(pgrep haproxy)  
    if [[ $check_code == "" ]]; then  
        err=$(expr $err + 1)  
        sleep 1  
        continue  
    else  
        err=0  
        break  
    fi  
done  
if [[ $err != "0" ]]; then  
    echo "systemctl stop keepalived"  
    /usr/bin/systemctl stop keepalived  
    exit 1  
else  
    exit 0  
fi  

chmod +x /etc/keepalived/check_apiserver.sh  

启动haproxy和keepalived  
systemctl daemon-reload  
systemctl enable --now haproxy  
systemctl enable --now keepalived  
![image](https://user-images.githubusercontent.com/80735002/175640958-a36e498c-8834-4be8-a2f3-7b9f40c03c20.png)    

查看vip，此时vip漂移到master03上了  
![image](https://user-images.githubusercontent.com/80735002/175644068-f9665efa-9557-4b42-9c61-37f8d92a1ffd.png) 
```

### <span id='g'>集群初始化  </span>
```
以下操作只在master01上执行  
vim kubeadm-config.yml  
# 如果不是高可用集群，vip改为master01的地址、16443改为apiserver的端口（默认是6443）即可  
apiVersion: kubeadm.k8s.io/v1beta2  
bootstrapTokens:  
- groups:  
  - system:bootstrappers:kubeadm:default-node-token  
  token: 7t2weq.bjbawausm0jaxury  
  ttl: 24h0m0s  
  usages:  
  - signing  
  - authentication  
kind: InitConfiguration  
localAPIEndpoint:  
  advertiseAddress: 192.168.1.10  
  bindPort: 6443  
nodeRegistration:  
  criSocket: unix:///var/run/containerd/containerd.sock  
  name: k8s-master01  
  taints:  
  - effect: NoSchedule  
    key: node-role.kubernetes.io/master  
---  
apiServer:  
  certSANs:  
  - 192.168.1.111  
  timeoutForControlPlane: 4m0s  
apiVersion: kubeadm.k8s.io/v1beta2  
certificatesDir: /etc/kubernetes/pki  
clusterName: kubernetes  
controlPlaneEndpoint: 192.168.1.111:16443  
controllerManager: {}  
dns:  
  type: CoreDNS  
etcd:  
  local:  
dataDir: /var/lib/etcd  
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers  
kind: ClusterConfiguration  
kubernetesVersion: v1.24.1 # 更改此处的版本号和kubeadm version一致  
networking:  
  dnsDomain: cluster.local  
  podSubnet: 172.16.0.0/12  
  serviceSubnet: 192.168.0.0/16  
scheduler: {}  

更新下文件  
kubeadm config migrate --old-config kubeadm-config.yml --new-config new_kubeadm_config.yaml  

将文件scp到master02、03  
for i in k8s-master02 k8s-master03; do scp new_kubeadm_config.yaml $i:/root/; done  

所有master提前下载镜像  
一定改源哈，不然国外的可能失败，国内推荐阿里云和清华的  
kubeadm config images pull --config new_kubeadm_config.yaml     
systemctl enable --now kubelet    

仅在master01上操作，初始化  
kubeadm init --config new_kubeadm_config.yaml --upload-certs  
不慌，因为我设置的cpu为1个，直接跳过这个错误即可  
kubeadm init --config new_kubeadm_config.yaml --upload-certs --ignore-preflight-errors=NumCPU  
![image](https://user-images.githubusercontent.com/80735002/175644106-3b1b9797-c6a4-4ebf-bbbe-48a79082e3b2.png)

若初始化失败，重置即可，命令如下  
kubeadm reset -f  
ipvsadm --clear  
rm -rf ~/.kube    

配置环境变量，用于访问集群  
vim /root/.bashrc  
export KUBECONFIG=/etc/kubernetes/admin.conf  
![image](https://user-images.githubusercontent.com/80735002/175644338-b5e6b3f8-8b81-43e7-8033-5139b739516f.png)
source /root/.bashrc
![image](https://user-images.githubusercontent.com/80735002/175644363-1741e104-b9ea-4f85-9910-1b52bd984657.png)

将master02、03加入高可用集群  
kubeadm join 192.168.1.111:16443 --token 7t2weq.bjbawausm0jaxury --discovery-token-ca-cert-hash sha256:7672542b7998ce614d7922d729d7809c09107ff03cf030dfbb57b8b411eccbb5 --control-plane --certificate-key 19faf074a7d05cba1dc7f6245d070cac29bf0c88fddd67e7245eca31583c8dc0 --ignore-preflight-errors=NumCPU  
![image](https://user-images.githubusercontent.com/80735002/175644409-4f188121-a5b9-4f4b-83e3-cdf7a19285af.png) 
不执行图中圈主部分，因为只允许master01执行kubectl命令  

如果token值过期：kubeadm token create --print-join-command  
master需要生成—certificate-key：kubeadm init phase upload-certs  --upload-certs  

master01查看nodes  
![image](https://user-images.githubusercontent.com/80735002/175644947-e4ab73ab-64d4-4f93-863b-6961ecd83b7a.png)
```

### <span id='h'>污点  </span>
```
生产中master节点一般部署系统服务，node节点部署业务、应用，但是我没用node，所以将三台master的污点给去掉  
查看污点
kubectl describe nodes k8s-master |grep Taints  
![image](https://user-images.githubusercontent.com/80735002/175644962-263cc5b5-3e04-4395-974e-624f8bcdced3.png) 

kubectl taint node k8s-master node-role.kubernetes.io/master:NoSchedule-  

再次查看污点
![image](https://user-images.githubusercontent.com/80735002/175644998-f4eb3a93-6362-409d-944c-9e97cde06073.png)
```

### <span id='i'>配置calico网络  </span>
```
仅在master01上执行  
修改pod网段  
查看下etcd中定义的pod网段  
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-cidr= | awk -F= '{print $NF}'  

设置环境变量，用于修改calico里Pod网段  
POD_SUBNET=`cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-cidr= | awk -F= '{print $NF}'`  
sed -i "s#POD_CIDR#${POD_SUBNET}#g" calico.yaml  
kubectl apply -f calico.yaml，我这里是将两个文件整合成为一个calico.yml了  

calico.yml去官网下载即可，官网应该是两个文件，一定要对应好版本  

查看容器状态
kubectl get pods -n kube-system  
![image](https://user-images.githubusercontent.com/80735002/175645027-23e6c9a7-6f4b-4a61-b068-b9087a6ed813.png) 
等会儿就好了  
![image](https://user-images.githubusercontent.com/80735002/175645087-1c6d536d-3c97-4ea1-a841-8a055f63887c.png)
![image](https://user-images.githubusercontent.com/80735002/175645212-3862652a-b951-42aa-bbfc-f69d9f643501.png)
```

### <span id='j'>Metrics部署  </span>
```
在新版的k8s中，系统资源的采集均使用metrics-server，可以通过metrics采集节点和podde 内存、磁盘、cpu和网络的使用率  

将master01家电的front-proxy-ca.crt scp到所有节点  
for i in k8s-master02 k8s-master03;do scp /etc/kubernetes/pki/front-proxy-ca.crt $i:/etc/kubernetes/pki/front-proxy-ca.crt;done  

github地址：https://github.com/kubernetes-sigs/metrics-server/releases  

vim comp.yaml  
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  labels:  
    k8s-app: metrics-server  
  name: metrics-server  
  namespace: kube-system  
---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRole  
metadata:  
  labels:  
    k8s-app: metrics-server  
    rbac.authorization.k8s.io/aggregate-to-admin: "true"  
    rbac.authorization.k8s.io/aggregate-to-edit: "true"  
    rbac.authorization.k8s.io/aggregate-to-view: "true"  
  name: system:aggregated-metrics-reader  
rules:  
- apiGroups:  
  - metrics.k8s.io  
  resources:  
  - pods  
  - nodes  
  verbs:  
  - get  
  - list  
  - watch  
---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRole  
metadata:  
  labels:  
    k8s-app: metrics-server  
  name: system:metrics-server  
rules:  
- apiGroups:  
  - ""  
  resources:  
  - nodes/metrics  
  verbs:  
  - get  
- apiGroups:  
  - ""  
  resources:  
  - pods  
  - nodes  
  verbs:  
  - get  
  - list  
  - watch  
---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: RoleBinding  
metadata:  
  labels:  
    k8s-app: metrics-server  
  name: metrics-server-auth-reader  
  namespace: kube-system  
roleRef:  
  apiGroup: rbac.authorization.k8s.io  
  kind: Role  
  name: extension-apiserver-authentication-reader  
subjects:  
- kind: ServiceAccount  
  name: metrics-server  
  namespace: kube-system  
---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRoleBinding  
metadata:  
  labels:  
    k8s-app: metrics-server  
  name: metrics-server:system:auth-delegator  
roleRef:  
  apiGroup: rbac.authorization.k8s.io  
  kind: ClusterRole  
  name: system:auth-delegator  
subjects:  
- kind: ServiceAccount  
  name: metrics-server  
  namespace: kube-system  
---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRoleBinding  
metadata:  
  labels:  
    k8s-app: metrics-server  
  name: system:metrics-server  
roleRef:  
  apiGroup: rbac.authorization.k8s.io  
  kind: ClusterRole  
  name: system:metrics-server  
subjects:  
- kind: ServiceAccount  
  name: metrics-server  
  namespace: kube-system  
---  
apiVersion: v1  
kind: Service  
metadata:  
  labels:  
    k8s-app: metrics-server  
  name: metrics-server  
  namespace: kube-system  
spec:  
  ports:  
  - name: https  
    port: 443  
    protocol: TCP  
    targetPort: https  
  selector:  
    k8s-app: metrics-server  
---  
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  labels:  
    k8s-app: metrics-server  
  name: metrics-server  
  namespace: kube-system  
spec:  
  selector:  
    matchLabels:  
      k8s-app: metrics-server  
  strategy:  
    rollingUpdate:  
      maxUnavailable: 0  
  template:  
    metadata:  
      labels:  
        k8s-app: metrics-server  
    spec:  
      containers:  
      - args:  
        - --cert-dir=/tmp  
        - --secure-port=4443  
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  
        - --kubelet-use-node-status-port  
        - --metric-resolution=15s  
        - --kubelet-insecure-tls ##添加该参数  
        - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt ## change to front-proxy-ca.crt for kubeadm  
        - --requestheader-username-headers=X-Remote-User ##  
        - --requestheader-group-headers=X-Remote-Group ##  
        - --requestheader-extra-headers-prefix=X-Remote-Extra- ##  
        image: registry.cn-beijing.aliyuncs.com/dotbalo/metrics-server:0.6.1 ##该为国内镜像源地址  
        imagePullPolicy: IfNotPresent  
        livenessProbe:  
          failureThreshold: 3  
          httpGet:  
            path: /livez  
            port: https  
            scheme: HTTPS  
          periodSeconds: 10  
        name: metrics-server  
        ports:  
        - containerPort: 4443  
          name: https  
          protocol: TCP  
        readinessProbe:  
          failureThreshold: 3  
          httpGet:  
            path: /readyz  
            port: https  
            scheme: HTTPS  
          initialDelaySeconds: 20  
          periodSeconds: 10  
        resources:  
          requests:  
            cpu: 100m  
            memory: 200Mi  
        securityContext:  
          allowPrivilegeEscalation: false  
          readOnlyRootFilesystem: true  
          runAsNonRoot: true  
          runAsUser: 1000  
        volumeMounts:  
        - mountPath: /tmp  
          name: tmp-dir  
        - name: ca-ssl  
          mountPath: /etc/kubernetes/pki  
      nodeSelector:  
        kubernetes.io/os: linux  
      priorityClassName: system-cluster-critical  
      serviceAccountName: metrics-server  
      volumes:  
      - emptyDir: {}  
        name: tmp-dir  
      - name: ca-ssl  
        hostPath:  
          path: /etc/kubernetes/pki  
---  
apiVersion: apiregistration.k8s.io/v1  
kind: APIService  
metadata:  
  labels:  
    k8s-app: metrics-server  
  annotations:  
    license: 34C7357D6D1C641D4CA1E369E7244F61  
  name: v1beta1.metrics.k8s.io  
spec:  
  group: metrics.k8s.io  
  groupPriorityMinimum: 100  
  insecureSkipTLSVerify: true  
  service:  
    name: metrics-server  
    namespace: kube-system  
  version: v1beta1  
  versionPriority: 100   
  
kubectl create -f comp.yaml  
查看服务状态  
kubectl get pods -n kube-system -l k8s-app=metrics-server  
![image](https://user-images.githubusercontent.com/80735002/175645235-e420a60f-277c-4aed-973a-39d21e413258.png) 
![image](https://user-images.githubusercontent.com/80735002/175645249-8ed2d326-019e-4bcd-92a5-c94efe7493da.png)
```

### <span id='k'>Dashboard部署  </span>
```
官方地址：https://github.com/kubernetes/dashboard  

安装最新版dashboard  
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.0/aio/deploy/recommended.yaml  
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard  
![image](https://user-images.githubusercontent.com/80735002/175647783-eb04d7d3-f8af-447a-973d-5b1d9575f750.png)

vi admin-user.yaml  
apiVersion: v1  
kind: ServiceAccount    
metadata:  
  name: admin-user  
  namespace: kubernetes-dashboard  
---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRoleBinding  
metadata:  
  name: admin-user  
roleRef:  
  apiGroup: rbac.authorization.k8s.io  
  kind: ClusterRole  
  name: cluster-admin  
subjects:  
- kind: ServiceAccount  
  name: admin-user  
  namespace: kubernetes-dashboard  
  
kubectl apply -f admin-user.yaml  
查看dashboard端口号  
kubectl get svc kubernetes-dashboard -n kubernetes-dashboard  
![Uploading image.png…]()

获取token  
kubectl -n kubernetes-dashboard create token admin-user  
![Uploading image.png…]()

登录  
![Uploading image.png…]()
```

### <span id='l'>命令自动补全</span>  
```
yum -y install bash-completion  
echo "source <(kubectl completion bash)" >> ~/.bashrc  
cat ~/.bashrc  
```



