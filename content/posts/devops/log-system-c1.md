---
author: "Kreedzt"
title: "日志管理系统本地搭建(一)"
date: "2023-08-05"
description: "基于 Docker, 使用 Grafana + loki + fluent-bit + MinIO 搭建简易的日志管理系统"
tags: ["devops", "docker", "Grafana", "loki", "fluent-bit", "minio"]
draft: false
---

## 目标

### 各软件包作用

- fluent-bit: 负责收集日志, 处理数据, 转发到 loki 中
- loki: 负责筛选并存储数据到 MinIO 中
- Grafana: 负责提供 loki web 操作面板
- MinIO: 存储 loki 处理的日志数据

### 处理流程

上报:
1. Docker container 上报日志到 fluent-bit 中
2. fluent-bit 采集数据转发到 loki 中
3. loki 处理日志数据, 索引与数据保存到 MinIO 中

查询:
1. Grafana: 进入 loki 面板, 通过标签筛选
2. loki 提取 MinIO 数据进行查询

## 环境准备

- [Docker](https://docs.docker.com/engine/install/)
- [Docker compose](https://docs.docker.com/compose/install/)

### 测试安装 Grafana 与 Loki

> Grafana Loki 章节的安装已经包含了 Grafana, 直接看 loki 章节即可

[官方文档](https://grafana.com/docs/loki/latest/installation/docker/)

按照 install with Docker compose 教程, 可以进行本地直接初次搭建测试:

```sh
wget https://raw.githubusercontent.com/grafana/loki/v2.8.0/production/docker-compose.yaml -O docker-compose.yaml
# 对于新版 docker compose cli, compose 作为 docker 子命令而非 docker-compose 命令
docker compose up
```

搭建完成后, 默认 Grafana web 访问端口为 3000, 访问后点击侧边栏: `Home > Explore`, 选择 `Loki` 即可进入 Loki 查询面板

通过官方的搭建文件, 我们会存在一个已有的上报数据, 标签为 `job=varlogs`, 可以通过 `label filters` 进行筛选

![grafana-1](../images/grafana-1.png)


在以上步骤执行成功后, 我们可以 `Ctrl-C`关闭此程序执行, 下面进行其他环境配置


### 外置安装 MinIO

> 本教程的 MinIO 为外置存储, 并不是与采集环境同属一台机器

[安装](https://min.io/docs/minio/container/index.html)

使用 MinIO 是为方便 Loki 存储数据, 作为 Loki 的外置存储环境

按照文档在外置设备安装 MinIO 后, 可以通过 9090 端口访问 web 管理后台.

#### 创建用户(可选)

在 `Administator -> Identity -> Users` 中创建 loki 子用户, 在 `Service Accounts` 中创建 AccessKey 以及 Secret Key, 以便其他设备通过 AccessKey 访问存储 bucket


#### 创建 Bucket

1. 进入到 Administrator -> Bukets 菜单中, 创建名为 `loki` 的 bucket, 我们在后续访问时需要填上此处的 bucket 名称, 所有 loki 的日志索引及数据都会存储到此 bucket 中
2. 点击 loki, 进入到 bucket 配置面板
3. 为访问安全, 将 Access Policy 更改为 private

此时的配置效果图应该如下所示:

> 注意: 此时的占用空间应该是 0, 此时因为我的机器上已经有日志数据存储了

![minio-1](../images/minio-1.png)

#### 访问测试(建议)

[mc](https://min.io/docs/minio/linux/reference/minio-mc.html?ref=docs) 是最简单的访问 MinIO 存储数据的命令行工具, 可以用来测试能否正常访问存储桶

如果配置正常, 如下命令应该正常访问并输出存储桶状态:

输入:
```sh
# 访问可以通过 admin 账号密码访问, 也可通过有权限的 AccessKey 与 SecretKey 访问, 此处我们使用 AccessKey 与 SecrectKey 访问
mc alias set loki http://192.168.2.205 kkkkkkk sssssss
```

输出:
```sh
Added `loki` successfully.
```

输入:
```sh
mc admin info loki
```

输出:
```sh
●  192.168.2.205:9000
   Uptime: 1 day
   Version: 2023-07-21T21:12:44Z
   Network: 1/1 OK
   Drives: 1/1 OK
   Pool: 1
```


### 测试安装 fluent-bit

[安装](https://docs.fluentbit.io/manual/installation/getting-started-with-fluent-bit)

#### HTTP 采集测试

[文档](https://docs.fluentbit.io/manual/pipeline/inputs/http)

可以按照教程通过 http 进行日志采集

本地使用 postman / curl 等工具发送 HTTP 请求, 检查 fluent-bit 日志输出, 以检测采集是否成功


#### 多输入 / 多输出

fluent-bit 可以通过 tag 名称来设置多输入多输出, 该章节在 [Router](https://docs.fluentbit.io/manual/concepts/data-pipeline/router) 中有所提及

在后续实际配置中, 也将采用多输入多输出配置

```conf
[INPUT]
    Name cpu
    Tag  my_cpu

[INPUT]
    Name mem
    Tag  my_mem

[OUTPUT]
    Name   es
    Match  my_cpu

[OUTPUT]
    Name   stdout
    Match  my_mem
```

以上的配置文件表明通过 tag 为 `my_cpu` 的采集日志输出到 `es` 服务中, 通过 tag 为 `my_mem` 的采集日志输出到 `stdout` 中

## 环境搭建

### 获取 Grafana Loki 环境基础配置

通过 [Docker compose 安装](https://grafana.com/docs/loki/latest/installation/docker/#install-with-docker-compose) 文档获取第一步连接

我们此时仅下载配置文件, 不直接执行.

> 以下内容具有时效性, 以官方文档最新内容为准

执行如下命令:
```sh
wget https://raw.githubusercontent.com/grafana/loki/v2.8.0/production/docker-compose.yaml -O docker-compose.yaml
```

### 配置其他环境

#### 创建 config 目录, 存储配置

在当前目录创建 `config` 文件夹, 用以存储 fluent-bit 及 loki 配置

目录结构:
```tree
.
├── config
│   ├── fluentbit.config
│   └── loki-config.yaml
└── docker-compose.yaml
```


#### 配置 fluent-bit

我们配置 2 个输入: 默认的 [Forward](https://docs.fluentbit.io/manual/pipeline/inputs/forward) 加上 [HTTP](https://docs.fluentbit.io/manual/pipeline/inputs/http) 作为输入源

将他们都输出到 loki, 但是输出的 labes 不同

> 使用 forward 可以适配 [fluentd log driver](https://docs.docker.com/config/containers/logging/fluentd/), 且 log driver 已经处理好了输出格式, 无需额外复杂配置

```conf
[INPUT]
    name forward
    listen  0.0.0.0
    port 24224
    tag container

[INPUT]
    name http
    listen 0.0.0.0
    port 9900
    tag http

[OUTPUT]
    name loki
    match container
    labels job=fluentbit
    # 按照官方文档获取的 docker compose, 此处的 host直接填写 loki 即可
    host loki

[OUTPUT]
    name loki
    match http
    labels job=fluentbit_http, $sub['stream']
    # 按照官方文档获取的 docker compose, 此处的 host直接填写 loki 即可
    host loki
```
配置效果如下:

- http input -> ouptut labels: `job=fluentbit_http, $sub['stream']`
- forward input -> ouptut labels: `job=fluentbit`

#### 配置 loki

按照 [本地部署 S3配置](https://grafana.com/docs/loki/latest/configuration/examples/#2-s3-cluster-exampleyaml), 可得到如下基础配置:

> 以下内容具有时效性, 以官方文档最新内容为准

```yaml

# This is a complete configuration to deploy Loki backed by a s3-Comaptible API
# like MinIO for storage. Loki components will use memberlist ring to shard and
# the index will be shipped to storage via boltdb-shipper.

auth_enabled: false

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: memberlist
  replication_factor: 1
  path_prefix: /loki # Update this accordingly, data will be stored here.

memberlist:
  join_members:
  # You can use a headless k8s service for all distributor, ingester and querier components.
  - loki-gossip-ring.loki.svc.cluster.local:7946 # :7946 is the default memberlist port.

schema_config:
  configs:
  - from: 2020-05-15
    store: boltdb-shipper
    object_store: s3
    schema: v11
    index:
      prefix: index_
      period: 24h

storage_config:
 boltdb_shipper:
   active_index_directory: /loki/index
   cache_location: /loki/index_cache
   shared_store: s3
 aws:
   s3: s3://access_key:secret_access_key@custom_endpoint/bucket_name
   s3forcepathstyle: true

compactor:
  working_directory: /loki/compactor
  shared_store: s3
  compaction_interval: 5m
```

我们重点需要修改如下几项:
- memberlist.join_members: 我们使用的 docker compose 部署, 此处的 k8s 服务名可以更换为本地
- storage_config.aws.s3: 我们使用的 MinIO 地址, 此处的地址需要手动替换

**注意**: s3 这一项配置项有 2 种方式
+ 按照 [MinIO 文档](https://blog.min.io/how-to-grafana-loki-minio/), 账号密码访问: 该访问方式的协议为 HTTP: `http://admin:password@192.168.2.205:9000/loki`
> 若 MinIO 也在本机部署, 且镜像配置写入在本地 docker compose 配置中, 可直接使用账号密码访问, 免去创建密钥问题
+ 按照 Loki 文档, 通过 AccessKey 与 SecretKey 访问:  该访问方式的协议为 s3: `s3://key:secrect@192.168.2.205:9000/loki`

修改后的内容如下:
```diff

# This is a complete configuration to deploy Loki backed by a s3-Comaptible API
# like MinIO for storage. Loki components will use memberlist ring to shard and
# the index will be shipped to storage via boltdb-shipper.

auth_enabled: false

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: memberlist
  replication_factor: 1
  path_prefix: /loki # Update this accordingly, data will be stored here.

memberlist:
  join_members:
  # You can use a headless k8s service for all distributor, ingester and querier components.
-  - loki-gossip-ring.loki.svc.cluster.local:7946 # :7946 is the default memberlist port.
+  # 直接使用本地服务名
+  - loki:7946 # :7946 is the default memberlist port.

schema_config:
  configs:
  - from: 2020-05-15
    store: boltdb-shipper
    object_store: s3
    schema: v11
    index:
      prefix: index_
      period: 24h

storage_config:
 boltdb_shipper:
   active_index_directory: /loki/index
   cache_location: /loki/index_cache
   shared_store: s3
 aws:
-   s3: s3://access_key:secret_access_key@custom_endpoint/bucket_name
+   # loki 为先前在 MinIO 创建的存储桶(bucket)名称
+   s3: s3://key:secrect@192.168.2.205:9000/loki
   s3forcepathstyle: true

compactor:
  working_directory: /loki/compactor
  shared_store: s3
  compaction_interval: 5m
```

#### 配置 docker compose

1. 添加 fluentbit 服务配置:

按照 [logging-pipeline](https://docs.fluentbit.io/manual/local-testing/logging-pipeline#docker-compose) 文档, 挂载正确的配置路径

```yaml
services:
  fluentbit:
    image: fluent/fluent-bit:2.1.8
    ports:
      # 此处的暴露端口对应之前 fluent-bit 的配置文件
      - "9900:9900"
      - "24224:24224"
    volumes:
      - ./config/fluentbit.config:/fluent-bit/etc/fluent-bit.conf
    networks:
      - loki
```

2. 挂载 loki 配置文件:

增加 `volumes` 挂载及调整 `command` 执行参数即可

```yaml
services:
  loki:
    image: grafana/loki:2.8.0
    ports:
      - "3100:3100"
    volumes:
      - ./config/loki-config.yaml:/etc/loki/loki.yaml
    command: -config.file=/etc/loki/loki.yaml
    networks:
      - loki
```


#### 启动服务并测试

当前目录执行 docker compose 启动
```sh
docker compose up -d
```

此时执行 `docker ps | grep "grafana"``, 应该输出如下几个服务:
```
cbdfe4a3c3db   fluent/fluent-bit:2.1.8                  "/fluent-bit/bin/flu…"   6 hours ago    Up 6 hours                      0.0.0.0:9900->9900/tcp, :::9900->9900/tcp, 2020/tcp, 0.0.0.0:24224->24224/tcp, :::24224->24224/tcp   grafana-fluentbit-1
4086af46e74b   grafana/grafana:latest                   "sh -euc 'mkdir -p /…"   23 hours ago   Up 23 hours                     0.0.0.0:3000->3000/tcp, :::3000->3000/tcp                                                            grafana-grafana-1
4426d21ae82b   grafana/promtail:2.8.0                   "/usr/bin/promtail -…"   23 hours ago   Up 23 hours                                                                                                                          grafana-promtail-1
a39128164cf9   grafana/loki:2.8.0                       "/usr/bin/loki -conf…"   23 hours ago   Up 23 hours                     0.0.0.0:3100->3100/tcp, :::3100->3100/tcp                                                            grafana-loki-1
```

等待数分钟, 此时 MinIO 的 loki 存储桶所占空间应该会有所增长, 若没有增长, 检查 `loki` 服务日志(多半情况下为 MinIO 存储桶无法访问, 进行 [访问测试(建议)](#访问测试建议) 小结排查问题)

#### fluent 上报测试

我们通过 docker run 执行 hello-world 镜像配合 [fluentd log driver](https://docs.docker.com/config/containers/logging/fluentd/) 来进行上报测试

输入命令启动 container:
```sh
docker run --log-driver=fluentd --log-opt fluentd-address=localhost:24224 hello-world
```

此时去 Grafana 查询 job=fluentbit 的日志内容应该存在以下输出

![fluentd log driver](../images/fluentd-log-driver-1.png)