# Kubernets安装部署

## 准备三台虚拟机

|   IP|主机名   |作用   |
| ------------ | ------------ | ------------ |
|  10.15.10.10 |Master   |主管理   |
|  10.15.10.11|Node1  |从节点1|
|  10.15.10.12|Node2  |从节点2|


## 三台虚拟机同时执行

- 修改主机名并添加到hosts文件中
```shell
cat >> /etc/hosts << EOF
10.15.10.10 Master
10.15.10.11 Node1
10.15.10.12 Node2
EOF
```
- 时间同步
```shell
systemctl start chronyd && systemctl enable chronyd
date
```
- 禁用firewalld
```shell
systemctl stop firewalld && systemctl disable firewalld
```
- 禁用selinux
```shell
setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```
- 禁用swap
```shell
#临时关闭
swapoff -a
#永久关闭
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
默认情况下，K8s为了追求高性能，不建议使用交换分区，为此它要求每个节点禁用swap，否则各个节点中的kubelet无法运行
- 网桥设置
```shell
cat > /etc/sysctl.d/kubernets.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system
```
- 安装docker
```shell
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum -y install docker-ce
systemctl enable docker && systemctl restart docker
docker -v
```
-配置daemon.json
```shell
vim /etc/docker/daemon.json
{
  "exec-opts": [ "native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://docker.m.daocloud.io"]
}
wq 退出编辑模式
systemctl daemon-reload && systemctl restart docker
```
- 安装k8s
1镜像
```shell
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
cat /etc/yum.repos.d/kubernetes.repo
```
2安装
```shell
yum install -y kubeadm-1.23.17 kubectl-1.23.17 kubelet-1.23.17
systemctl enable kubelet
journalctl -xefu kubelet
```
## 异步执行
### Master
- 设置主机名
```shell
hostnamectl set-hostname Master
```
初始化Kubernetes，保存该命令输出的一个join命令，该join命令需要在node角色的节点上执行
```shell
kubeadm init \
--apiserver-advertise-address=10.15.10.10 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.23.17 \
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16 \
--ignore-preflight-errors=all
```
- 环境配置
1普通用户
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
2root用户
```shell
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /etc/profile
source /etc/profile
```
- 配置flannel
```shell
cd /opt
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
- 上传
```shell
kubectl apply -f /opt/kube-flannel.yml 
```
### node
- 设置主机名
```shell
hostnamectl set-hostname node1
# 两台节点分别执行
hostnamectl set-hostname node2
```
- 执行kubeadm join命令，该命令是master节点 初始化k8s时生成的
```shell
kubeadm join 10.15.10.10:6443 --token gd1cai.ui4b3g2at99x45dv \
	--discovery-token-ca-cert-hash sha256:319e6c28368cca00eedb8458f2cf2ecef2e9d53c96f45f62e8b7f6113ad994ae 
```
上面命令中token默认有效期为24小时，过期后可以在master节点执行kubeadm token create --print-join-command重新创建token
环境配置
```shell
echo "export KUBECONFIG=/etc/kubernetes/kubelet.conf" >> /etc/profile
source /etc/profile
```
###节点状态
```shell
kubectl get node
NAME     STATUS     ROLES                  AGE     VERSION
master   NotReady   control-plane,master   16m     v1.23.17
node1    NotReady   <none>                 3m52s   v1.23.17
node2    NotReady   <none>                 3m47s   v1.23.17
```
## 使用k8s安装Nginx
- 配置
```shell
vim /opt/nginx.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
          image: nginx:latest
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              protocol: TCP
              containerPort: 80
          resources:
            limits:
              cpu: "1.0"
              memory: 512Mi
            requests:
              cpu: "0.5"
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  annotations:
  name: nginx-service
spec:
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32001
      protocol: TCP
  selector:
    app: nginx
  sessionAffinity: None
  type: NodePort
```
- 上传
```shell
kubectl apply -f /opt/nginx.yaml 
```
- 状态
```shell
kubectl get pod,svc
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-59d7db4f6f-c8kxp   0/1     Pending   0          64s
pod/nginx-deployment-59d7db4f6f-hvb57   0/1     Pending   0          64s
pod/nginx-deployment-59d7db4f6f-rztv2   0/1     Pending   0          64s

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP        46m
service/nginx-service   NodePort    10.104.70.59   <none>        80:32001/TCP   64s

```
# 111111111111
```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.15.10.10:6443 --token l5k69z.cx8rjeti1pm4pade \
	--discovery-token-ca-cert-hash sha256:52a33e07705e673bfb26d6f301c323aba598e9192ef41be0afe998bacbd01982 
```







#aaa
