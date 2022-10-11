# 教程

本教程演示如何使用 Debezium 监控 MySQL 数据库。随着数据库中的数据发生变化，您将看到生成的事件流。

在本教程中，您将启动 Debezium 服务，使用简单的示例数据库运行 MySQL 数据库服务器，并使用 Debezium 监控数据库的更改。

先决条件

- Docker 已安装并正在运行。

  本教程使用 Docker 和 Debezium 容器镜像来运行所需的服务。您应该使用最新版本的 Docker。有关更多信息，请参阅[Docker 引擎安装文档](https://docs.docker.com/engine/installation/)。

|      | 此示例也可以使用 Podman 运行。有关更多信息，请参阅[Podman](https://podman.io/)。 |
| ---- | ------------------------------------------------------------ |

## Debezium 简介

Debezium 是一个分布式平台，可将现有数据库中的信息转换为事件流，使应用程序能够检测并立即响应数据库中的行级更改。

[Debezium 构建在Apache Kafka](http://kafka.apache.org/)之上，并提供了一组与[Kafka Connect](https://kafka.apache.org/documentation.html#connect)兼容的连接器。每个连接器都与特定的数据库管理系统 (DBMS) 一起使用。连接器通过在发生更改时检测更改并将每个更改事件的记录流式传输到 Kafka 主题来记录 DBMS 中数据更改的历史记录。然后，消费应用程序可以从 Kafka 主题中读取结果事件记录。

通过利用 Kafka 可靠的流媒体平台，Debezium 使应用程序能够正确、完整地使用数据库中发生的更改。即使您的应用程序意外停止或失去连接，它也不会错过中断期间发生的事件。应用程序重新启动后，它会从停止的位置继续读取主题。

下面的教程向您展示了如何通过简单的配置来部署和使用[Debezium MySQL 连接器。](https://debezium.io/documentation/reference/stable/connectors/mysql.html#debezium-connector-for-mysql)有关部署和使用 Debezium 连接器的更多信息，请参阅连接器文档。

其他资源

- [用于 Cassandra 的 Debezium 连接器](https://debezium.io/documentation/reference/stable/connectors/cassandra.html#debezium-connector-for-cassandra)
- [用于 Db2 的 Debezium 连接器](https://debezium.io/documentation/reference/stable/connectors/db2.html#debezium-connector-for-db2)
- [用于 MongoDB 的 Debezium 连接器](https://debezium.io/documentation/reference/stable/connectors/mongodb.html#debezium-connector-for-mongodb)
- [MySQL 的 Debezium 连接器](https://debezium.io/documentation/reference/stable/connectors/mysql.html#debezium-connector-for-mysql)
- [用于 Oracle 数据库的 Debezium 连接器](https://debezium.io/documentation/reference/stable/connectors/oracle.html#debezium-connector-for-oracle)
- [PostgreSQL 的 Debezium 连接器](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#debezium-connector-for-postgresql)
- [SQL Server 的 Debezium 连接器](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#debezium-connector-for-sql-server)
- [用于 Vitess 的 Debezium 连接器](https://debezium.io/documentation/reference/stable/connectors/vitess.html#debezium-connector-for-vitess)

## 启动服务

使用 Debezium 需要三个独立的服务： [ZooKeeper](http://zookeeper.apache.org/)、Kafka 和 Debezium 连接器服务。[在本教程中，您将使用Docker](http://docker.com/)和[Debezium 容器镜像](https://quay.io/organization/debezium)为每个服务启动单实例。

要启动本教程所需的服务，您必须：

- [启动 Zookeeper](https://debezium.io/documentation/reference/stable/tutorial.html#starting-zookeeper)
- [启动Kafka](https://debezium.io/documentation/reference/stable/tutorial.html#starting-kafka)
- [启动 MySQL 数据库](https://debezium.io/documentation/reference/stable/tutorial.html#starting-mysql-database)
- [启动 MySQL 命令行客户端](https://debezium.io/documentation/reference/stable/tutorial.html#starting-mysql-command-line-client)
- [启动 Kafka 连接](https://debezium.io/documentation/reference/stable/tutorial.html#starting-kafka-connect)

### 使用 Docker 运行 Debezium 的注意事项

本教程使用[Docker](http://docker.com/)和[Debezium 容器镜像](https://hub.docker.com/u/debezium/)来运行 ZooKeeper、Kafka、Debezium 和 MySQL 服务。在单独的容器中运行每个服务可以简化设置，以便您可以看到 Debezium 的运行情况。

|      | 在生产环境中，您将运行集群以提供性能、可靠性、复制和容错。通常，您可以在[OpenShift](https://www.openshift.com/)或[Kubernetes](http://kubernetes.io/)等平台上部署这些服务，这些平台管理在多个主机和机器上运行的多个 Docker 容器，或者您可以安装在专用硬件上。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

使用 Docker 运行 Debezium 时，您应该注意以下注意事项：

- ZooKeeper 和 Kafka 的容器是临时的。

  ZooKeeper 和 Kafka 通常会将它们的本地数据存储在容器内，这需要您将主机上的目录作为卷挂载。这样，当容器停止时，保留的数据仍然存在。但是，本教程会跳过此设置 - 当容器停止时，所有持久数据都会丢失。这样，当您完成本教程时，清理工作就很简单了。

  |      | 有关存储持久数据的更多信息，请参阅[容器映像](https://quay.io/organization/debezium)的文档。 |
  | ---- | ------------------------------------------------------------ |
  |      |                                                              |

- 本教程要求您在不同的容器中运行每个服务。

  为避免混淆，您将在单独的终端中在前台运行每个容器。这样，容器的所有输出都将显示在用于运行它的终端中。

  |      | Docker 还允许您以*分离*模式（使用`-d`选项）运行容器，其中启动容器并`docker`立即返回命令。但是，分离模式容器不会在终端中显示它们的输出。要查看输出，您需要使用该命令。有关更多信息，请参阅 Docker 文档。`docker logs --follow --name *<container-name>*` |
  | ---- | ------------------------------------------------------------ |
  |      |                                                              |

### 启动 Zookeeper

ZooKeeper 是您必须启动的第一个服务。

步骤

1. 打开一个终端并使用它在容器中启动 ZooKeeper。

   此命令使用 1.9 版`quay.io/debezium/zookeeper`映像运行一个新容器：

   ```shell
   $ docker run -it --rm --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 quay.io/debezium/zookeeper:1.9
   ```

   - `-it`

     容器是交互式的，这意味着终端的标准输入和输出附加到容器上。

   - `--rm`

     容器停止时将被移除。

   - `--name zookeeper`

     容器的名称。

   - `-p 2181:2181 -p 2888:2888 -p 3888:3888`

     将容器的三个端口映射到 Docker 主机上的相同端口。这使得其他容器（以及容器外的应用程序）能够与 ZooKeeper 进行通信。

|      | 如果您使用 Podman，请运行以下命令：s`$ sudo podman pod create --name=dbz --publish "9092,3306,8083" $ sudo podman run -it --rm --name zookeeper --pod dbz quay.io/debezium/zookeeper:1.9` |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

1. 验证 ZooKeeper 是否已启动并正在侦听 port `2181`。

   您应该会看到类似于以下内容的输出：

   ```shell
   Starting up in standalone mode
   ZooKeeper JMX enabled by default
   Using config: /zookeeper/conf/zoo.cfg
   2017-09-21 07:15:55,417 - INFO  [main:QuorumPeerConfig@134] - Reading configuration from: /zookeeper/conf/zoo.cfg
   2017-09-21 07:15:55,419 - INFO  [main:DatadirCleanupManager@78] - autopurge.snapRetainCount set to 3
   2017-09-21 07:15:55,419 - INFO  [main:DatadirCleanupManager@79] - autopurge.purgeInterval set to 1
   ...
   port 0.0.0.0/0.0.0.0:2181  
   ```

   |      | 此行表明 ZooKeeper 已准备好并正在侦听端口 2181。当 ZooKeeper 生成它时，终端将继续显示附加输出。 |
   | ---- | ------------------------------------------------------------ |
   |      |                                                              |

### 启动Kafka

启动 ZooKeeper 后，您可以在新容器中启动 Kafka。

|      | Debezium 1.9.6.Final 已经针对多个版本的 Kafka Connect 进行了测试。请参考[Debezium 测试矩阵](https://debezium.io/releases)来确定 Debezium 和 Kafka Connect 之间的兼容性。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

步骤

1. 打开一个新终端并使用它在容器中启动 Kafka。

   此命令使用 1.9 版`quay.io/debezium/kafka`映像运行一个新容器：

   ```shell
   $ docker run -it --rm --name kafka -p 9092:9092 --link zookeeper:zookeeper quay.io/debezium/kafka:1.9
   ```

   - `-it`

     容器是交互式的，这意味着终端的标准输入和输出附加到容器上。

   - `--rm`

     容器停止时将被移除。

   - `--name kafka`

     容器的名称。

   - `-p 9092:9092`

     将容器中的端口映射`9092`到 Docker 主机上的相同端口，以便容器外的应用程序可以与 Kafka 通信。

   - `--link zookeeper:zookeeper`

     告诉容器它可以`zookeeper`在同一个 Docker 主机上运行的容器中找到 ZooKeeper。

   |      | 如果您使用 Podman，请运行以下命令：`$ sudo podman run -it --rm --name kafka --pod dbz quay.io/debezium/kafka:1.9` |
   | ---- | ------------------------------------------------------------ |
   |      |                                                              |

   |      | 在本教程中，您将始终从 Docker 容器中连接到 Kafka。这些容器中的任何一个都可以`kafka`通过链接到它来与容器通信。如果您需要从Docker 容器*外部*连接到 Kafka ，则必须设置`-e`选项以通过 Docker 主机通告 Kafka 地址（`-e ADVERTISED_HOST_NAME=`后跟 Docker 主机的 IP 地址或可解析的主机名）。 |
   | ---- | ------------------------------------------------------------ |
   |      |                                                              |

2. 验证 Kafka 是否已启动。

   您应该会看到类似于以下内容的输出：

   ```shell
   ...
   2017-09-21 07:16:59,085 - INFO  [main-EventThread:ZkClient@713] - zookeeper state changed (SyncConnected)
   2017-09-21 07:16:59,218 - INFO  [main:Logging$class@70] - Cluster ID = LPtcBFxzRvOzDSXhc6AamA
   ...
   2017-09-21 07:16:59,649 - INFO  [main:Logging$class@70] - [Kafka Server 1], started  
   ```

   |      | Kafka 代理已成功启动并准备好进行客户端连接。当 Kafka 生成它时，终端将继续显示额外的输出。 |
   | ---- | ------------------------------------------------------------ |

### 启动 MySQL 数据库

至此，您已经启动了 ZooKeeper 和 Kafka，但您仍然需要一个数据库服务器，Debezium 可以从中捕获更改。在此过程中，您将使用示例数据库启动 MySQL 服务器。

步骤

1. 打开一个新终端，并使用它来启动一个新容器，该容器运行一个预先配置有`inventory`数据库的 MySQL 数据库服务器。

   [此命令使用基于](https://github.com/debezium/docker-images/blob/main/examples/mysql/1.9/Dockerfile)[mysql:8.0](https://hub.docker.com/r/_/mysql/)映像的1.9 版`quay.io/debezium/example-mysql`映像运行新容器。它还定义并填充了一个示例数据库：`inventory`

   ```shell
   $ docker run -it --rm --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw quay.io/debezium/example-mysql:1.9
   ```

   - `-it`

     容器是交互式的，这意味着终端的标准输入和输出附加到容器上。

   - `--rm`

     容器停止时将被移除。

   - `--name mysql`

     容器的名称。

   - `-p 3306:3306`

     将容器中的端口（默认 MySQL 端口）映射`3306`到 Docker 主机上的相同端口，以便容器外的应用程序可以连接到数据库服务器。

   - `-e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw`

     创建具有 Debezium MySQL 连接器所需的最低权限的用户和密码。

|      | 如果您使用 Podman，请运行以下命令：`$ sudo podman run -it --rm --name mysql --pod dbz -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw quay.io/debezium/example-mysql:1.9` |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

1. 验证 MySQL 服务器是否启动。

   MySQL 服务器会随着配置的修改而启动和停止几次。您应该会看到类似于以下内容的输出：

   ```shell
   ...
   [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.27'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server - GPL.
   [System] [MY-011323] [Server] X Plugin ready for connections. Bind-address: '::' port: 33060, socket: /var/run/mysqld/mysqlx.sock
   ```

### 启动 MySQL 命令行客户端

启动 MySQL 后，启动 MySQL 命令行客户端，以便访问示例`inventory`数据库。

步骤

1. 打开一个新终端，并使用它在容器中启动 MySQL 命令行客户端。

   [此命令使用mysql:8.0](https://hub.docker.com/r/_/mysql/)映像运行一个新容器，并定义一个 shell 命令以使用正确的选项运行 MySQL 命令行客户端：

   ```shell
   $ docker run -it --rm --name mysqlterm --link mysql --rm mysql:8.0 sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
   ```

   - `-it`

     容器是交互式的，这意味着终端的标准输入和输出附加到容器上。

   - `--rm`

     容器停止时将被移除。

   - `--name mysqlterm`

     容器的名称。

   - `--link mysql`

     将容器链接到`mysql`容器。

|      | 如果您使用 Podman，请运行以下命令：`$ sudo podman run -it --rm --name mysqlterm --pod dbz --rm mysql:5.7 sh -c 'exec mysql -h 0.0.0.0 -uroot -pdebezium'` |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

1. 验证 MySQL 命令行客户端是否已启动。

   您应该会看到类似于以下内容的输出：

   ```mysql
   mysql: [Warning] Using a password on the command line interface can be insecure.
   Welcome to the MySQL monitor.  Commands end with ; or \g.
   Your MySQL connection id is 9
   Server version: 8.0.27 MySQL Community Server - GPL
   
   Copyright (c) 2000, 2021, Oracle and/or its affiliates.
   
   Oracle is a registered trademark of Oracle Corporation and/or its
   affiliates. Other names may be trademarks of their respective
   owners.
   
   Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
   
   mysql>
   ```

2. 在`mysql>`命令提示符下，切换到库存数据库：

   ```sql
   mysql> use inventory;
   ```

3. 列出数据库中的表：

   ```sql
   mysql> show tables;
   +---------------------+
   | Tables_in_inventory |
   +---------------------+
   | addresses           |
   | customers           |
   | geom                |
   | orders              |
   | products            |
   | products_on_hand    |
   +---------------------+
   6 rows in set (0.00 sec)
   ```

4. 使用 MySQL 命令行客户端浏览数据库，查看数据库中预加载的数据。

   例如：

   ```sql
   mysql> SELECT * FROM customers;
   +------+------------+-----------+-----------------------+
   | id   | first_name | last_name | email                 |
   +------+------------+-----------+-----------------------+
   | 1001 | Sally      | Thomas    | sally.thomas@acme.com |
   | 1002 | George     | Bailey    | gbailey@foobar.com    |
   | 1003 | Edward     | Walker    | ed@walker.com         |
   | 1004 | Anne       | Kretchmar | annek@noanswer.org    |
   +------+------------+-----------+-----------------------+
   4 rows in set (0.00 sec)
   ```

### 启动 Kafka 连接

`inventory`启动 MySQL 并使用 MySQL 命令行客户端连接到数据库后，您将启动 Kafka Connect 服务。该服务公开了一个 REST API 来管理 Debezium MySQL 连接器。

程序

1. 打开一个新终端，并使用它在容器中启动 Kafka Connect 服务。

   此命令使用 1.9 版本的`quay.io/debezium/connect`映像运行一个新容器：

   ```shell
   $ docker run -it --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link kafka:kafka --link mysql:mysql quay.io/debezium/connect:1.9
   ```

   - `-it`

     容器是交互式的，这意味着终端的标准输入和输出附加到容器上。

   - `--rm`

     容器停止时将被移除。

   - `--name connect`

     容器的名称。

   - `-p 8083:8083`

     将容器中的端口映射`8083`到 Docker 主机上的相同端口。这使得容器外的应用程序能够使用 Kafka Connect 的 REST API 来设置和管理新的容器实例。

   - `-e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses`

     设置 Debezium 映像所需的环境变量。

   - `--link kafka:kafka --link mysql:mysql`

     将容器链接到运行 Kafka 和 MySQL 服务器的容器。

|      | 如果您使用 Podman，请运行以下命令：`$ sudo podman run -it --rm --name connect --pod dbz -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses quay.io/debezium/connect:1.9` |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

|      | 如果您提供`--hostname`命令选项，则 Kafka Connect REST API 将**不会**`localhost`在接口上侦听。当暴露 REST 端口时，这可能会导致问题。如果这是一个问题，则设置环境变量`REST_HOST_NAME=0.0.0.0`，以确保可以从所有接口访问 REST API。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

1. 验证 Kafka Connect 是否已启动并准备好接受连接。

   您应该会看到类似于以下内容的输出：

   ```shell
   ...
   2020-02-06 15:48:33,939 INFO   ||  Kafka version: 3.0.0   [org.apache.kafka.common.utils.AppInfoParser]
   ...
   2020-02-06 15:48:34,485 INFO   ||  [Worker clientId=connect-1, groupId=1] Starting connectors and tasks using config offset -1   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
   2020-02-06 15:48:34,485 INFO   ||  [Worker clientId=connect-1, groupId=1] Finished starting connectors and tasks   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
   ```

2. 使用 Kafka Connect REST API 检查 Kafka Connect 服务的状态。

   Kafka Connect 公开了一个 REST API 来管理 Debezium 连接器。要与 Kafka Connect 服务通信，您可以使用该命令将 API 请求发送到 Docker 主机的 8083 端口（您在启动 Kafka Connect 时`curl`映射到容器中的 8083 端口）。`connect`

   |      | 这些命令使用`localhost`. 如果您使用的是非本地 Docker 平台（例如 Docker Toolbox），请替换`localhost`为您的 Docker 主机的 IP 地址。 |
   | ---- | ------------------------------------------------------------ |
   |      |                                                              |

   1. 打开一个新终端并检查 Kafka Connect 服务的状态：

      ```none
      $ curl -H "Accept:application/json" localhost:8083/
      {"version":"3.2.0","commit":"cb8625948210849f"}  
      ```

      |      | 响应显示 Kafka Connect 版本 3.2.0 正在运行。 |
      | ---- | -------------------------------------------- |
      |      |                                              |

   2. 检查向 Kafka Connect 注册的连接器列表：

      ```none
      $ curl -H "Accept:application/json" localhost:8083/connectors/
      []  
      ```

      |      | 目前没有向 Kafka Connect 注册连接器。 |
      | ---- | ------------------------------------- |
      |      |                                       |

## 部署 MySQL 连接器

启动 Debezium 和 MySQL 服务后，您就可以部署 Debezium MySQL 连接器，以便它可以开始监视示例 MySQL 数据库 ( `inventory`)。

此时，您正在运行 Debezium 服务、带有示例`inventory`数据库的 MySQL 数据库服务器以及连接到数据库的 MySQL 命令行客户端。要部署 MySQL 连接器，您必须：

- [注册 MySQL 连接器以监控`inventory`数据库](https://debezium.io/documentation/reference/stable/tutorial.html#registering-connector-monitor-inventory-database)

  注册连接器后，它将开始监视数据库服务器`binlog`，并为每一行更改生成更改事件。

- [观看 MySQL 连接器启动](https://debezium.io/documentation/reference/stable/tutorial.html#watching-connector-start-up)

  在连接器启动时查看 Kafka Connect 日志输出有助于您更好地了解它必须完成的每项任务，然后它才能开始监控`binlog`.

### 注册连接器以监控`inventory`数据库

通过注册 Debezium MySQL 连接器，连接器将开始监控 MySQL 数据库服务器的`binlog`. 记录数据库的`binlog`所有事务（例如对单个行的更改和对模式的更改）。当数据库中的一行发生更改时，Debezium 会生成一个更改事件。

|      | 在生产环境中，您通常会使用 Kafka 工具手动创建必要的主题，包括指定副本的数量，或者使用 Kafka Connect 机制来自定义[自动创建的](https://debezium.io/documentation/reference/stable/configuration/topic-auto-create-config.html)主题的设置。但是，对于本教程，Kafka 配置为仅使用一个副本自动创建主题。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

程序

1. 查看您将注册的 Debezium MySQL 连接器的配置。

   在注册连接器之前，您应该熟悉它的配置。在下一步中，您将注册以下连接器：

   ```json
   {
     "name": "inventory-connector",  
     "config": {  
       "connector.class": "io.debezium.connector.mysql.MySqlConnector",
       "tasks.max": "1",  
       "database.hostname": "mysql",  
       "database.port": "3306",
       "database.user": "debezium",
       "database.password": "dbz",
       "database.server.id": "184054",  
       "database.server.name": "dbserver1",  
       "database.include.list": "inventory",  
       "database.history.kafka.bootstrap.servers": "kafka:9092",  
       "database.history.kafka.topic": "schema-changes.inventory"  
     }
   }
   ```

   |      | 连接器的名称。                                               |
   | ---- | ------------------------------------------------------------ |
   |      | 连接器的配置。                                               |
   |      | 任何时候都只能运行一项任务。因为 MySQL 连接器读取 MySQL 服务器的`binlog`，所以使用单个连接器任务可确保正确的顺序和事件处理。Kafka Connect 服务使用连接器来启动一个或多个完成工作的任务，它会自动在 Kafka Connect 服务集群中分配正在运行的任务。如果任何服务停止或崩溃，这些任务将重新分配给正在运行的服务。 |
   |      | 数据库主机，即运行 MySQL 服务器的 Docker 容器的名称 ( `mysql`)。Docker 操作容器内的网络堆栈，以便可以`/etc/hosts`使用容器名称作为主机名来解析每个链接的容器。如果 MySQL 在正常网络上运行，您将为该值指定 IP 地址或可解析的主机名。 |
   |      | 唯一的服务器 ID 和名称。服务器名称是 MySQL 服务器或服务器集群的逻辑标识符。此名称将用作所有 Kafka 主题的前缀。 |
   |      | `inventory`只会检测到数据库中的更改。                        |
   |      | 连接器将使用此代理（您向其发送事件的同一代理）和主题名称将数据库模式的历史记录存储在 Kafka 中。`binlog`重新启动后，连接器将恢复在连接器应该开始读取的时间点存在的数据库模式。 |

   有关更多信息，请参阅[MySQL 连接器配置属性](https://debezium.io/documentation/reference/stable/connectors/mysql.html#mysql-connector-properties)。

|      | 出于安全原因，您不应将密码或其他机密以纯文本形式放入连接器配置中。相反，任何秘密都应该通过[KIP-297](https://cwiki.apache.org/confluence/display/KAFKA/KIP-297%3A+Externalizing+Secrets+for+Connect+Configurations)（“Externalizing Secrets for Connect Configurations”）中定义的机制进行外部化。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

1. 打开一个新终端，并使用`curl`命令注册 Debezium MySQL 连接器。

   此命令使用 Kafka Connect 服务的 API 提交`POST`对`/connectors`资源的请求，该请求包含描述新连接器（称为`inventory-connector`）的 JSON 文档。

   此命令用于`localhost`连接到 Docker 主机。如果您使用的是非本地 Docker 平台，请替换`localhost`为您的 Docker 主机的 IP 地址。

   ```shell
   $ curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.include.list": "inventory", "database.history.kafka.bootstrap.servers": "kafka:9092", "database.history.kafka.topic": "dbhistory.inventory" } }'
   ```

   |      | Windows 用户可能需要转义双引号。例如：`$ curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ \"name\": \"inventory-connector\", \"config\": { \"connector.class\": \"io.debezium.connector.mysql.MySqlConnector\", \"tasks.max\": \"1\", \"database.hostname\": \"mysql\", \"database.port\": \"3306\", \"database.user\": \"debezium\", \"database.password\": \"dbz\", \"database.server.id\": \"184054\", \"database.server.name\": \"dbserver1\", \"database.include.list\": \"inventory\", \"database.history.kafka.bootstrap.servers\": \"kafka:9092\", \"database.history.kafka.topic\": \"dbhistory.inventory\" } }'`否则，您可能会看到如下错误：`{"error_code":500,"message":"Unexpected character ('n' (code 110)): was expecting double-quote to start field name\n at [Source: (org.glassfish.jersey.message.internal.ReaderInterceptorExecutor$UnCloseableInputStream); line: 1, column: 4]"}` |
   | ---- | ------------------------------------------------------------ |
   |      |                                                              |

|      | 如果您使用 Podman，请运行以下命令：`$ curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "0.0.0.0", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.include.list": "inventory", "database.history.kafka.bootstrap.servers": "0.0.0.0:9092", "database.history.kafka.topic": "dbhistory.inventory" } }'` |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

1. 验证`inventory-connector`是否包含在连接器列表中：

   ```shell
   $ curl -H "Accept:application/json" localhost:8083/connectors/
   ["inventory-connector"]
   ```

2. 查看连接器的任务：

   ```shell
   $ curl -i -X GET -H "Accept:application/json" localhost:8083/connectors/inventory-connector
   ```

   您应该会看到类似于以下内容的响应（为便于阅读而格式化）：

   ```json
   HTTP/1.1 200 OK
   Date: Thu, 06 Feb 2020 22:12:03 GMT
   Content-Type: application/json
   Content-Length: 531
   Server: Jetty(9.4.20.v20190813)
   
   {
     "name": "inventory-connector",
     ...
     "tasks": [
       {
         "connector": "inventory-connector",  
         "task": 0
       }
     ]
   }
   ```

   |      | 连接器正在运行一个任务（task `0`）来完成它的工作。连接器仅支持单个任务，因为 MySQL 将其所有活动记录在一个连续的`binlog`. 这意味着连接器只需要一个阅读器即可获得所有事件的一致、有序的视图。 |
   | ---- | ------------------------------------------------------------ |
   |      |                                                              |

### 观察连接器启动

注册连接器时，会在 Kafka Connect 容器中生成大量日志输出。通过查看此输出，您可以更好地了解连接器从创建到开始读取 MySQL 服务器的`binlog`.

注册`inventory-connector`连接器后，您可以查看 Kafka Connect 容器 ( `connect`) 中的日志输出以跟踪连接器的状态。

前几行显示了`inventory-connector`正在创建和启动的连接器 ( )：

```shell
...
2021-11-30 01:38:44,223 INFO   ||  [Worker clientId=connect-1, groupId=1] Tasks [inventory-connector-0] configs updated   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
2021-11-30 01:38:44,224 INFO   ||  [Worker clientId=connect-1, groupId=1] Handling task config update by restarting tasks []   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
2021-11-30 01:38:44,224 INFO   ||  [Worker clientId=connect-1, groupId=1] Rebalance started   [org.apache.kafka.connect.runtime.distributed.WorkerCoordinator]
2021-11-30 01:38:44,224 INFO   ||  [Worker clientId=connect-1, groupId=1] (Re-)joining group   [org.apache.kafka.connect.runtime.distributed.WorkerCoordinator]
2021-11-30 01:38:44,227 INFO   ||  [Worker clientId=connect-1, groupId=1] Successfully joined group with generation Generation{generationId=3, memberId='connect-1-7b087c69-8ac5-4c56-9e6b-ec5adabf27e8', protocol='sessioned'}   [org.apache.kafka.connect.runtime.distributed.WorkerCoordinator]
2021-11-30 01:38:44,230 INFO   ||  [Worker clientId=connect-1, groupId=1] Successfully synced group in generation Generation{generationId=3, memberId='connect-1-7b087c69-8ac5-4c56-9e6b-ec5adabf27e8', protocol='sessioned'}   [org.apache.kafka.connect.runtime.distributed.WorkerCoordinator]
2021-11-30 01:38:44,231 INFO   ||  [Worker clientId=connect-1, groupId=1] Joined group at generation 3 with protocol version 2 and got assignment: Assignment{error=0, leader='connect-1-7b087c69-8ac5-4c56-9e6b-ec5adabf27e8', leaderUrl='http://172.17.0.7:8083/', offset=4, connectorIds=[inventory-connector], taskIds=[inventory-connector-0], revokedConnectorIds=[], revokedTaskIds=[], delay=0} with rebalance delay: 0   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
2021-11-30 01:38:44,232 INFO   ||  [Worker clientId=connect-1, groupId=1] Starting connectors and tasks using config offset 4   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
2021-11-30 01:38:44,232 INFO   ||  [Worker clientId=connect-1, groupId=1] Starting task inventory-connector-0   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
...
```

再往下，您应该会看到来自连接器的如下输出：

```shell
...
2021-11-30 01:38:44,406 INFO   ||  Kafka version: 3.0.0   [org.apache.kafka.common.utils.AppInfoParser]
2021-11-30 01:38:44,406 INFO   ||  Kafka commitId: 8cb0a5e9d3441962   [org.apache.kafka.common.utils.AppInfoParser]
2021-11-30 01:38:44,407 INFO   ||  Kafka startTimeMs: 1638236324406   [org.apache.kafka.common.utils.AppInfoParser]
2021-11-30 01:38:44,437 INFO   ||  Database history topic '(name=dbhistory.inventory, numPartitions=1, replicationFactor=1, replicasAssignments=null, configs={cleanup.policy=delete, retention.ms=9223372036854775807, retention.bytes=-1})' created   [io.debezium.relational.history.KafkaDatabaseHistory]
2021-11-30 01:38:44,497 INFO   ||  App info kafka.admin.client for dbserver1-dbhistory unregistered   [org.apache.kafka.common.utils.AppInfoParser]
2021-11-30 01:38:44,499 INFO   ||  Metrics scheduler closed   [org.apache.kafka.common.metrics.Metrics]
2021-11-30 01:38:44,499 INFO   ||  Closing reporter org.apache.kafka.common.metrics.JmxReporter   [org.apache.kafka.common.metrics.Metrics]
2021-11-30 01:38:44,499 INFO   ||  Metrics reporters closed   [org.apache.kafka.common.metrics.Metrics]
2021-11-30 01:38:44,499 INFO   ||  Reconnecting after finishing schema recovery   [io.debezium.connector.mysql.MySqlConnectorTask]
2021-11-30 01:38:44,524 INFO   ||  Requested thread factory for connector MySqlConnector, id = dbserver1 named = change-event-source-coordinator   [io.debezium.util.Threads]
2021-11-30 01:38:44,525 INFO   ||  Creating thread debezium-mysqlconnector-dbserver1-change-event-source-coordinator   [io.debezium.util.Threads]
2021-11-30 01:38:44,526 INFO   ||  WorkerSourceTask{id=inventory-connector-0} Source task finished initialization and start   [org.apache.kafka.connect.runtime.WorkerSourceTask]
2021-11-30 01:38:44,529 INFO   MySQL|dbserver1|snapshot  Metrics registered   [io.debezium.pipeline.ChangeEventSourceCoordinator]
2021-11-30 01:38:44,529 INFO   MySQL|dbserver1|snapshot  Context created   [io.debezium.pipeline.ChangeEventSourceCoordinator]
2021-11-30 01:38:44,534 INFO   MySQL|dbserver1|snapshot  No previous offset has been found   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:44,534 INFO   MySQL|dbserver1|snapshot  According to the connector configuration both schema and data will be snapshotted   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:44,534 INFO   MySQL|dbserver1|snapshot  Snapshot step 1 - Preparing   [io.debezium.relational.RelationalSnapshotChangeEventSource]
...
```

Debezium 日志输出使用*映射诊断上下文*(MDC) 在日志输出中提供特定于线程的信息，并更容易理解多线程 Kafka Connect 服务中发生的情况。这包括连接器类型（`MySQL`在上面的日志消息中）、连接器的逻辑名称（`dbserver1`上面）和连接器的活动（`task`和`snapshot`）`binlog`。

在上面的日志输出中，前几行涉及`task`连接器的活动，并报告了一些簿记信息（在这种情况下，连接器是在没有事先偏移的情况下启动的）。接下来的三行涉及`snapshot`连接器的活动，并报告正在使用`debezium`MySQL 用户以及与该用户关联的 MySQL 授权启动快照。

|      | 如果连接器无法连接，或者它没有看到任何表或`binlog`，请检查这些授权以确保包括上面列出的所有内容。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

接下来，连接器报告构成快照操作的步骤：

```shell
...
2021-11-30 01:38:44,534 INFO   MySQL|dbserver1|snapshot  Snapshot step 1 - Preparing   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:44,535 INFO   MySQL|dbserver1|snapshot  Snapshot step 2 - Determining captured tables   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:44,535 INFO   MySQL|dbserver1|snapshot  Read list of available databases   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:44,537 INFO   MySQL|dbserver1|snapshot  	 list of available databases is: [information_schema, inventory, mysql, performance_schema, sys]   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:44,537 INFO   MySQL|dbserver1|snapshot  Read list of available tables in each database   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:44,548 INFO   MySQL|dbserver1|snapshot  	snapshot continuing with database(s): [inventory]   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:44,551 INFO   MySQL|dbserver1|snapshot  Snapshot step 3 - Locking captured tables [inventory.addresses, inventory.customers, inventory.geom, inventory.orders, inventory.products, inventory.products_on_hand]   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:44,552 INFO   MySQL|dbserver1|snapshot  Flush and obtain global read lock to prevent writes to database   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:44,557 INFO   MySQL|dbserver1|snapshot  Snapshot step 4 - Determining snapshot offset   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:44,560 INFO   MySQL|dbserver1|snapshot  Read binlog position of MySQL primary server   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:44,562 INFO   MySQL|dbserver1|snapshot  	 using binlog 'mysql-bin.000003' at position '156' and gtid ''   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:44,562 INFO   MySQL|dbserver1|snapshot  Snapshot step 5 - Reading structure of captured tables   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:44,562 INFO   MySQL|dbserver1|snapshot  All eligible tables schema should be captured, capturing: [inventory.addresses, inventory.customers, inventory.geom, inventory.orders, inventory.products, inventory.products_on_hand]   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:45,058 INFO   MySQL|dbserver1|snapshot  Reading structure of database 'inventory'   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:45,187 INFO   MySQL|dbserver1|snapshot  Snapshot step 6 - Persisting schema history   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,273 INFO   MySQL|dbserver1|snapshot  Releasing global read lock to enable MySQL writes   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:45,274 INFO   MySQL|dbserver1|snapshot  Writes to MySQL tables prevented for a total of 00:00:00.717   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
2021-11-30 01:38:45,274 INFO   MySQL|dbserver1|snapshot  Snapshot step 7 - Snapshotting data   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,275 INFO   MySQL|dbserver1|snapshot  Snapshotting contents of 6 tables while still in transaction   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,275 INFO   MySQL|dbserver1|snapshot  Exporting data from table 'inventory.addresses' (1 of 6 tables)   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,276 INFO   MySQL|dbserver1|snapshot  	 For table 'inventory.addresses' using select statement: 'SELECT `id`, `customer_id`, `street`, `city`, `state`, `zip`, `type` FROM `inventory`.`addresses`'   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,295 INFO   MySQL|dbserver1|snapshot  	 Finished exporting 7 records for table 'inventory.addresses'; total duration '00:00:00.02'   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,296 INFO   MySQL|dbserver1|snapshot  Exporting data from table 'inventory.customers' (2 of 6 tables)   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,296 INFO   MySQL|dbserver1|snapshot  	 For table 'inventory.customers' using select statement: 'SELECT `id`, `first_name`, `last_name`, `email` FROM `inventory`.`customers`'   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,304 INFO   MySQL|dbserver1|snapshot  	 Finished exporting 4 records for table 'inventory.customers'; total duration '00:00:00.008'   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,304 INFO   MySQL|dbserver1|snapshot  Exporting data from table 'inventory.geom' (3 of 6 tables)   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,305 INFO   MySQL|dbserver1|snapshot  	 For table 'inventory.geom' using select statement: 'SELECT `id`, `g`, `h` FROM `inventory`.`geom`'   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,316 INFO   MySQL|dbserver1|snapshot  	 Finished exporting 3 records for table 'inventory.geom'; total duration '00:00:00.011'   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,316 INFO   MySQL|dbserver1|snapshot  Exporting data from table 'inventory.orders' (4 of 6 tables)   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,316 INFO   MySQL|dbserver1|snapshot  	 For table 'inventory.orders' using select statement: 'SELECT `order_number`, `order_date`, `purchaser`, `quantity`, `product_id` FROM `inventory`.`orders`'   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,325 INFO   MySQL|dbserver1|snapshot  	 Finished exporting 4 records for table 'inventory.orders'; total duration '00:00:00.008'   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,325 INFO   MySQL|dbserver1|snapshot  Exporting data from table 'inventory.products' (5 of 6 tables)   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,325 INFO   MySQL|dbserver1|snapshot  	 For table 'inventory.products' using select statement: 'SELECT `id`, `name`, `description`, `weight` FROM `inventory`.`products`'   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,343 INFO   MySQL|dbserver1|snapshot  	 Finished exporting 9 records for table 'inventory.products'; total duration '00:00:00.017'   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,344 INFO   MySQL|dbserver1|snapshot  Exporting data from table 'inventory.products_on_hand' (6 of 6 tables)   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,344 INFO   MySQL|dbserver1|snapshot  	 For table 'inventory.products_on_hand' using select statement: 'SELECT `product_id`, `quantity` FROM `inventory`.`products_on_hand`'   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,353 INFO   MySQL|dbserver1|snapshot  	 Finished exporting 9 records for table 'inventory.products_on_hand'; total duration '00:00:00.009'   [io.debezium.relational.RelationalSnapshotChangeEventSource]
2021-11-30 01:38:45,355 INFO   MySQL|dbserver1|snapshot  Snapshot - Final stage   [io.debezium.pipeline.source.AbstractSnapshotChangeEventSource]
2021-11-30 01:38:45,356 INFO   MySQL|dbserver1|snapshot  Snapshot ended with SnapshotResult [status=COMPLETED, offset=MySqlOffsetContext [sourceInfoSchema=Schema{io.debezium.connector.mysql.Source:STRUCT}, sourceInfo=SourceInfo [currentGtid=null, currentBinlogFilename=mysql-bin.000003, currentBinlogPosition=156, currentRowNumber=0, serverId=0, sourceTime=2021-11-30T01:38:45.352Z, threadId=-1, currentQuery=null, tableIds=[inventory.products_on_hand], databaseName=inventory], snapshotCompleted=true, transactionContext=TransactionContext [currentTransactionId=null, perTableEventCount={}, totalEventCount=0], restartGtidSet=null, currentGtidSet=null, restartBinlogFilename=mysql-bin.000003, restartBinlogPosition=156, restartRowsToSkip=0, restartEventsToSkip=0, currentEventLengthInBytes=0, inTransaction=false, transactionId=null, incrementalSnapshotContext =IncrementalSnapshotContext [windowOpened=false, chunkEndPosition=null, dataCollectionsToSnapshot=[], lastEventKeySent=null, maximumKey=null]]]   [io.debezium.pipeline.ChangeEventSourceCoordinator]
...
```

这些步骤中的每一个都报告连接器正在执行的操作以执行一致的快照。例如，第 6 步涉及`create`对正在捕获的表的 DDL 语句进行逆向工程，并在获取全局写锁后仅 1 秒，第 7 步读取每个表中的所有行并报告所用时间和数量找到的行数。在这种情况下，连接器在不到 1 秒的时间内完成了其一致的快照。

|      | 数据库的快照过程将花费更长的时间，但连接器会输出足够的日志消息，您可以跟踪它正在处理的内容，即使表具有大量行也是如此。尽管在快照过程开始时使用了独占写锁，但即使对于大型数据库，它也不应该持续很长时间。这是因为在复制任何数据之前释放了锁。有关更多信息，请参阅[MySQL 连接器文档](https://debezium.io/documentation/reference/stable/connectors/mysql.html#debezium-connector-for-mysql)。 |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

接下来，Kafka Connect 报告了一些“错误”。但是，您可以放心地忽略这些警告：这些消息仅意味着创建了*新的*Kafka 主题，并且 Kafka 必须为每个主题分配一个新的领导者：

```shell
...
2021-11-30 01:38:45,555 WARN   ||  [Producer clientId=connector-producer-inventory-connector-0] Error while fetching metadata with correlation id 3 : {dbserver1=LEADER_NOT_AVAILABLE}   [org.apache.kafka.clients.NetworkClient]
2021-11-30 01:38:45,691 WARN   ||  [Producer clientId=connector-producer-inventory-connector-0] Error while fetching metadata with correlation id 9 : {dbserver1.inventory.addresses=LEADER_NOT_AVAILABLE}   [org.apache.kafka.clients.NetworkClient]
2021-11-30 01:38:45,813 WARN   ||  [Producer clientId=connector-producer-inventory-connector-0] Error while fetching metadata with correlation id 13 : {dbserver1.inventory.customers=LEADER_NOT_AVAILABLE}   [org.apache.kafka.clients.NetworkClient]
2021-11-30 01:38:45,927 WARN   ||  [Producer clientId=connector-producer-inventory-connector-0] Error while fetching metadata with correlation id 18 : {dbserver1.inventory.geom=LEADER_NOT_AVAILABLE}   [org.apache.kafka.clients.NetworkClient]
2021-11-30 01:38:46,043 WARN   ||  [Producer clientId=connector-producer-inventory-connector-0] Error while fetching metadata with correlation id 22 : {dbserver1.inventory.orders=LEADER_NOT_AVAILABLE}   [org.apache.kafka.clients.NetworkClient]
2021-11-30 01:38:46,153 WARN   ||  [Producer clientId=connector-producer-inventory-connector-0] Error while fetching metadata with correlation id 26 : {dbserver1.inventory.products=LEADER_NOT_AVAILABLE}   [org.apache.kafka.clients.NetworkClient]
2021-11-30 01:38:46,269 WARN   ||  [Producer clientId=connector-producer-inventory-connector-0] Error while fetching metadata with correlation id 31 : {dbserver1.inventory.products_on_hand=LEADER_NOT_AVAILABLE}   [org.apache.kafka.clients.NetworkClient]
...
```

最后，日志输出显示连接器已从其快照模式转换为持续读取 MySQL 服务器的`binlog`：

```shell
...
2021-11-30 01:38:45,362 INFO   MySQL|dbserver1|streaming  Starting streaming   [io.debezium.pipeline.ChangeEventSourceCoordinator]
...
Nov 30, 2021 1:38:45 AM com.github.shyiko.mysql.binlog.BinaryLogClient connect
INFO: Connected to mysql:3306 at mysql-bin.000003/156 (sid:184054, cid:13)
2021-11-30 01:38:45,392 INFO   MySQL|dbserver1|binlog  Connected to MySQL binlog at mysql:3306, starting at MySqlOffsetContext [sourceInfoSchema=Schema{io.debezium.connector.mysql.Source:STRUCT}, sourceInfo=SourceInfo [currentGtid=null, currentBinlogFilename=mysql-bin.000003, currentBinlogPosition=156, currentRowNumber=0, serverId=0, sourceTime=2021-11-30T01:38:45.352Z, threadId=-1, currentQuery=null, tableIds=[inventory.products_on_hand], databaseName=inventory], snapshotCompleted=true, transactionContext=TransactionContext [currentTransactionId=null, perTableEventCount={}, totalEventCount=0], restartGtidSet=null, currentGtidSet=null, restartBinlogFilename=mysql-bin.000003, restartBinlogPosition=156, restartRowsToSkip=0, restartEventsToSkip=0, currentEventLengthInBytes=0, inTransaction=false, transactionId=null, incrementalSnapshotContext =IncrementalSnapshotContext [windowOpened=false, chunkEndPosition=null, dataCollectionsToSnapshot=[], lastEventKeySent=null, maximumKey=null]]   [io.debezium.connector.mysql.MySqlStreamingChangeEventSource]
2021-11-30 01:38:45,392 INFO   MySQL|dbserver1|streaming  Waiting for keepalive thread to start   [io.debezium.connector.mysql.MySqlStreamingChangeEventSource]
2021-11-30 01:38:45,393 INFO   MySQL|dbserver1|binlog  Creating thread debezium-mysqlconnector-dbserver1-binlog-client   [io.debezium.util.Threads]
...
```

## 查看更改事件

部署 Debezium MySQL 连接器后，它开始监视`inventory`数据库中的数据更改事件。

当您观察连接器启动时，您看到事件被写入带有`dbserver1`前缀（连接器名称）的以下主题：

- `dbserver1`

  写入所有 DDL 语句的[架构更改主题。](https://debezium.io/documentation/reference/stable/connectors/mysql.html#mysql-schema-change-topic)

- `dbserver1.inventory.products`

  捕获数据库`products`中表的更改事件。`inventory`

- `dbserver1.inventory.products_on_hand`

  捕获数据库`products_on_hand`中表的更改事件。`inventory`

- `dbserver1.inventory.customers`

  捕获数据库`customers`中表的更改事件。`inventory`

- `dbserver1.inventory.orders`

  捕获数据库`orders`中表的更改事件。`inventory`

在本教程中，您将探索该`dbserver1.inventory.customers`主题。在本主题中，您将看到不同类型的更改事件，以了解连接器如何捕获它们：

- [Viewing a *create* event](https://debezium.io/documentation/reference/stable/tutorial.html#viewing-create-event)
- [Updating the database and viewing the *update* event](https://debezium.io/documentation/reference/stable/tutorial.html#updating-database-viewing-update-event)
- [Deleting a record in the database and viewing the *delete* event](https://debezium.io/documentation/reference/stable/tutorial.html#deleting-record-database-viewing-delete-event)
- [Restarting Kafka Connect and changing the database](https://debezium.io/documentation/reference/stable/tutorial.html#restarting-kafka-connect-service)

### Viewing a *create* event

By viewing the `dbserver1.inventory.customers` topic, you can see how the MySQL connector captured *create* events in the `inventory` database. In this case, the *create* events capture new customers being added to the database.

Procedure

1. Open a new terminal, and use it to start the `watch-topic` utility to watch the `dbserver1.inventory.customers` topic from the beginning of the topic.

   The `watch-topic` utility is very simple and limited in functionality. It is not intended to be used by an application to consume events. In that scenario, you would instead use Kafka consumers and the applicable consumer libraries that offer full functionality and flexibility.

   This command runs the `watch-topic` utility in a new container using the 1.9 version of the `debezium/kafka` image:

   ```shell
   $ docker run -it --rm --name watcher --link zookeeper:zookeeper --link kafka:kafka quay.io/debezium/kafka:1.9 watch-topic -a -k dbserver1.inventory.customers
   ```

   - `-a`

     Watches all events since the topic was created. Without this option, `watch-topic` would only show the events recorded after you start watching.

   - `-k`

     Specifies that the output should include the event’s key. In this case, this contains the row’s primary key.

   |      | If you use Podman, run the following command:`$ sudo podman run -it --rm --name watcher --pod dbz quay.io/debezium/kafka:1.9 watch-topic -a -k dbserver1.inventory.customers` |
   | ---- | ------------------------------------------------------------ |
   |      |                                                              |

   The `watch-topic` utility returns the event records from the `customers` table. There are four events, one for each row in the table. Each event is formatted in JSON, because that is how you configured the Kafka Connect service. There are two JSON documents for each event: one for the key, and one for the value.

   You should see output similar to the following:

   ```shell
   Using ZOOKEEPER_CONNECT=172.17.0.2:2181
   Using KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.17.0.7:9092
   Using KAFKA_BROKER=172.17.0.3:9092
   Contents of topic dbserver1.inventory.customers:
   {"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1001}}
   ...
   ```

   |      | This utility keeps watching the topic, so any new events will automatically appear as long as the utility is running. |
   | ---- | ------------------------------------------------------------ |
   |      |                                                              |

2. For the last event, review the details of the *key*.

   Here are the details of the *key* of the last event (formatted for readability):

   ```json
   {
     "schema":{
       "type":"struct",
         "fields":[
           {
             "type":"int32",
             "optional":false,
             "field":"id"
           }
         ],
       "optional":false,
       "name":"dbserver1.inventory.customers.Key"
     },
     "payload":{
       "id":1004
     }
   }
   ```

   The event has two parts: a `schema` and a `payload`. The `schema` contains a Kafka Connect schema describing what is in the payload. In this case, the payload is a `struct` named `dbserver1.inventory.customers.Key` that is not optional and has one required field (`id` of type `int32`).

   The `payload` has a single `id` field, with a value of `1004`.

   By reviewing the *key* of the event, you can see that this event applies to the row in the `inventory.customers` table whose `id` primary key column had a value of `1004`.

3. Review the details of the same event’s *value*.

   该事件的*值*显示该行已创建，并描述了它包含的内容（在本例中，是插入行的`id`、`first_name`、`last_name`和）。`email`

   以下是最后一个事件的*值*的详细信息（为便于阅读而格式化）：

   ```json
   {
     "schema": {
       "type": "struct",
       "fields": [
         {
           "type": "struct",
           "fields": [
             {
               "type": "int32",
               "optional": false,
               "field": "id"
             },
             {
               "type": "string",
               "optional": false,
               "field": "first_name"
             },
             {
               "type": "string",
               "optional": false,
               "field": "last_name"
             },
             {
               "type": "string",
               "optional": false,
               "field": "email"
             }
           ],
           "optional": true,
           "name": "dbserver1.inventory.customers.Value",
           "field": "before"
         },
         {
           "type": "struct",
           "fields": [
             {
               "type": "int32",
               "optional": false,
               "field": "id"
             },
             {
               "type": "string",
               "optional": false,
               "field": "first_name"
             },
             {
               "type": "string",
               "optional": false,
               "field": "last_name"
             },
             {
               "type": "string",
               "optional": false,
               "field": "email"
             }
           ],
           "optional": true,
           "name": "dbserver1.inventory.customers.Value",
           "field": "after"
         },
         {
           "type": "struct",
           "fields": [
             {
               "type": "string",
               "optional": true,
               "field": "version"
             },
             {
               "type": "string",
               "optional": false,
               "field": "name"
             },
             {
               "type": "int64",
               "optional": false,
               "field": "server_id"
             },
             {
               "type": "int64",
               "optional": false,
               "field": "ts_sec"
             },
             {
               "type": "string",
               "optional": true,
               "field": "gtid"
             },
             {
               "type": "string",
               "optional": false,
               "field": "file"
             },
             {
               "type": "int64",
               "optional": false,
               "field": "pos"
             },
             {
               "type": "int32",
               "optional": false,
               "field": "row"
             },
             {
               "type": "boolean",
               "optional": true,
               "field": "snapshot"
             },
             {
               "type": "int64",
               "optional": true,
               "field": "thread"
             },
             {
               "type": "string",
               "optional": true,
               "field": "db"
             },
             {
               "type": "string",
               "optional": true,
               "field": "table"
             }
           ],
           "optional": false,
           "name": "io.debezium.connector.mysql.Source",
           "field": "source"
         },
         {
           "type": "string",
           "optional": false,
           "field": "op"
         },
         {
           "type": "int64",
           "optional": true,
           "field": "ts_ms"
         }
       ],
       "optional": false,
       "name": "dbserver1.inventory.customers.Envelope",
       "version": 1
     },
     "payload": {
       "before": null,
       "after": {
         "id": 1004,
         "first_name": "Anne",
         "last_name": "Kretchmar",
         "email": "annek@noanswer.org"
       },
       "source": {
         "version": "1.9.6.Final",
         "name": "dbserver1",
         "server_id": 0,
         "ts_sec": 0,
         "gtid": null,
         "file": "mysql-bin.000003",
         "pos": 154,
         "row": 0,
         "snapshot": true,
         "thread": null,
         "db": "inventory",
         "table": "customers"
       },
       "op": "r",
       "ts_ms": 1486500577691
     }
   }
   ```

   这部分事件要长得多，但和事件的*key*一样，它也有 a`schema`和 a `payload`。`schema`包含一个名为（版本 1）的 Kafka Connect 模式，`dbserver1.inventory.customers.Envelope`它可以包含五个字段：

   - `op`

     包含描述操作类型的字符串值的必填字段。MySQL 连接器的值`c`用于创建（或插入）、`u`更新、`d`删除和`r`读取（在快照的情况下）。

   - `before`

     一个可选字段，如果存在，则包含事件发生*之前行的状态。*该结构将由`dbserver1.inventory.customers.Value`Kafka Connect 模式描述，`dbserver1`连接器将其用于表中的所有行`inventory.customers`。

   - `after`

     一个可选字段，如果存在，则包含事件发生*后行的状态。*该结构`dbserver1.inventory.customers.Value`由`before`.

   - `source`

     一个必填字段，其中包含描述事件源元数据的结构，在 MySQL 的情况下，它包含几个字段：连接器名称、记录事件的文件的名称、该`binlog`文件中`binlog`事件出现的位置，事件中的行（如果有多个），受影响的数据库和表的名称，进行更改的 MySQL 线程 ID，此事件是否是快照的一部分，以及 MySQL 服务器（如果可用） ID，以及以秒为单位的时间戳。

   - `ts_ms`

     一个可选字段，如果存在，则包含连接器处理事件的时间（使用运行 Kafka Connect 任务的 JVM 中的系统时钟）。

   |      | 事件的 JSON 表示比它们描述的行长得多。这是因为，对于每个事件键和值，Kafka Connect都会提供描述*有效负载的**模式*。随着时间的推移，这种结构可能会发生变化。但是，在事件本身中拥有键和值的模式使得使用应用程序更容易理解消息，尤其是随着时间的推移。Debezium MySQL 连接器基于数据库表的结构构造这些模式。如果您使用 DDL 语句来更改 MySQL 数据库中的表定义，则连接器会读取这些 DDL 语句并更新其 Kafka Connect 模式。这是使每个事件的结构与事件发生时它起源的表完全一样的唯一方法。但是，包含单个表的所有事件的 Kafka 主题可能具有对应于表定义的每个状态的事件。JSON 转换器在每条消息中都包含键和值模式，因此它确实会产生非常冗长的事件。或者，您可以使用[Apache Avro](https://debezium.io/documentation/reference/stable/configuration/avro.html)作为序列化格式，这会产生更小的事件消息。这是因为它将每个 Kafka Connect 模式转换为 Avro 模式，并将 Avro 模式存储在单独的模式注册表服务中。因此，当 Avro 转换器序列化事件消息时，它只放置模式的唯一标识符以及值的 Avro 编码二进制表示。结果，通过网络传输并存储在 Kafka 中的序列化消息比您在此处看到的要小得多。事实上，Avro 转换器能够使用 Avro 模式演变技术来维护模式注册表中每个模式的历史记录。 |
   | ---- | ------------------------------------------------------------ |
   |      |                                                              |

4. 将事件的*键*和*值*模式与`inventory`数据库的状态进行比较。在运行 MySQL 命令行客户端的终端中，运行以下语句：

   ```sql
   mysql> SELECT * FROM customers;
   +------+------------+-----------+-----------------------+
   | id   | first_name | last_name | email                 |
   +------+------------+-----------+-----------------------+
   | 1001 | Sally      | Thomas    | sally.thomas@acme.com |
   | 1002 | George     | Bailey    | gbailey@foobar.com    |
   | 1003 | Edward     | Walker    | ed@walker.com         |
   | 1004 | Anne       | Kretchmar | annek@noanswer.org    |
   +------+------------+-----------+-----------------------+
   4 rows in set (0.00 sec)
   ```

   这表明您查看的事件记录与数据库中的记录匹配。

### 更新数据库并查看*更新*事件

现在您已经了解了 Debezium MySQL 连接器如何捕获数据库中的*创建*事件，`inventory`现在您将更改其中一条记录并查看连接器如何捕获它。

通过完成此过程，您将了解如何查找有关数据库提交中更改内容的详细信息，以及如何比较更改事件以确定更改发生时间与其他更改的关系。

程序

1. 在运行 MySQL 命令行客户端的终端中，运行以下语句：

   ```sql
   mysql> UPDATE customers SET first_name='Anne Marie' WHERE id=1004;
   Query OK, 1 row affected (0.05 sec)
   Rows matched: 1  Changed: 1  Warnings: 0
   ```

2. 查看更新的`customers`表格：

   ```sql
   mysql> SELECT * FROM customers;
   +------+------------+-----------+-----------------------+
   | id   | first_name | last_name | email                 |
   +------+------------+-----------+-----------------------+
   | 1001 | Sally      | Thomas    | sally.thomas@acme.com |
   | 1002 | George     | Bailey    | gbailey@foobar.com    |
   | 1003 | Edward     | Walker    | ed@walker.com         |
   | 1004 | Anne Marie | Kretchmar | annek@noanswer.org    |
   +------+------------+-----------+-----------------------+
   4 rows in set (0.00 sec)
   ```

3. 切换到终端运行`watch-topic`以查看*新*的第五个事件。

   通过更改`customers`表中的记录，Debezium MySQL 连接器生成了一个新事件。您应该会看到两个新的 JSON 文档：一个是事件的*key*，一个是新事件的*value*。

   以下是*更新事件**密钥*的详细信息（为便于阅读而格式化）：

   ```json
     {
       "schema": {
         "type": "struct",
         "name": "dbserver1.inventory.customers.Key"
         "optional": false,
         "fields": [
           {
             "field": "id",
             "type": "int32",
             "optional": false
           }
         ]
       },
       "payload": {
         "id": 1004
       }
     }
   ```

   此*键*与之前事件的*键*相同。

   这是新事件的*价值*。该部分没有变化`schema`，因此仅`payload`显示该部分（为便于阅读而格式化）：

   ```json
   {
     "schema": {...},
     "payload": {
       "before": {  
         "id": 1004,
         "first_name": "Anne",
         "last_name": "Kretchmar",
         "email": "annek@noanswer.org"
       },
       "after": {  
         "id": 1004,
         "first_name": "Anne Marie",
         "last_name": "Kretchmar",
         "email": "annek@noanswer.org"
       },
       "source": {  
         "name": "1.9.6.Final",
         "name": "dbserver1",
         "server_id": 223344,
         "ts_sec": 1486501486,
         "gtid": null,
         "file": "mysql-bin.000003",
         "pos": 364,
         "row": 0,
         "snapshot": null,
         "thread": 3,
         "db": "inventory",
         "table": "customers"
       },
       "op": "u",  
       "ts_ms": 1486501486308  
     }
   }
   ```

   |      | 该`before`字段现在具有数据库提交之前的行的状态和值。         |
   | ---- | ------------------------------------------------------------ |
   |      | 该`after`字段现在具有行的更新状态，`first_name`值为 now `Anne Marie`。 |
   |      | `source`字段结构具有许多与以前相同的值，除了和`ts_sec`字段`pos`已更改（`file`在其他情况下可能已更改）。 |
   |      | `op`字段值为 now `u`，表示该行因更新而更改。                 |
   |      | 该`ts_ms`字段显示 Debezium 处理此事件的时间戳。              |

   By viewing the `payload` section, you can learn several important things about the *update* event:

   - By comparing the `before` and `after` structures, you can determine what actually changed in the affected row because of the commit.
   - By reviewing the `source` structure, you can find information about MySQL’s record of the change (providing traceability).
   - By comparing the `payload` section of an event to other events in the same topic (or a different topic), you can determine whether the event occurred before, after, or as part of the same MySQL commit as another event.

### Deleting a record in the database and viewing the *delete* event

Now that you have seen how the Debezium MySQL connector captured the *create* and *update* events in the `inventory` database, you will now delete one of the records and see how the connector captures it.

By completing this procedure, you will learn how to find details about *delete* events, and how Kafka uses *log compaction* to reduce the number of *delete* events while still enabling consumers to get all of the events.

Procedure

1. In the terminal that is running the MySQL command line client, run the following statement:

   ```sql
   mysql> DELETE FROM customers WHERE id=1004;
   Query OK, 1 row affected (0.00 sec)
   ```

   |      | 如果上述命令因违反外键约束而失败，则必须使用以下语句从*地址表中删除客户地址的引用：*`mysql> DELETE FROM addresses WHERE customer_id=1004;` |
   | ---- | ------------------------------------------------------------ |
   |      |                                                              |

2. 切换到终端运行`watch-topic`以查看*两个*新事件。

   通过删除`customers`表中的一行，Debezium MySQL 连接器生成了两个新事件。

3. 查看第一个新事件的*键*和*值*。

   以下是第一个新事件的*密钥*的详细信息（为便于阅读而格式化）：

   ```json
   {
     "schema": {
       "type": "struct",
       "name": "dbserver1.inventory.customers.Key"
       "optional": false,
       "fields": [
         {
           "field": "id",
           "type": "int32",
           "optional": false
         }
       ]
     },
     "payload": {
       "id": 1004
     }
   }
   ```

   此*键*与您查看的前两个事件中的*键*相同。

   这是第一个新事件的*值*（为便于阅读而格式化）：

   ```json
   {
     "schema": {...},
     "payload": {
       "before": {  
         "id": 1004,
         "first_name": "Anne Marie",
         "last_name": "Kretchmar",
         "email": "annek@noanswer.org"
       },
       "after": null,  
       "source": {  
         "name": "1.9.6.Final",
         "name": "dbserver1",
         "server_id": 223344,
         "ts_sec": 1486501558,
         "gtid": null,
         "file": "mysql-bin.000003",
         "pos": 725,
         "row": 0,
         "snapshot": null,
         "thread": 3,
         "db": "inventory",
         "table": "customers"
       },
       "op": "d",  
       "ts_ms": 1486501558315  
     }
   }
   ```

   |      | 该`before`字段现在具有与数据库提交一起删除的行的状态。       |
   | ---- | ------------------------------------------------------------ |
   |      | 该`after`字段是`null`因为该行不再存在。                      |
   |      | `source`字段结构具有许多与以前相同的值，除了和`ts_sec`字段`pos`已更改（`file`在其他情况下可能已更改）。 |
   |      | `op`字段值为 now `d`，表示该行已被删除。                     |
   |      | 该`ts_ms`字段显示 Debezium 处理此事件的时间戳。              |

   因此，此事件为消费者提供了处理删除行所需的信息。还提供了旧值，因为某些消费者可能要求他们正确处理删除。

4. 查看第二个新事件的*键*和*值*。

   这是第二个新事件的*关键*（为便于阅读而格式化）：

   ```json
     {
       "schema": {
         "type": "struct",
         "name": "dbserver1.inventory.customers.Key"
         "optional": false,
         "fields": [
           {
             "field": "id",
             "type": "int32",
             "optional": false
           }
         ]
       },
       "payload": {
         "id": 1004
       }
     }
   ```

   再一次，此*键*与您查看的前三个事件中的键完全相同。

   这是同一事件的*值（为便于阅读而格式化）：*

   ```json
   {
     "schema": null,
     "payload": null
   }
   ```

   如果 Kafka 设置为*log compacted*，如果主题后面至少有一条具有相同键的消息，它将从主题中删除较旧的消息。最后一个事件称为*墓碑*事件，因为它有一个键和一个空值。这意味着 Kafka 将删除所有具有相同密钥的先前消息。即使之前的消息将被删除，墓碑事件意味着消费者仍然可以从头开始阅读主题而不会错过任何事件。

### 重启 Kafka Connect 服务

现在您已经了解了 Debezium MySQL 连接器如何捕获*创建*、*更新*和*删除*事件，现在您将看到它如何捕获更改事件，即使它没有运行。

Kafka Connect 服务自动管理其注册连接器的任务。因此，如果它下线，当它重新启动时，它将启动任何非运行任务。这意味着即使 Debezium 没有运行，它仍然可以报告数据库中的更改。

在此过程中，您将停止 Kafka Connect，更改数据库中的一些数据，然后重新启动 Kafka Connect 以查看更改事件。

程序

1. 打开一个新终端并使用它来停止`connect`运行 Kafka Connect 服务的容器：

   ```shell
   $ docker stop connect
   ```

   容器停止，`connect`Kafka Connect 服务正常关闭。

   因为您使用该`--rm`选项运行容器，所以 Docker 一旦停止就会删除容器。

2. 服务关闭时，切换到 MySQL 命令行客户端的终端，并添加几条记录：

   ```sql
   mysql> INSERT INTO customers VALUES (default, "Sarah", "Thompson", "kitt@acme.com");
   mysql> INSERT INTO customers VALUES (default, "Kenneth", "Anderson", "kander@acme.com");
   ```

   记录被添加到数据库中。但是，由于 Kafka Connect 未运行， `watch-topic`因此不会记录任何更新。

   |      | 在生产系统中，您将有足够的代理来处理生产者和消费者，并为每个主题维护最少数量的同步副本。因此，如果有足够多的代理失败，以至于不再有最小数量的 ISR，Kafka 将变得不可用。在这种情况下，生产者（如 Debezium 连接器）和消费者将等待 Kafka 集群或网络恢复。这意味着，当数据库中的数据发生更改时，您的消费者可能暂时看不到任何更改事件。这是因为没有产生变化事件。一旦 Kafka 集群重新启动或网络恢复，Debezium 将恢复生成更改事件，并且您的消费者将继续消费他们中断的事件。 |
   | ---- | ------------------------------------------------------------ |
   |      |                                                              |

3. 打开一个新终端，并使用它来重新启动容器中的 Kafka Connect 服务。

   此命令使用最初启动时使用的相同选项启动 Kafka Connect：

   ```shell
   $ docker run -it --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka --link mysql:mysql quay.io/debezium/connect:1.9
   ```

   Kafka Connect 服务启动，连接到 Kafka，读取前一个服务的配置，并启动注册的连接器，这些连接器将在上次停止的地方恢复。

   以下是此重新启动服务的最后几行：

   ```shell
   ...
   2021-11-30 01:49:07,938 INFO   ||  Get all known binlogs from MySQL   [io.debezium.connector.mysql.MySqlConnection]
   2021-11-30 01:49:07,941 INFO   ||  MySQL has the binlog file 'mysql-bin.000003' required by the connector   [io.debezium.connector.mysql.MySqlConnectorTask]
   2021-11-30 01:49:07,967 INFO   ||  Requested thread factory for connector MySqlConnector, id = dbserver1 named = change-event-source-coordinator   [io.debezium.util.Threads]
   2021-11-30 01:49:07,968 INFO   ||  Creating thread debezium-mysqlconnector-dbserver1-change-event-source-coordinator   [io.debezium.util.Threads]
   2021-11-30 01:49:07,968 INFO   ||  WorkerSourceTask{id=inventory-connector-0} Source task finished initialization and start   [org.apache.kafka.connect.runtime.WorkerSourceTask]
   2021-11-30 01:49:07,971 INFO   MySQL|dbserver1|snapshot  Metrics registered   [io.debezium.pipeline.ChangeEventSourceCoordinator]
   2021-11-30 01:49:07,971 INFO   MySQL|dbserver1|snapshot  Context created   [io.debezium.pipeline.ChangeEventSourceCoordinator]
   2021-11-30 01:49:07,976 INFO   MySQL|dbserver1|snapshot  A previous offset indicating a completed snapshot has been found. Neither schema nor data will be snapshotted.   [io.debezium.connector.mysql.MySqlSnapshotChangeEventSource]
   2021-11-30 01:49:07,977 INFO   MySQL|dbserver1|snapshot  Snapshot ended with SnapshotResult [status=SKIPPED, offset=MySqlOffsetContext [sourceInfoSchema=Schema{io.debezium.connector.mysql.Source:STRUCT}, sourceInfo=SourceInfo [currentGtid=null, currentBinlogFilename=mysql-bin.000003, currentBinlogPosition=156, currentRowNumber=0, serverId=0, sourceTime=null, threadId=-1, currentQuery=null, tableIds=[], databaseName=null], snapshotCompleted=false, transactionContext=TransactionContext [currentTransactionId=null, perTableEventCount={}, totalEventCount=0], restartGtidSet=null, currentGtidSet=null, restartBinlogFilename=mysql-bin.000003, restartBinlogPosition=156, restartRowsToSkip=0, restartEventsToSkip=0, currentEventLengthInBytes=0, inTransaction=false, transactionId=null, incrementalSnapshotContext =IncrementalSnapshotContext [windowOpened=false, chunkEndPosition=null, dataCollectionsToSnapshot=[], lastEventKeySent=null, maximumKey=null]]]   [io.debezium.pipeline.ChangeEventSourceCoordinator]
   2021-11-30 01:49:07,981 INFO   MySQL|dbserver1|streaming  Requested thread factory for connector MySqlConnector, id = dbserver1 named = binlog-client   [io.debezium.util.Threads]
   2021-11-30 01:49:07,983 INFO   MySQL|dbserver1|streaming  Starting streaming   [io.debezium.pipeline.ChangeEventSourceCoordinator]
   ...
   ```

   这些行表明该服务在关闭之前找到了上一个任务之前记录的偏移量，连接到 MySQL 数据库，`binlog`从该位置开始读取，并从该时间点以来 MySQL 数据库中的任何更改生成事件。

4. 切换到运行的终端`watch-topic`以查看您在 Kafka Connect 离线时创建的两条新记录的事件：

   ```json
   {"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1005}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":true,"field":"version"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"server_id"},{"type":"int64","optional":false,"field":"ts_sec"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"boolean","optional":true,"field":"snapshot"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"db"},{"type":"string","optional":true,"field":"table"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope","version":1},"payload":{"before":null,"after":{"id":1005,"first_name":"Sarah","last_name":"Thompson","email":"kitt@acme.com"},"source":{"version":"1.9.6.Final","name":"dbserver1","server_id":223344,"ts_sec":1490635153,"gtid":null,"file":"mysql-bin.000003","pos":1046,"row":0,"snapshot":null,"thread":3,"db":"inventory","table":"customers"},"op":"c","ts_ms":1490635181455}}
   {"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1006}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":true,"field":"version"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"server_id"},{"type":"int64","optional":false,"field":"ts_sec"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"boolean","optional":true,"field":"snapshot"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"db"},{"type":"string","optional":true,"field":"table"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope","version":1},"payload":{"before":null,"after":{"id":1006,"first_name":"Kenneth","last_name":"Anderson","email":"kander@acme.com"},"source":{"version":"1.9.6.Final","name":"dbserver1","server_id":223344,"ts_sec":1490635160,"gtid":null,"file":"mysql-bin.000003","pos":1356,"row":0,"snapshot":null,"thread":3,"db":"inventory","table":"customers"},"op":"c","ts_ms":1490635181456}}
   ```

   这些事件是与您之前看到的类似的*创建事件。*如您所见，Debezium 仍然报告数据库中的所有更改，即使它没有运行（只要它在 MySQL 数据库从其`binlog`丢失的提交中清除之前重新启动）。

## 清理环境

完成本教程后，您可以使用 Docker 停止所有正在运行的容器。

程序

1. 停止每个容器：

   ```shell
   $ docker stop mysqlterm watcher connect mysql kafka zookeeper
   ```

   Docker 停止每个容器。因为您`--rm`在启动它们时使用了该选项，所以 Docker 也会删除它们。

|      | 如果您使用 Podman，请运行以下命令：`$ sudo podman pod kill dbz $ sudo podman pod rm dbz` |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

1. 验证所有进程是否已停止并已被删除：

   ```shell
   $ docker ps -a
   ```

   如果任何进程仍在运行，请使用或停止它们。`docker stop *<process-name>*``docker stop *<containerId>*`

## 下一步

完成本教程后，请考虑以下后续步骤：

- 进一步探索本教程。

  使用 MySQL 命令行客户端添加、修改和删除数据库表中的行，并查看对主题的影响。`watch-topic`您可能需要为每个主题运行单独的命令。请记住，您不能删除由外键引用的行。

- 尝试使用适用于 Postgres、MongoDB、SQL Server 和 Oracle 的 Debezium 连接器运行本教程。

  您可以使用位于[Debezium 示例存储库中的本教程的](https://github.com/debezium/debezium-examples/tree/main/tutorial)[Docker Compose](https://docs.docker.com/compose/)版本。为使用 MySQL、Postgres、MongoDB、SQL Server 和 Oracle 运行教程提供了 Docker Compose 文件。