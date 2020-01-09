## 部署MySQL监控并配置grafana绘图
授权账号访问数据库 
```
#创建用户
CREATE  USER root IDENTIFIED  BY  '123456';
#授权用户
GRANT ALL PRIVILEGES ON   *.*  TO  'root'@'%';
#刷新权限
FLUSH  PRIVILEGES;
```
Docker运行mysql-exporter监控
```
#授权账号密码、IP地址、端口自行修改
docker run -d --restart=always \
  -p 9104:9104 --name mysqld_exporter \
  -e DATA_SOURCE_NAME="root:1@(192.168.37.10:3306)/" \
  prom/mysqld-exporter
```
测试访问对应主机的9104端口，可以看到很多数据，这就代表exporter监控成功了，下面将数据绘制成图表展示
本此使用helm环境安装Prometheus，将采集数据的地址加入到Prometheus，需要三个文件
```
cat mysql_endpoint.yml 
apiVersion: v1
kind: Endpoints
metadata:
  name: mysql-metrics   
  namespace: monitoring  #和Prometheus在同一个名称空间
  labels:
    k8s-app: node-mysql   
subsets:
  - addresses:
    - ip: 192.168.37.10   #运行docker-exporter的主机IP
    ports:
    - name: mysql
      port: 9104         #exporter的端口
      protocol: TCP
```
```
cat mysql_servermonitor.yml 
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  namespace: monitoring
  name: mysql-metrics
  labels:
    k8s-app: node-mysql
    app: prometheus-operator-prometheus
    chart: prometheus-operator-8.3.1
    heritage: Helm
    release: prometheus
spec:
  selector:
    matchLabels:
      k8s-app: node-mysql
  namespaceSelector:
    matchNames:
    - monitoring
  endpoints:
  - port: mysql
    interval: 10s
    honorLabels: true
```
```
cat mysql_service.yml 
apiVersion: v1
kind: Service
metadata:
  name: mysql-metrics
  namespace: monitoring
  labels:
    k8s-app: node-mysql
spec:
  type: ClusterIP
  ports:
  - name: mysql
    port: 9104
    protocol: TCP
    targetPort: 9104
```
生成相关文件

`kubectl apply -f .`
### 访问Prometheus，配置grafana绘图
![Image text](https://github.com/pornhub91/helm/blob/master/png/MySQL.png)
![Image text](https://github.com/pornhub91/helm/blob/master/png/MySQL1.png)
![Image text](https://github.com/pornhub91/helm/blob/master/png/MySQL2.png)
