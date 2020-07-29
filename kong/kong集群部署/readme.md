
在1.x版本中，kong的集群其实是通过运行多个实例，访问同一个数据库来实现的。具体表现为

- 定时轮询数据库，获取最新的 Services，Routes，Consumers，Plugins等信息，并缓存它们，直到下一次请求数据库时再更新数据
- 如果某个节点通过admin api对数据库中保存的代理配置进行更改，这个节点本身会立即生效，但其他节点需要等到下一次轮询时才会获取最新的数据



到了2.0版本，kong提供了 [混合模式](https://docs.konghq.com/2.0.x/hybrid-mode/) 来部署kong集群

在这种模式下，kong的节点被分为两种角色，分别是控制节点`CP`和数据节点`DP`


### 控制节点 CP
用于提供admin api，负责直接连接数据库并管理各种代理配置。它监听两个端口

- admin_listen (默认8001): 原来的admin api
- cluster_listen (默认8005): 用于与数据节点DP连接，提供最新的配置

### 数据节点 DP
提供代理的服务，但是代理配置不再从数据库直接获取，而是通过连接 CP 进行获取

监听端口
- proxy_listen (默认8000) ：提供代理服务

### 混合模式的优点

相较于1.x版本的集群，现在的混合模式有以下优点

- 减少数据库访问量：现在只有CP节点直接连接数据库
- 提高安全性：一个DP节点的服务器遭受入侵不会影响到其他的DP节点
- 易于管理：只需要通过CP节点就可以获取集群状态信息


## 安装步骤

### 1. 生成证书/秘钥对
首先生成证书/秘钥对保证CP与DP之间的通信安全

- 执行命令 `kong hybrid gen_cert` 在当前目录下生成`cluster.crt`和`cluster.key`这两个文件

将这两个文件传输到所有需要部署CP和DP节点的服务器上

### 2. 部署CP节点

- 复制默认配置
```
cd /etc/kong
cp kong.conf.default cp.conf
vim cp.conf
```

- 修改配置文件 
```sh
role = control_plane #指定为CP节点

#上一步生成的文件路径
cluster_cert = cluster.crt
cluster_cert_key = cluster.key

# 还需要指定数据库配置
# database = postgres
# pg_host =
# pg_password =
```

- 启动 
    - `kong start -c /etc/kong/cp.conf`
    - -c 后面填自己改的配置文件名

### 3. 部署DP节点

- 修改配置文件
```sh
role = data_plane #指定为DP节点
cluster_control_plane = CP的ip:8005
database = off

#第一步生成的文件路径
cluster_cert = cluster.crt
cluster_cert_key = cluster.key
lua_ssl_trusted_certificate = cluster.crt
```

- 启动 
    - `kong start -c /etc/kong/dp.conf`
    - -c 后面填自己改的配置文件名

备注： CP节点不能够再作为代理服务，如果需要在CP节点所在的服务器再部署一个DP节点的话，需要更改配置 `prefix` 来区分它们的工作目录（它们的端口号不冲突，无需修改）


### 4. 检查集群状态
- 访问 `http://{{cp节点ip}}:8001/clustering/status` 就可以查看当前集群中有几个数据节点加入了

>   混合模式部署的官方文档地址 https://docs.konghq.com/2.0.x/hybrid-mode/