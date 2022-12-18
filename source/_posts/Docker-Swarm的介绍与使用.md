title: Docker Swarm的介绍与使用
author: 几时西风
tags:
  - docker
  - 云原生
categories:
  - docker
date: 2022-12-18 17:29:00
---
# Docker Swarm的介绍与使用

## What

- Docker(容器) Swarm(集群)——简单来说，Docker swarm就是容器集群（包含了集群管理和编排功能），由Docker公司研发并且在Dokcer v1.12版本后自带此服务（早期作为一项独立服务叫做swarmkit，需要单独安装）。

- 集群管理：创建新集群、升级集群的主节点和工作节点、执行节点维护(例如内核升级)和升级运行集群的版本等。

- 容器编排：应用一般由单独容器化的组件（通常称为微服务）组成，这些组件必须按特定顺序在网络中进行组织并照计划运行，这种对多个容器进行组织的流程即称为容器编排（使用Docker Compose实际就是容器编排）。

  For Example:

  standalone-mysql-5.7.yaml

```yaml
version: "2"
services:
  nacos: # 服务名称
    image: nacos/nacos-server:latest # 使用的镜像
    container_name: nacos-standalone-mysql # 容器名称
    env_file: # 环境变量文件加载
      - ../env/nacos-standlone-mysql.env
    volumes: # 宿主与容器文件挂载
      - ./standalone-logs/:/home/nacos/logs
      - ./init.d/custom.properties:/home/nacos/init.d/custom.properties
    ports: # 宿主与容器间端口映射
      - "8848:8848"
      - "9555:9555"
    depends_on: # 依赖的服务
      - mysql
    restart: on-failure # 重启策略
  mysql: # 数据库基础服务
    container_name: mysql
    image: nacos/nacos-mysql:5.7
    env_file:
      - ../env/mysql.env
    volumes:
      - ./mysql:/var/lib/mysql
    ports:
      - "3306:3306"
  prometheus: # prometheus 用于监控nacos服务，为grafana提供基础数据
    container_name: prometheus
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus-standalone.yaml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    depends_on:
      - nacos
    restart: on-failure
  grafana: # 监控面板，配合prometheus使用
    container_name: grafana
    image: grafana/grafana:latest
    ports:
      - 3000:3000
    restart: on-failure

```



## Key Concepts

- 架构

![架构图](\blog\images\pasted-11.png)

从架构图可以看出 Docker Client使用Swarm对 集群(Cluster)进行调度使用。

上图可以看出，Swarm是典型的master-slave结构，通过发现服务来选举manager。manager是中心管理节点，各个node上运行agent接受manager的统一管理，集群会自动通过Raft协议分布式选举出manager节点，无需额外的发现服务支持，避免了单点的瓶颈问题，同时也内置了DNS的负载均衡和对外部负载均衡机制的集成支持。



- node（节点）: 一个节点是docker引擎集群的一个实例，也可以将其视为一个Docker节点。可以在单个物理计算机或云服务器上运行一个或多个节点，但在部署生产集群时通常将节点分布部署在多个物理和云计算机上。
  要将应用程序部署到swarm，需要将服务定义信息提交给 **manager node(管理节点)**。管理节点将称为**tasks(任务)的工作单元**分派给 **worker nodes(工作节点)**。
  管理节点还执行编排和集群管理功能，从而使群集处于需要的状态。管理节点会**选举**一个领导者来执行编排任务。
  工作节点接收并执行从管理器节点分派的任务。默认情况下，管理器节点**也可以作为工作节点运行**，但可以将它们配置为**仅运行管理任务并且仅是管理节点**。代理程序在每个工作节点上运行着一个代理程序，这个代理程序会报告分配给这个工作节点的任务的详细信息。工作节点会通知管理节点分配给它的任务的当前状态，以便管理节点可以维护每个工作节点的状态。

- service(服务)：服务是要执行在节点上的一组task（任务）的定义，它是集群系统的核心结构，同时也是用户与swarm系统进行交互的核心内容。Service有两种运行模式，一种是replicated，swarm会将服务所需的任务按照你定义的数量复制部署在多个节点上；另一种是global，在所有符合运行条件的Node上，都运行一个服务所需的任务。

- task(任务):一个任务是一个docker容器和容器内运行命令的集合。它是swarm任务调度的原子单位，管理节点会根据服务内对任务的定义将任务分派到工作节点，一旦一个任务分配到了一个节点，这个任务就不能再移动都其它节点，他只能再一开始分派到的节点上运行或者直到失败。

- 负载均衡：swarm使用了**入口负载均衡器**来把服务开放给swarm外部，swarm可以自动给服务分配外部端口也可以由用户自定义外部端口，如果用户不定义外部端口的话，swarm会在30000~3276这个范围内随机给服务分配一个端口。外部组件比如云负载均衡器或者nginx之类的组件，可以**从集群内的任意一个节点（无论该节点是否在运行要访问服务的任务）来访问服务**。所有集群内节点都可以路由入口连接到一个运行着任务的实例。另外swarm模式还有一个内部DNS组件来给服务自动分派一个DNS入口。swarm manager使用一个**内部负载均衡器**根据每个服务的DNS地址来分配集群内服务的请求。

  

## Why

为什么使用docker swarm，其实也就是一些功能特点，满足了需求就用：

1. **与Docker Engine集成的集群管理**:使用Docker Engine CLI创建一组Docker引擎，您可以在其中部署应用程序服务。您不需要其他编排软件来创建或管理群集。

2. **节点分散式设计：**Docker Engine不是在部署时处理节点角色之间的差异，而是在运行时处理角色变化。您可以使用Docker Engine部署两种类型的节点，管理节点和工作节点。这意味着您可以很方便地从单个镜像开始构建整个群集。

3. **声明式服务模型：**Docker Engine使用声明式方法来定义应用程序堆栈中各种服务的所需状态。例如，您可以描述由具有消息队列服务和数据库后端的Web前端服务组成的应用程序。

4. **可扩容与缩放容器：**对于每个服务，您可以声明要运行的任务数。当您扩容或缩放时，swarm管理器通过添加或删除任务来自动适应，以保持所需的任务数量来保证集群的可靠状态。

5. **容器容错状态协调：**集群管理节点不断监视群集状态，并协调期望状态与实际状态之间的任何差异。例如，如果设置了一个运行10个副本容器的服务，然后托管其中两个副本容器的工作节点崩溃了，那么管理器将创建两个新副本容器以替换崩溃的副本容器。 swarm管理器将新副本容器分配给正在运行和可用的工作节点上。

6. **多主机网络：**您可以为服务指定承载网络。当swarm管理器初始化或更新应用程序时，它会自动为承载网络上的容器分配地址。

7. **服务发现：**Swarm管理节点为swarm中的每个服务分配唯一的DNS名称，并对运行的容器进行负载均衡。您可以通过嵌入在swarm中的DNS服务器查询在集群中运行的每个容器。

8. **负载均衡：**您可以将服务的端口公开给外部负载平衡器。在内部，swarm允许您指定如何在节点之间分发服务容器。

9. **缺省安全：**集群中的每个节点强制执行TLS验证和加密，以保护其自身与所有其他节点之间的通信。可以选择使用自签名根证书或来自自定义根CA的证书。

10. **滚动更新：**在已经运行期间，您可以增量地应用服务更新到节点。 swarm管理器允许您控制将服务部署到不同节点集之间的延迟。如果出现任何问题，您可以将任务回滚到服务的先前版本。

    

## How

0. **Set up(一些准备)：**
   - 三台linux机器，之间网络互通并且都安装了docker(最好同一版本，安装方法可以参照以前的文章)
   - 三台机器中选一台作为manager节点，记住该机器的ip,确保其它节点可以通过该ip访问到manager节点
   - 开放三台机器的对应端口：2377(TCP，用于集群管理通讯),7946(TCP and UDP，用于节点间通讯),4789(UDP,用于服务负载网络)

1. **Create a swarm(创建集群)：**

   - ssh连接到选定的管理节点上

   - 在管理节点上运行如下命令

     ```bash
     $ docker swarm init --advertise-addr <MANAGER-IP>
     ```

     注意运行命令后的输出，其中docker swarm join命令就是用来将其它工作节点加入集群的命令,最后的docker swarm join-token manager是用于向集群内新增manager节点的，因此要记住token：

     ```ba
     $ docker swarm init --advertise-addr 192.168.99.100
     Swarm initialized: current node (dxn1zf6l61qsb1josjja83ngz) is now a manager.
     
     To add a worker to this swarm, run the following command:
     
         docker swarm join \
         --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
         192.168.99.100:2377
     
     To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
     ```

     **如果不小心忘记了token，可以在管理节点上运行docker swarm join-token worker来查看自动生成的加入工作节点命令**

   - 在管理节点上运行docker info命令可以看到如下输出显示当前集群的状态：

     ```bash
     $ docker info
     
     Containers: 2
     Running: 0
     Paused: 0
     Stopped: 2
       ...snip...
     Swarm: active
       NodeID: dxn1zf6l61qsb1josjja83ngz
       Is Manager: true
       Managers: 1
       Nodes: 1
       ...snip...
     ```

   - 在管理节点上运行docker node ls命令可以看到如下输出显示当前集群节点信息，其中带*的这行表示你目前连接到了这个节点上：

     ```bash
     $ docker node ls
     
     ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
     dxn1zf6l61qsb1josjja83ngz *  manager1  Ready   Active        Leader
     ```

2. **向集群中加入节点：**

   - ssh连接到剩余的两台作为工作节点的机器上

   - 运行之前记录下来的加入工作节点的命令，具体如下：

     ```bash
     $ docker swarm join \
       --token  SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
       192.168.99.100:2377
     
     This node joined a swarm as a worker.
     ```

   - ssh回到管理节点，运行docker node ls命令来查看两个工作节点是否被成功加入集群，具体如下：

     ```bash
     $ docker node ls
     ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
     03g1y59jwfg7cf99w4lt0f662    worker2   Ready   Active
     9j68exjopxe7wfl6yuxml7a7j    worker1   Ready   Active
     dxn1zf6l61qsb1josjja83ngz *  manager1  Ready   Active        Leader
     ```

3. **部署服务到集群：**

   - ssh连接到管理节点上

   - 运行如下命令：

     ```bash
     $ docker service create --replicas 1 --name helloworld alpine ping docker.com
     ```

     docker service create: 用于创建服务的命令

     --name helloworld：服务名称叫做helloworld

     --replicas 1: 服务的任务副本（或者说容器数量）只保留一个副本

     alpine ：镜像名 ，代表 Alpine Linux container

     ping docker.com: 容器内运行的命令

   - 运行docker service ls查看运行中的服务，具体如下图：

     ```bash
     $ docker service ls
     
     ID            NAME        SCALE  IMAGE   COMMAND
     9uk4639qpg7n  helloworld  1/1    alpine  ping docker.com
     ```



4. **查看服务**：

   查看服务详细信息：

   ```bash
   [manager1]$ docker service inspect --pretty helloworld
   
   ID:		9uk4639qpg7npwf3fn2aasksr
   Name:		helloworld
   Service Mode:	REPLICATED
    Replicas:		1
   Placement:
   UpdateConfig:
    Parallelism:	1
   ContainerSpec:
    Image:		alpine
    Args:	ping docker.com
   Resources:
   Endpoint Mode:  vip
   ```

   查看服务运行节点：

   ```bash
   [manager1]$ docker service ps helloworld
   
   NAME                                    IMAGE   NODE     DESIRED STATE  CURRENT STATE           ERROR               PORTS
   helloworld.1.8p1vev3fq5zm0mi8g0as41w35  alpine  worker2  Running        Running 3 minutes
   ```

   在服务运行节点上使用docker ps 查看运行中的容器：

   ```bash
   [worker2]$ docker ps
   
   CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
   e609dde94e47        alpine:latest       "ping docker.com"   3 minutes ago       Up 3 minutes                            helloworld.1.8p1vev3fq5zm0mi8g0as41w35
   ```

5. **扩容服务：**

   - 使用 docker service scale <SERVICE-ID>=<NUMBER-OF-TASKS> 来扩容，具体如下：

     ```bash
     $ docker service scale helloworld=5
     
     helloworld scaled to 5
     ```

     查看扩容后的结果：

   - ```bash
     $ docker service ps helloworld
     
     NAME                                    IMAGE   NODE      DESIRED STATE  CURRENT STATE
     helloworld.1.8p1vev3fq5zm0mi8g0as41w35  alpine  worker2   Running        Running 7 minutes
     helloworld.2.c7a7tcdq5s0uk3qr88mf8xco6  alpine  worker1   Running        Running 24 seconds
     helloworld.3.6crl09vdcalvtfehfh69ogfb1  alpine  worker1   Running        Running 24 seconds
     helloworld.4.auky6trawmdlcne8ad8phb0f1  alpine  manager1  Running        Running 24 seconds
     helloworld.5.ba19kca06l18zujfwxyc5lkyn  alpine  worker2   Running        Running 24 seconds
     ```

6. **删除服务:**

   使用docker service rm 即可删除服务，具体如下

   ```bash
   $ docker service rm helloworld
   
   helloworld
   ```

7. **滚动更新服务**：

   - 以redis 为例，先创建一个有三个副本的redis服务：

     ```bash
     $ docker service create \
       --replicas 3 \
       --name redis \
       --update-delay 10s \
       redis:3.0.6
     
     0u6a4s31ybk7yw2wyvtikmu50
     ```

     注意通过--update-delay声明了服务更新的延迟时间为10秒，也可以通过 单独或组合使用10s,10m,10h来表示不同的延迟，比如10m30s代表十分三十秒

   - 开始更新，将redis版本进行升级：

     ```bash
     $ docker service update --image redis:3.0.7 redis
     redis
     ```

   - 升级过程中可以使用docker service inspect 来查看升级状况,如下图是升级失败：

     ```bash
     docker service inspect --pretty redis
     
     ID:             0u6a4s31ybk7yw2wyvtikmu50
     Name:           redis
     ...snip...
     Update status:
      State:      paused
      Started:    11 seconds ago
      Message:    update paused due to failure or early termination of task 9p7ith557h8ndf0ui9s0q951b
     ...snip...
     ```

     升级失败了重新运行升级命令docker service update redis即可

   - 升级过程中可以使用docker service ps来查看升级状况，有的会是3.0.6 running,有的是3.0.7running,一旦升级完毕则全部都是3.0.7 running如下图 ：

     ```bash
     $ docker service ps redis
     
     NAME                                   IMAGE        NODE       DESIRED STATE  CURRENT STATE            ERROR
     redis.1.dos1zffgeofhagnve8w864fco      redis:3.0.7  worker1    Running        Running 37 seconds
      \_ redis.1.88rdo6pa52ki8oqx6dogf04fh  redis:3.0.6  worker2    Shutdown       Shutdown 56 seconds ago
     redis.2.9l3i4j85517skba5o7tn5m8g0      redis:3.0.7  worker2    Running        Running About a minute
      \_ redis.2.66k185wilg8ele7ntu8f6nj6i  redis:3.0.6  worker1    Shutdown       Shutdown 2 minutes ago
     redis.3.egiuiqpzrdbxks3wxgn8qib1g      redis:3.0.7  worker1    Running        Running 48 seconds
      \_ redis.3.ctzktfddb2tepkr45qcmqln04  redis:3.0.6  mmanager1  Shutdown       Shutdown 2 minutes ago
     ```

8. **排空一个节点**：

   某些情况下要把某个节点下线或者关机时，需要先将节点上的tasks转移，这个过程就叫做排空,如下图

   ```bash
   docker node update --availability drain worker1
   
   worker1
   ```

   再运行docker service ps redis，可以看到worker1上的容器都停止了并在worker2上开了新的容器：

   ```bash
   $ docker service ps redis
   
   NAME                                    IMAGE        NODE      DESIRED STATE  CURRENT STATE           ERROR
   redis.1.7q92v0nr1hcgts2amcjyqg3pq       redis:3.0.6  manager1  Running        Running 4 minutes
   redis.2.b4hovzed7id8irg1to42egue8       redis:3.0.6  worker2   Running        Running About a minute
    \_ redis.2.7h2l8h3q3wqy5f66hlv9ddmi6   redis:3.0.6  worker1   Shutdown       Shutdown 2 minutes ago
   redis.3.9bg7cezvedmkgg6c8yzvbhwsd       redis:3.0.6  worker2   Running        Running 4 minutes
   ```

   当你需要使排空的节点可用，运行以下命令即可，之后该节点就可以正常接受新的任务了：

   ```bash
   $ docker node update --availability active worker1
   
   worker1
   ```

9. **暴露服务端口**

   - 如docker中使用-p 8080:80一样，也可以使用同样的方式暴露服务端口，但下面的命令更为清晰：

   ```bash
   $ docker service create \
     --name my-web \
     --publish published=8080,target=80 \
     --replicas 2 \
     nginx
   ```

   **之后你访问三台机器的 8080端口，都可以访问到nginx，即使那台机器上没有运行nginx的任务**——这个特性被叫做routing mesh(路由混合)

   - 需要更新服务端口的话可以使用如下命令

   ```bash
   $ docker service update \
     --publish-add published=<PUBLISHED-PORT>,target=<CONTAINER-PORT> \
     <SERVICE>
   ```

   - 默认暴露的是TCP端口，如果要暴露UDP端口需要如下设置

     ```bash
     $ docker service create --name dns-cache \
       --publish published=53,target=53 \
       --publish published=53,target=53,protocol=udp \
       dns-cache
       或者
     $ docker service create --name dns-cache \
       -p 53:53 \
       -p 53:53/udp \
       dns-cache
     ```

   - 跳过路由混合：

     如果不想使用以上“访问三台机器的 8080端口，都可以访问到nginx，即使那台机器上没有运行nginx的任务”这个特性，可以通过对端口声明mode的方式进行控制，如下所示：

     ```bash
     $ docker service create --name dns-cache \
       --publish published=53,target=53,protocol=udp,mode=host \
       --mode global \
       dns-cache
     ```

     这种情况下，访问机器的53端口，就要求那台机器上必须要有运行中的task，并且访问的就是那个task

10. **使用服务承载网络**：

    如果集群中多个服务需要互相通讯，就可以使用承载网络，如下所示:

    - 先创建网络

      ```bash
      $ docker network create --driver overlay my-network
      ```

    - 创建服务时声明其所在网络

      ```bash
      $ docker service create \
        --replicas 3 \
        --network my-network \
        --name my-web \
        nginx
      ```

    - 将已存在的服务加入到某个网络

      ```bash
      docker service update --network-add my-network my-web
      ```



## Docker stack

**Docker Compose缺点是不能在分布式多机器上使用；Docker swarm缺点是不能同时编排多个服务，所以才有了Docker Stack，可以在分布式多机器上同时编排多个服务。**stack就是一组共用一个承载网络的服务的集合。



例子：

0. Setup

- service1:

```java
@Slf4j
@RestController
public class HelloRest {
    @GetMapping("/service1/getHello")
    public String getHello(){
        log.info("service1!!!");
        return "hello from service1";
    }
}
```

```dockerfile
FROM openjdk:8
EXPOSE 8080
ADD  target/service1-0.0.1-SNAPSHOT.jar /demo.jar
ENTRYPOINT ["java", "-jar", "demo.jar"]
```



- service2:

```java
@Slf4j
@RestController
public class HelloRest {
    @GetMapping("/service2/getHello")
    public String getHello(){
        log.info("service2!!!");
        return "hello from service2";
    }
}
```

```dockerfile
FROM openjdk:8
EXPOSE 8081
ADD  target/service2-0.0.1-SNAPSHOT.jar /demo.jar
ENTRYPOINT ["java", "-jar", "demo.jar"]
```



1. 分别打包镜像

   ```bash
   docker build -t service1:V1 .
   docker build -t service2:V1 .
   ```

2. 编写docker-compose.yml文件

   ```yaml
   version: "3.9"
   services:
     service1:
       image: "zhangjie47/service1:V1"
       deploy:
           replicas: 2
       ports:
         - "8080:8080"
   
     service2:
       image: "zhangjie47/service2:V1"
       deploy:
           replicas: 3
       ports:
         - "8081:8081"
   ```

3. stack 部署

   ```bash
   # myapps是stack的自定义名称，使用具体路径的compose配置文件进行部署
   docker stack deploy myapps --compose-file=docker-compose.yml
   或者
   docker stack deploy myapps -c docker-compose.yml
   Creating network myapps_default
   Creating service myapps_service1
   Creating service myapps_service2
   ```

4. 查看docker stack

   ```bash
   # 查看所有stack的信息
   docker stack ls
   NAME      SERVICES   ORCHESTRATOR
   myapps    2          Swarm
   
   # 查看某个stack中的所有任务信息
   docker stack ps myapps
   ID             NAME                IMAGE                    NODE      DESIRED STATE   CURRENT STATE
   tvvujrf3qcr1   myapps_service1.1   zhangjie47/service1:V1   node1     Running         Running 46 seconds ago       
   igjeydmmvzzm   myapps_service1.2   zhangjie47/service1:V1   manager   Running         Running 46 seconds ago       
   7p5c96eplwl3   myapps_service2.1   zhangjie47/service2:V1   node1     Running         Running 34 seconds ago       
   7shglsajip5d   myapps_service2.2   zhangjie47/service2:V1   manager   Running         Running 39 seconds ago       
   upo0mr7j9tn1   myapps_service2.3   zhangjie47/service2:V1   node2     Running         Preparing 41 seconds ago  
   
   # 查看某个stack中的所有服务信息
   docker stack services myapps
   ID             NAME              MODE         REPLICAS   IMAGE                    PORTS
   icz3kjn0skb3   myapps_service1   replicated   2/2        masonzhang/service1:V1   *:8080->8080/tcp
   myuzlwnrxag4   myapps_service2   replicated   3/3        masonzhang/service2:V1   *:8081->8081/tcp
   
   # 查看具体服务信息等请参照上文
   
   # 移除stack
   docker stack rm myapps
   Removing service myapps_service1
   Removing service myapps_service2
   Removing network myapps_default
   
   ```

   