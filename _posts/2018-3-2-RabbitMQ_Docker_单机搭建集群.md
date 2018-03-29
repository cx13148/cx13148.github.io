# Rabbit MQ + Docker 单机搭建集群

搭建环境： centos7 + java jdk

## 安装Docker

1. 删除旧版本

    ```
    $ sudo yum remove docker \
                      docker-client \
                      docker-client-latest \
                      docker-common \
                      docker-latest \
                      docker-latest-logrotate \
                      docker-logrotate \
                      docker-selinux \
                      docker-engine-selinux \
                      docker-engine
    ```

2. 用yum安装新版本

- 安装依赖包. `yum-utils` 提供 `yum-config-manager` 功能, `devicemapper storage driver`则需要 `device-mapper-persistent-data` 以及 `lvm2`。

    ```
    $ sudo yum install -y yum-utils \
      device-mapper-persistent-data \
      lvm2
    ```

- 用以下命令添加稳定源
    ```
    $ sudo yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo
    ```

- 安装Docker CE
    ```
    $ sudo yum install docker-ce
    ```

- 检查安装
    ```
    $ sudo docker run hello-world
    ```

## Docker 常用命令

1. docker服务

    ```
    启停重启服务
    service docker start(stop restart)
    开机启动
    systemctl enable docker.service
    ```
    
2. docker镜像、容器
    
    ```
    显示docker镜像
    docker images
    删除docker镜像
    docker rmi -f [imageid]
    显示docker容器
    docker ps
    停止docker容器
    docker stop [containerId]
    删除docker容器
    docker rmi -f [containerId]
    ```
    
3. 启动容器
    
    ```
    docker run [imageId]
    -d 后台运行
    -p 默认桥接网络模式，映射端口
    -P 自动添加映射
    --net=host 网络主机模式
    -v 挂载容器和主机间的路径
    --restart=always 随着docker服务开机启动
    ```
    
4. 上传下载  

    ```
    下载
    docker pull 192.168.1.106:5000/ht/tomcat:8
    打版本
    docker tag tomcat:8 192.168.1.106:5000/ht/tomcat:8
    上传
    docker push 192.168.1.106:5000/ht/tomcat:8
    ```
    
5. 其它命令  
    
    ```
    进入容器
    docker exec [containerId] -it bash
    执行命令
    docker exec [containerId] -it [command]
    容器控制台日志
    docker logs -f [containerId]
    ```
        
## 设置rabbitmq cluster  

Clustering Ref: http://www.rabbitmq.com/clustering.html  
rabbitmqctl Ref: http://www.rabbitmq.com/rabbitmqctl.8.html  
- **Erlang Cookie**  
    因为RabbitMQ采用Erlang开发，Erlang通过认证Erlang cookie的方式来允许节点间相互通信。所以RabbitMQ需要相互通信的节点必须使用相同的Erlang cookie。
- **在docker中添加内部网络**  
   
    ```
    docker network create mqnet
    ```
    
- **添加rabbitmq容器**  
    设置同样的erlang cookie，加入之前创建的docker网络， 实际上可能还需要配置DNS
    
    ```
    docker run -d\
    --name=rabbitmq1\
    -P\
    -e RABBITMQ_NODENAME=rabbitmq1\
    -e RABBITMQ_DEFAULT_USER=user\
    -e RABBITMQ_DEFAULT_PASS=password\
    -e RABBITMQ_ERLANG_COOKIE='ABCDEFGHIJKASAQESDAS'\
    -h rabbitmq1\
    --net=mqnet rabbitmq:3.7.4-management
    ```
    
- **加入集群**  
    默认情况下，rabbitmq集群会扩展scalebility,而非availability,
    By default, contents of a queue within a RabbitMQ cluster are located on a single node (the node on which the queue was declared). This is in contrast to exchanges and bindings, which can always be considered to be on all nodes. Queues can optionally be made mirrored across multiple nodes. Each mirrored queue consists of one master and one or more mirrors, with the oldest mirror being promoted to the new master if the old master disappears for any reason.

    ```
    #磁盘节点
    docker exec rabbitmq2 bash -c \
    "rabbitmqctl stop_app && \
    rabbitmqctl reset && \
    rabbitmqctl join_cluster rabbitmq1@rabbitmq1 && \
    rabbitmqctl start_app"
    #内存节点
    docker exec rabbitmq3 bash -c \
    "rabbitmqctl stop_app && \
    rabbitmqctl reset && \
    rabbitmqctl join_cluster --ram rabbitmq1@rabbitmq1 && \
    rabbitmqctl start_app"
    ```
    
- **设置镜像队列**  
    Ref: http://www.rabbitmq.com/ha.html
    ha-mode : excatly,all,nodes  
    ha-params : excatly 模式下可以设置镜像个数
    下面的例子为 设置所有队列为镜像队列，除了amq开头的交换器绑定队列。
   
    ```
    docker exec rabbitmq1 rabbitmqctl set_policy HA '^(?!amq\.).*' '{"ha-mode": "all"}'
    ```
    
- **退出集群**  
    
    ```
    docker exec rabbitmq3 bash -c \
    "rabbitmqctl stop_app && \
    rabbitmqctl reset && \
    rabbitmqctl start_app"
    ```
    
    Notice:  
        When the entire cluster is brought down, **the last node to go down must be the first node to be brought online.** If this doesn't happen, the nodes will wait 30 seconds for the last disc node to come back online, and fail afterwards. If the last node to go offline cannot be brought back up, it can be removed from the cluster using the `forget_cluster_node` command - consult the rabbitmqctl manpage for more information.
        If all cluster nodes stop in a simultaneous and uncontrolled manner (for example with a power cut) you can be left with a situation in which all nodes think that some other node stopped after them. In this case you can use the force_boot command on one node to make it bootable again - consult the rabbitmqctl manpage for more information.  
