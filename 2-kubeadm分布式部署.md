## 官方提供Kubernetes部署3种方式 ##

- minikube
- kubeadm
- 二进制包### 1. 安装要求- 操作系统
- Ubuntu 16.04+
- Debian 9
- CentOS 7
- RHEL 7
- Fedora 25/26 (best-effort)
- 其他
- 内存2GB + ，2核CPU +
- 集群节点之间可以通信
- 每个节点唯一主机名，MAC地址和product_uuid
- 检查MAC地址：使用ip link或者ifconfig -a
- 检查product_uuid：cat /sys/class/dmi/id/product_uuid
- 禁止swap分区。这样才能使kubelet正常工作

    `关闭swap分区`
# 
    # swapoff -a### 2. 准备环境    关闭防火墙：
    # systemctl stop firewalld
    # systemctl disable firewalld
    关闭selinux：    # sed -i 's/enforcing/disabled/' /etc/selinux/config
    # setenforce 0
    关闭swap：
    # swapoff -a # 临时
    # vim /etc/fstab # 永久
    添加主机名与IP对应关系：
    # cat /etc/hosts
    192.168.0.11 k8s-master
    192.168.0.12 k8s-node1
    192.168.0.13 k8s-node2
    同步时间：
    # yum install ntpdate -y
    # ntpdate ntp.api.bz### 3. 安装Docker    # yum install -y yum-utils device-mapper-persistent-data lvm2
    # yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    目前docker最大支持docker-ce-17.03，所以要指定该版本安装：
    # yum install docker-ce-17.03.3.ce -y
    如果提示container-selinux依赖问题，先安装ce-17.03匹配版本：
    # yum localinstall
    https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux17.03.3.ce-1.el7.noarch.rpm
    # systemctl enable docker && systemctl start docker

### 4. 安装kubeadm，kubelet和kubectl

- kubeadm： 引导集群的命令
- kubelet：集群中运行任务的代理程序
- kubectl：命令行管理工具#### 4.1 添加阿里云YUM软件源    cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
    EOF

# 
#### 4.2 安装kubeadm，kubelet和kubectl
    setenforce 0
    yum install -y kubelet kubeadm kubectl  --disableexcludes=kubernetes
    systemctl enable kubelet && systemctl start kubelet



注意：使用Docker时，kubeadm会自动检查kubelet的cgroup驱动程序，并/var/lib/kubelet/kubeadm-flags.env
在运行时将其设置在文件中。如果使用的其他CRI，则必须在/etc/default/kubelet中cgroup-driver值修改为
cgroupfs：

    # cat /var/lib/kubelet/kubeadm-flags.env
    KUBELET_KUBEADM_ARGS=--cgroup-driver=cgroupfs --cni-bin-dir=/opt/cni/bin --cni-confdir=/etc/cni/net.d
    --network-plugin=cni
    # systemctl daemon-reload
    # systemctl restart kubelet

### 5. 使用kubeadm创建单个Master集群
#### 5.1 默认下载镜像地址在国外无法访问，先从准备好所需镜像
保存到脚本之间运行：

    K8S_VERSION=v1.11.2
    ETCD_VERSION=3.2.18
    DASHBOARD_VERSION=v1.8.3
    FLANNEL_VERSION=v0.10.0-amd64
    DNS_VERSION=1.1.3
    PAUSE_VERSION=3.1
    # 基本组件
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:$K8S_VERSION
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:$K8S_VERSION
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:$K8S_VERSION
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:$K8S_VERSION
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64:$ETCD_VERSION
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$DNS_VERSION
    # 网络组件
    docker pull quay.io/coreos/flannel:$FLANNEL_VERSION
    # 修改tag
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:$K8S_VERSION k8s.gcr.io/kube-apiserver-amd64:$K8S_VERSION
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:$K8S_VERSION k8s.gcr.io/kube-controller-manager-amd64:$K8S_VERSION
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:$K8S_VERSION k8s.gcr.io/kube-scheduler-amd64:$K8S_VERSION
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:$K8S_VERSION k8s.gcr.io/kube-proxy-amd64:$K8S_VERSION
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64:$ETCD_VERSION k8s.gcr.io/etcd-amd64:$ETCD_VERSION
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION k8s.gcr.io/pause:$PAUSE_VERSION
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$DNS_VERSION k8s.gcr.io/coredns:$DNS_VERSION


#### 5.2 初始化Master

    清空网络规则：
    setenforce 0
    iptables -F
    iptables -t nat -F
    iptables -I FORWARD -s 0.0.0.0/0 -d 0.0.0.0/0 -j ACCEPT  

# 
    
    # kubeadm init --kubernetes-version=1.11.2 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.131.10
    
#
	mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

#### 5.3 安装Pod网络 - 插件
    # kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
#### 5.4 加入工作节点


在Node节点切换到root账号：
格式：kubeadm join --token : --discovery-token-ca-cert-hash sha256:

    # kubeadm join 192.168.0.11:6443 --token 6hk68y.0rdz1wdjyh85ntkr --discovery-token-cacert-hash sha256:d1d3f59ae37fbd632707cbeb9b095d0d0b19af535078091993c4bc4d9d2a7782


### 6. kubernetes dashboard

    # wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml修改镜像地址：    # registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.0修改Service：    # kubectl apply -f kubernetes-dashboard.yaml创建一个管理员角色k8s-admin.yaml：
    apiVersion: v1    kind: ServiceAccount    metadata:      name: dashboard-admin      namespace: kube-system    ---    kind: ClusterRoleBinding    apiVersion: rbac.authorization.k8s.io/v1beta1    metadata:      name: dashboard-admin    subjects:      - kind: ServiceAccount        name: dashboard-admin        namespace: kube-system    roleRef:      kind: ClusterRole      name: cluster-admin      apiGroup: rbac.authorization.k8s.iomaster执行:    # kubectl apply -f k8s-admin.yaml使用上述创建账号的token登录Kubernetes Dashboard：    # kubectl get secret -n kube-system    # kubectl describe secret dashboard-admin-token-bwdjj -n kube-system#    [root@k8s-master ~]# kubectl describe secret dashboard-admin-token-9frch -n kube-system    Name: dashboard-admin-token-9frch    Namespace:kube-system    Labels:   <none>    Annotations:  kubernetes.io/service-account.name=dashboard-admin              kubernetes.io/service-account.uid=1a72e69a-bbad-11e8-ba3d-000c294ad95a    Type:  kubernetes.io/service-account-token    Data    ====    ca.crt: 1025 bytes    namespace:  11 bytes`token:`     `eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tOWZyY2giLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMWE3MmU2OWEtYmJhZC0xMWU4LWJhM2QtMDAwYzI5NGFkOTVhIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.fgGxZL86NvXK1LLa1PVlEIq1mQLNMIsXeEVDOKQ9n5WGi5DvrUBjORPxKNWiGhDse7xvSCPoQv7Vh-lWErAnjtNaU2IJyL_s0MbqgxRiKnhz2BL2s50stFqqVdSLAvbtaae-J4-lJkTWZsk4aC9FwTIaavc020zgpKrtZLV26QQRxT_C_18BN1nMekf_gjJ4-6avxkgfq05Zk4vxihQWHdeFHb53Uf9AfLfkKQvlExM0SxFpkL9qGGhATl4iDcWlXrltoLn_YLSr2SsaskgaRVQBqhhRfYkUjIJELMz2ERqixQI1721fshJR6ImY7cbs2MRFNqsgD0xltuPgH95Ciw`接下来使用token令牌登录即可！至此，一个完整的分布式k8s系统就搭建成功了！撒花完结！啾啾啾ing