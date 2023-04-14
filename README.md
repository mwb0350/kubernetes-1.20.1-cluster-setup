# kubernetes-1.20.1-cluster-setup
一看就会的kubernetes 1.20.1集群搭建

### **一、安装前准备**

1、有条件的话直接上腾讯、阿里公有云买3台云主机，最小配置2C2G20G就好

2、没条件的话，就在自己电脑上用Vmware开3台虚拟机，配置一样2C2G20G

3、操作系统CentOS 7.x，本文使用的是7.6版本做演示

### **二、环境初始化**

分别在3台机器上面执行

```dsconfig
#根据规划设置主机名
hostnamectl set-hostname master
hostnamectl set-hostname node1
hostnamectl set-hostname node2
 
#添加hosts
cat >> /etc/hosts << EOF
192.168.153.129 master
192.168.153.130 node1
192.168.153.131 node2
EOF
 
#关闭防火墙
systemctl stop firewalld && systemctl disable firewalld
 
#关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config && setenforce 0
 
#关闭swap
swapoff -a && sed -ri 's/.*swap.*/#&/' /etc/fstab
 
#时间同步
yum install ntpdate -y && ntpdate ntp2.aliyun.com
 
#配置内核参数，将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### **三、安装Docker**

分别在3台机器上面执行

```awk
# 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# 2: 添加软件源信息
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 3: 更新并安装Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# 4: 开启Docker服务
sudo systemctl start docker && systemctl enable docker
 
# 可以通过修改daemon配置文件/etc/docker/daemon.json来使用加速器
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://s2q9fn53.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload && sudo systemctl restart docker
```

### **四、安装kubelet、kubeadm、kubectl**

分别在3台机器上面执行

```awk
#添加kubernetes阿里YUM源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
# 可以用下面的命令查看可以安装的版本
yum list kubeadm --showduplicates | sort -r
 
#安装
yum install -y kubelet-1.20.1-0 kubeadm-1.20.1-0 kubectl-1.20.1-0 --disableexcludes=kubernetes
 
# 启动开机启动kubelet
systemctl start kubelet
systemctl enable kubelet
```

### **五、部署Kubernetes Master**

只在master上面执行

```routeros
# 创建集群
kubeadm init --kubernetes-version=v1.20.1 --apiserver-advertise-address=192.168.153.129  --image-repository  registry.aliyuncs.com/google_containers --service-cidr=10.10.0.0/16 --pod-network-cidr=10.122.0.0/16
 
参数说明：
kubernetes-version：要安装的版本
pod-network-cidr：负载容器的子网网段
service-cidr：svc网段
image-repository：指定镜像仓库
apiserver-advertise-address：节点绑定的服务器ip(写master节点IP)
#提示initialized successfully！表示初始化成功
 
# 接着根据提示拷贝admin.config的内容到当前用户的$HOME/.kube/config中并授权
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
 
#执行完后可以运行查看node节点情况，这时候可以看到master节点是noready状态，原因是没有部署网络插件
kubectl get nodes
```

### **六、部署网络插件**

常用的网络插件有flannel和calico，这里选择flannel

```awk
# 创建目录
mkdir -p /opt/yaml
# 编辑kube-flannel.yaml
获取部署flannel的yaml文件（https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml）到/opt/yaml目录中，（链接需要翻墙才能访问，这边贴下获取到的内容）
vim /opt/yaml/kube-flannel.yaml
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.flannel.unprivileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
    apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
spec:
  privileged: false
  volumes:
  - configMap
  - secret
  - emptyDir
  - hostPath
  allowedHostPaths:
  - pathPrefix: "/etc/cni/net.d"
  - pathPrefix: "/etc/kube-flannel"
  - pathPrefix: "/run/flannel"
  readOnlyRootFilesystem: false
  # Users and groups
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  # Privilege Escalation
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  # Capabilities
  allowedCapabilities: ['NET_ADMIN', 'NET_RAW']
  defaultAddCapabilities: []
  requiredDropCapabilities: []
  # Host namespaces
  hostPID: false
  hostIPC: false
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  # SELinux
  seLinux:
    # SELinux is unused in CaaSP
    rule: 'RunAsAny'
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups: ['extensions']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames: ['psp.flannel.unprivileged']
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.14.0
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.14.0
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
# 将quay.io换成quay.mirrors.ustc.edu.cn（中科大）的镜像
sed -i 's#quay.io/coreos/flannel#quay.mirrors.ustc.edu.cn/coreos/flannel#' /opt/yaml/kube-flannel.yaml
 
# 部署flannel
kubectl apply -f /opt/yaml/kube-flannel.yaml
#等个几分钟，再看master上节点状态就是ready了
```

### **七、扩容node节点进集群**

在两台node上面执行添加命令

```gauss
#命令在第五步，创建集群成功后会显示在界面上
kubeadm join 192.168.153.129:6443 --token 7spol7.nxxabat5ljyws8xo --discovery-token-ca-cert-hash sha256:4f31b94b2beda59334b5da507673752605ab49fde6ef87487c81b314a39b466f
 
#如果忘记token，可以在master节点重新生成，token有效期24小时
kubeadm token create --print-join-command
 
#在master节点上运行查看node节点命令，就可以看到3台node节点了
```

### **八、检查集群状态**

```awk
# 查看node状态都是ready
kubectl get nodes
 
# 看出pod状态都是running
kubectl get pod  --all-namespaces
 
#查看集群状态都是Healthy ok
kubectl get cs
如果出现unhealthy，分别修改下面两个文件，注释 - --port=0，然后重启kubelet
vim /etc/kubernetes/manifests/kube-controller-manager.yaml
vim /etc/kubernetes/manifests/kube-scheduler.yaml
systemctl restart kubelet.service
```
