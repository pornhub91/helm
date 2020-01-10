## helm安装Prometheus/grafana监控k8s集群信息

确保K8S组件时间同步无异常，如果没有则先同步时间： 

    #配置亚洲时区
    rm -f /etc/localtime
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    #配置阿里云时间服务器
    yum -y install ntp &> /dev/null
    ntpdate -u ntp1.aliyun.com
    hwclock -w

安装helm： 

    #下载 Helm 
    wget https://get.helm.sh/helm-v3.0.0-linux-amd64.tar.gz
    #解压 Helm
    tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
    #复制执行文件到 bin 目录下
    cp linux-amd64/helm /usr/local/bin/
    #添加官方仓库地址
    helm repo add stable https://kubernetes-charts.storage.googleapis.com
    #查看本地已添加的存储库
    helm search repo stable
拉取prometheus镜像：

    helm pull stable/prometheus-operator
    #解压
    tar zxvf prometheus-operator-8.3.2.tgz
    #以下操作都在此目录执行
    cd prometheus-operator/
    
    #安装CRD：
    kubectl apply -f crds/
    #创建名称空间：
    kubectl create namespace monitoring
    #安装Prometheus-Operator
    helm install prometheus --namespace=monitoring ./ --set prometheusOperator.createCustomResource=false
    #卸载命令
    helm uninstall prometheus --namespace=monitoring  

安装nfs组件以及配置storageclass的依赖项：
```
#创建挂载相关目录
mkdir -p /data/k8s

#所有节点都要安装并启动nfs-utils 
yum -y install  nfs-utils  rpcbind &> /dev/null
systemctl restart nfs-utils   
systemctl restart rpcbind     && systemctl enable rpcbind   
systemctl restart nfs-server  && systemctl enable nfs-server


#创建nfs访问规则
echo "/data/k8s  *(rw,async,no_root_squash)" >> /etc/exports
exportfs -r

#检查nfs是否正常
showmount -e localhost
```

创建storageclass相关配置文件，先创建nfs-service-account授权
```
cat nfs-sa.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```

创建nfs存储,只需要修改对应的主机和挂载目录即可
```
cat nfs-client.yaml 
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  selector:
    matchLabels:
      app: nfs-client-provisioner
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs-storage
            - name: NFS_SERVER
              value: 192.168.37.10   #修改为nfsIP地址
            - name: NFS_PATH
              value: /data/k8s       #路径可自行修改
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.37.10 #修改为nfsIP地址
            path: /data/k8s       #路径可自行修改

```
创建storageclass，当pod匹配到storageclass时，会自动创建pvc，以下是创建了两个storageclass
```
cat nfs-class.yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: prometheus-nfsclass
provisioner: nfs-storage
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: grafana-nfsclass
provisioner: nfs-storage
```
### 以下是文件修改相关配置，我会将整个chart上传，如果使用下面的修改无效，直接使用我上传的文件来使用

修改values.yaml里的存储项：

```
vim values.yaml
...
    storageSpec:      #在160817行
      volumeClaimTemplate:
        spec:
          storageClassName: prometheus-nfsclass   #修改为上面创建的class名
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi   #可以调整存储大小
...

```

修改grafana的存储项：

```
vim charts/grafana/values.yaml
...
persistence:    #1883行
  type: pvc
  enabled: true
  storageClassName: grafana-nfsclass
  accessModes:
    - ReadWriteOnce
  size: 10Gi
  finalizers:
    - kubernetes.io/pvc-protection
...
```
修改Prometheus监听端口：

    vim values.yaml
    #修改Prometheus监听集群Etcd端口，文件的741行
    ...
      service:
        port: 2381  #默认为2379，修改为2381
        targetPort: 2381 #默认为2379，修改为2381
        # selector:
        #   component: etd
    ...
    
    #Prometheus对外端口默认是30090，文件的1226行
    #网络类型默认是clusterIP，文件的1234行
    
    ...
    nodePort: 30090 #默认端口
    ## Loadbalancer IP
    ## Only use if service.type is "loadbalancer"
    loadBalancerIP: ""
    loadBalancerSourceRanges: []
    ## Service type
    ##
    type: NodePort #默认为clusterIP，修改为NodePort
    ...
    
修改grafana端口：
    
    vim charts/grafana/values.yaml
    ...
    service:
     type: NodePort  #默认为clusterIP，修改为NodePort用于外部访问
     port: 80
     targetPort: 3000
     nodePort: 30090 #添加此行指定访问端口，如不指定则随机生成
        # targetPort: 4181 To be used with a proxy extraContainer
     annotations: {}
     labels: {}
     portName: service
    ... 
##### 修改配置后更新：

	helm upgrade prometheus ./ -nmonitoring
	#Grafana默认密码是：prom-operator
	#可以在values.yaml文件里修改：adminPassword: prom-operator
本次部署使用的是kubeadm安装，安装后etcd的监听在127.0.0.1，修改为监听所有网段
```
vim /etc/kubernetes/manifests/etcd.yaml
- --listen-metrics-urls=http://127.0.0.1:2381 修改为：
- --listen-metrics-urls=http://0.0.0.0:2381
#修改后删除etcd pod，K8S会自动创建
kubectl get pod -nkube-system
kubectl delete pod/etcd-master -nkube-system
```
准备工作基本做完了，浏览器访问Prometheus查看
![Image text](https://github.com/pornhub91/helm/blob/master/png/prometheus.png)

![Image text](https://github.com/pornhub91/helm/blob/master/png/Prometheus1.png)

![Image text](https://github.com/pornhub91/helm/blob/master/png/grafana.png)

![Image text](https://github.com/pornhub91/helm/blob/master/png/grafana1.png)

![Image text](https://github.com/pornhub91/helm/blob/master/png/grafana2.png)
### 问题记录
![问题记录](https://github.com/pornhub91/helm/blob/master/png/error.png)
搭建后如果出现monitoring/prometheus-prometheus-oper-kube-proxy/0 (0/3 up) 无法监控到的问题，需要更改kube-proxy默认的configmap
```
#查看kube-system名称空间下的configmap
kubectl get cm -nkube-system
NAME                                 DATA   AGE
coredns                              1      22d
extension-apiserver-authentication   6      22d
kube-flannel-cfg                     2      22d
kube-proxy                           2      22d
kubeadm-config                       2      22d
kubelet-config-1.17                  1      22d

#修改configmap
kubectl edit cm/kube-proxy -nkube-system
...
    kind: KubeProxyConfiguration
    metricsBindAddress: "0.0.0.0:10249"#10249是默认的proxy metrics监听端口，可能会发生此配置为空的情况，这时需要手动修改为0.0.0.0：10249
    mode: ""
    nodePortAddresses: null
...
```
