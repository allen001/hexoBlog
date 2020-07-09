---
title: Elasticsearch && Kibana容器化部署
date: 2020-05-22 16:01:48
toc: true
comments: true
tags:
- springboot
- elasticsearch
- kibana
---

## Elasticsearch搭建步骤

1：安装es的镜像

```bash
docker pull elasticsearch:6.7.1
```

2：在`/user/local`目录创建一个文件夹`ES/config`

```bash
# 创建文件夹es/config
mkdir -p /user/local/ES/config
# 进入config
cd /user/local/ES/config/
# 分别创建三个文件
vim es1.yml
vim es2.yml
vim es3.yml
```

3：分别三个文件内容如下：

`es1.yml`解读配置参数意义

```bash
#集群唯一名称，所有节点一致
cluster.name: elasticsearch-cluster
#节点名称
node.name: es-node1
#设置可以访问的ip,默认为0.0.0.0，这里全部设置通过
network.host: 0.0.0.0
#设置其它结点和该结点交互的ip地址
network.publish_host: 127.0.0.1
#设置对外服务的http端口，默认为9200
http.port: 9200
#设置节点之间交互的tcp端口，默认是9300
transport.tcp.port: 9300
# 是否支持跨域，是：true
http.cors.enabled: true
# 表示支持所有域名
http.cors.allow-origin: "*"
#配置该结点是否有资格被选举为主结点（候选主结点），为了防止脑裂，配置奇数个候选主结点
node.master: true
#配置该结点是数据结点，用于保存数据
node.data: true
#集群个节点IP地址
discovery.zen.ping.unicast.hosts:["127.0.0.1:9300","127.0.0.1:9301","127.0.0.1:9302"]
#自动发现master节点的最小数
discovery.zen.minimum_master_nodes: 1
```

<!-- more -->

es1.yml

```bash
cluster.name: elasticsearch-cluster
node.name: es-node1
network.host: 0.0.0.0
network.publish_host: 127.0.0.1
http.port: 9200
transport.tcp.port: 9300
http.cors.enabled: true
http.cors.allow-origin: "*"
node.master: true
node.data: true
discovery.zen.ping.unicast.hosts:["127.0.0.1:9300","127.0.0.1:9301","127.0.0.1:9302"]
discovery.zen.minimum_master_nodes: 1
```

es2.yml

```bash
cluster.name: elasticsearch-cluster
node.name: es-node2
network.host: 0.0.0.0
network.publish_host: 127.0.0.1
http.port: 9201
transport.tcp.port: 9301
http.cors.enabled: true
http.cors.allow-origin: "*"
node.master: true
node.data: true
discovery.zen.ping.unicast.hosts:["127.0.0.1:9300","127.0.0.1:9301","127.0.0.1:9302"]
discovery.zen.minimum_master_nodes: 1
```

es3.yml

```bash
cluster.name: elasticsearch-cluster
node.name: es-node3
network.host: 0.0.0.0
network.publish_host: 127.0.0.1
http.port: 9202
transport.tcp.port: 9302
http.cors.enabled: true
http.cors.allow-origin: "*"
node.master: true
node.data: true
discovery.zen.ping.unicast.hosts:["127.0.0.1:9300","127.0.0.1:9301","127.0.0.1:9302"]
discovery.zen.minimum_master_nodes: 1
```

4：分别创建三个容器

查看elasticsearch:6.7.1的容器Id `e2667f5db289`

```bash
docker run -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d -p 9200:9200 -p 9300:9300 -v /usr/local/es/config/es1.yml:/usr/share/elasticsearch/config/elasticsearch.yml --name ES01 e2667f5db289

docker run -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d -p 9201:9201 -p 9301:9301 -v /usr/local/es/config/es2.yml:/usr/share/elasticsearch/config/elasticsearch.yml --name ES02 e2667f5db289

docker run -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d -p 9202:9202 -p 9302:9302 -v /usr/local/es/config/es3.yml:/usr/share/elasticsearch/config/elasticsearch.yml --name ES03 e2667f5db289
```

**如果遇到elasticsearch启动时遇到的错误**

```bash
#最大虚拟内存区域vm.max_map_count[65530]太低，请至少增加到[262144]
max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```

> 在 /etc/sysctl.conf文件最后添加一行，退出系统目录
>
> ```bash
> vim /etc/sysctl.conf
> #最后一行添加
> vm.max_map_count = 655360
> #并执行命令：
> sysctl -p
> #然后，重新启动elasticsearch，即可启动成功。
> ```
>
> 即可永久修改

重启三个es容器

```bash
docker restart ES01 ES02 ES03
```

访问：http://127.0.0.1:9200

访问：http://127.0.0.1:9201

访问：http://127.0.0.1:9202

查看ES集群是否搭建成功

访问：http://127.0.0.1:9200/_cat/nodes?pretty

![](https://i.loli.net/2020/05/22/ypr972ltbwuoMUv.png)

集群配置成功

![](https://i.loli.net/2020/05/22/tklK8XIZc43y9nB.png)

5：检查容器三个集群容器是否启动

```bash
docker ps -a
```

![](https://i.loli.net/2020/05/22/N8bH1tBJ6OV7ElY.png)

安装配置 Elasticsearch-Head 管理界面

```bash
# Docker 部署 ElasticSearch-Head
# 为什么要安装ElasticSearch-Head呢，原因是需要有一个管理界面进行查看ElasticSearch相关信息
# 拉取镜像
docker pull mobz/elasticsearch-head:5
# 运行容器
docker run -d --name es_admin -p 9100:9100 mobz/elasticsearch-head:5

# 等待启动成功
```

6：然后通过`es-head`客户端连接

![](https://i.loli.net/2020/05/22/QdlHjzoetfnsvCS.png)

在上面的连接中输入任何节点的地址都可以。比如：

http://127.0.0.1:9200

http://127.0.0.1:9201

http://127.0.0.1:9202

![](https://i.loli.net/2020/05/22/yWvJah2NUq4b18d.png)

**页面数据查询报406错误**

![](https://i.loli.net/2020/05/22/evR1DUiJuFSVPch.png)

* 进入head安装目录（cd进 _site/ 目录，编辑vendor.js共2处）

```bash
docker exec -it es_admin /bin/bash
ls
cd _site/
```

**定位到6886行 :6886**

> 6886行==> contentType: “application/x-www-form-urlencoded"
>
>  改成 
>
> contentType: "application/json;charset=UTF-8"

**定位到7573行 :7573**

> 7573行==> var inspectData = s.contentType === “application/x-www-form-urlencoded” &&
>
> 改成
>
> var inspectData = s.contentType === "application/json;charset=UTF-8" &&

如果提示找不到vim方法，参考下面

```bash
# 更新源
[root@localhost ~]# apt-get update
# 先更新，防止提示：Unable to locate package vim
[root@localhost ~]# apt-get install vim
```

安装失败apt-get或安装速度太慢：
实际在使用过程中，运行 apt-get update，然后执行 apt-get install vim，下载地址由于是海外地
址，下载速度异常慢而且可能中断更新流程，所以做下面配置

```bash
mv /etc/apt/sources.list /etc/apt/sources.list.bak
echo "deb http://mirrors.163.com/debian/ jessie main non-free contrib" >>
/etc/apt/sources.list
echo "deb http://mirrors.163.com/debian/ jessie-proposed-updates main non-
free contrib" >>/etc/apt/sources.list
echo "deb-src http://mirrors.163.com/debian/ jessie main non-free contrib"
>>/etc/apt/sources.list
echo "deb-src http://mirrors.163.com/debian/ jessie-proposed-updates main
non-free contrib" >>/etc/apt/sources.list
```

重启容器

```bash
docker restart es_admin
```

![](https://i.loli.net/2020/05/22/WzhxKHyRTbFJafm.png)

## Kibana图形化管理ES

1：下载Kibana镜像

```bash
# 下载Kibana镜像
docker pull kibana:6.7.1
# 查看镜像
docker images
```

2：编辑kibana.yml配置文件

```bash
cd /usr/local/ES
mkdir -p data/elk/
```

kibana.yml配置文件放在宿主机 /data/elk/目录下，内容如下：

```bash
server.name: kibana
server.host: "0"
elasticsearch.hosts: ["http://127.0.0.1:9200","http://127.0.0.1:9201","http://127.0.0.1:92
02"]
xpack.monitoring.ui.container.elasticsearch.enabled: true
```

3：运行Kibana

```bash
docker run -d --restart=always --log-driver json-file --log-opt max-size=100m --log-opt max-file=2 --name xinyar-kibana -p 5601:5601 -v /usr/local/ES/data/elk/kibana.yml:/usr/share/kibana/config/kibana.yml kibana:6.8.8
```

```bash
# 查看容器启动状态
docker ps
```

4：访问Kibana

访问 http://127.0.0.1:5601 (启动可能会较慢，如失败等几秒再尝试刷新一下)

![](https://i.loli.net/2020/05/22/mGZpv9VIwgzDWTO.png)

![](https://i.loli.net/2020/05/22/mKatWEQLow9qCTZ.png)



## Springboot && Elasticsearch

github地址：https://github.com/allen001/es-search

修改application.properties中es连接ip地址

## 资料

* [ElasticSearch6.5.4<一> 单机部署以及简单尝试](https://blog.csdn.net/qq_25012687/article/details/98728578)

* [ElasticSearch6.5.4<二> 几个重要概念以及常用搜索](https://blog.csdn.net/qq_25012687/article/details/100825774)

* [ElasticSearch6.5.4<三> 中文以及拼音的操作](https://blog.csdn.net/qq_25012687/article/details/100851367)
* [ElasticSearch6.5.4<四> Java使用ES并实战搜索](https://blog.csdn.net/qq_25012687/article/details/100877285)
* [ElasticSearch6.5.4<五> 集群操作](https://blog.csdn.net/qq_25012687/article/details/100915731)
* [ElasticSearch6.5.4<六> ELK](https://blog.csdn.net/qq_25012687/article/details/100972100)
* [ElasticSearch6.5.4<七> ES分布式原理以及工作原理](https://blog.csdn.net/qq_25012687/article/details/101012234)
* [ElasticSearch6.5.4<八> ES常见问题](https://blog.csdn.net/qq_25012687/article/details/101012305)
* [SpringBoot集成ElasticSearch的几种方式](https://blog.csdn.net/qq_25012687/article/details/101050412)