# jmeter+grafana+influxdb可视化性能监控平台

### 1、安装influxdb

```
docker search influx
docker pull influxdb
docker run --name my_influ -p 8086:8086 influxdb
docker exec -it my_influ /bin/bash
```

创建数据库

```
influx
create database jmeter
show databases
exit
```

### 2、安装grafana

```
docker search grafana
docker pull grafana/grafana
docker run --name my_grafana -p 3000:3000 grafana/grafana
```

进入容器也可以通过docker的k8s进入

![image-20200609004055644](/images/image-20200609004055644.png)

### 3、配置

#### 避坑提示

> 访问使用内网私有地址如本机的192.168.31.209，不要使用127.0.0.1，不然有坑

访问grafana http://192.168.31.209:3000/，登录admin/admin

#### 添加数据源

即刚才创建的数据库

![image-20200609005038118](/images/image-20200609005038118.png)

配置如下

![image-20200609004930603](/images/image-20200609004930603.png)

#### 展示模板

https://grafana.com/grafana/dashboards/5496

下载对应的json文件，导入到grafana

![image-20200609005646865](/images/image-20200609005646865.png)

导入成功后可以看到对应的界面。

### 4、jemeter配置

#### 安装与启动

*略*

#### 线程组配置

![image-20200609010030461](/images/image-20200609010030461.png)

![image-20200609010317113](/images/image-20200609010317113.png)

![image-20200609010055881](/images/image-20200609010055881.png)

#### 后端监听器配置

![image-20200609010201308](/images/image-20200609010201308.png)

![image-20200609010412184](/images/image-20200609010412184.png)

### 5、开始压测

点击jemeter启动按钮开始压测

### 6、测试结果

![2020-06-08T17-11-47.922Z](/images/2020-06-08T17-11-47.922Z.png)


