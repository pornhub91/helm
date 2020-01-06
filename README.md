## helm安装Prometheus/grafana监控k8s集群信息

确保K8S组件时间同步无异常，如果没有则先同步时间： 

    #配置阿里云时间服务器
    sed -i "s^server 0.centos.pool.ntp.org iburst^server ntp1.aliyun.com iburst^g" /etc/chrony.conf
    sed -i "s^server 1^#server 1^g" /etc/chrony.conf
    sed -i "s^server 2^#server 2^g" /etc/chrony.conf
    sed -i "s^server 3^#server 3^g" /etc/chrony.conf
    sed -i "s^server 4^#server 4^g" /etc/chrony.conf
    systemctl restart chronyd.service
    #配置亚洲时区
    rm -f /etc/localtime
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

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
    cd prometheus-operator/
    
    #安装CRD：
    kubectl apply -f crds/
    #创建名称空间：
    kubectl create namespace monitoring
    #安装Prometheus-Operator
    helm install prometheus --namespace=monitoring ./ --set prometheusOperator.createCustomResource=false
    
修改端口相关配置：

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
修改配置后更新：

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
