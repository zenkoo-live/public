# 服务端开发

---

## 服务列表

服务ID|服务名|说明|负责人
---|---|---|---
8001|svc.web.viewer|直播用户端BFF|商伟
8002|svc.web.streamer|主播端BFF|商伟
8003|svc.web.dashboard|管理后台BFF|商伟
1001|svc.infra.setting|配置服务|王龙
1002|svc.infra.static|静态资源存储|张浩
1003|svc.infra.notifier|通知服务（邮件/短信/第三方推送）|王龙
3001|svc.biz.account|账号服务|张浩
3002|svc.biz.gift|礼物服务|王雷云
3003|svc.biz.room|房间服务|王磊云
3101|svc.biz.asset|资产服务|张志华
3102|svc.biz.trade|订单服务|张志华

## 添加服务流程

### 创建项目代码仓库

根据如下命名规范创建项目：

* svc.infra.* - 业务无关的基建服务
* svc.biz.* - 业务服务
* svc.data.* - 需要多个服务共享的数据抽象服务
* svc.web.* - BFF

如：svc.biz.union，用于工会相关业务

### 项目源码目录结构

```text
-----
  |-- main.go  // 入口
  |-- Makefile
  |-- Dockerfile
  |-- .drone.yml
  |-- .gitignore
  |-- handler  // HTTP或RPC handler
        |-- request  // HTTP请求定义
        |-- response  // HTTP返回定义
  |-- service  // 业务逻辑
  |-- model  // 数据模型
  |-- utils
```

### 入口服务

```go
    svc := micro.NewService(
        micro.Server(
            srvGrpc.NewServer(
                server.Name(AppName+"::"+SvcName+"@grpc"),
            ),
        ),

        micro.Name(AppName+"::"+SvcName),
        micro.Version(Version),
        micro.Broker(
            runtime.Broker(),
        ),
        micro.Registry(
            runtime.Registry(),
        ),
        micro.Cache(
            runtime.Cache(),
        ),
    )
```

即使BFF服务，也会正常初始化一个grpc service。

### HTTP Endpoint

对于BFF,svc.base中初始化了由go-fiber驱动的HTTP服务，这需要在配置中正确配置了fiber：

```go
    fiberApp := runtime.Fiber()
    if fiberApp != nil {
        // HTTP Handler
        handler.InitMisc(fiberApp)  // 这里初始化所需的handlers
    }

    svc.Init(
        micro.BeforeStart(func() error {
            runtime.StartHTTP()

            return nil
        }),
        micro.BeforeStop(func() error {
            runtime.StopHTTP()

            return nil
        }),
    )
```

HTTP服务不会注册到微服务registry,这与go-micro的HTTP server不同。

### 配置

配置文件和远程config可以同时被支持，svc.base通过环境变量决定配置加载源。当配置了 **ZENKOO_CONFIG_FILE_PATH** 环境变量时，服务会加载对应的配置文件，json和yaml格式均被支持（基于go-micro的config file source）。如：

```shell
ZENKOO_CONFIG_FILE_PATH=config.json bin/svc
```

将尝试加载配置文件config.json。

当配置了 **ZENKOO_CONFIG_CONSUL_PREFIX** 环境变量时，服务会从consul的KV存储中加载配置数据。此时，默认的consul地址为localhost:8500。可以通过 **ZENKOO_CONFIG_CONSUL_ADDRESS** 配置consul地址：

```shell
ZENKOO_CONFIG_CONSUL_PREFIX=/zenkoo/config/svc.biz.account bin/svc

ZENKOO_CONFIG_CONSUL_ADDRESS=172.18.0.8 ZENKOO_CONFIG_CONSUL_PREFIX=/zenkoo/config/svc.biz.union bin/svc
```

#### base配置项

svc.base自动scan配置原中的子集“base”，base支持如下格式：

```json
{
  "registry": {  // 注册中心，如忽略此项，go-micro使用本地的mdns进行服务注册。
    "driver": "consul",  // 支持consul和etcd。
    "address": [ "localhost:8500" ]  // 相应的注册中心地址。
  },
  "broker": {  // 消息订阅，如忽略此项，go-micro使用HTTP进行消息通讯。
    "driver": "nats",  // 支持nats，kafka，rabbitmq。
    "address": [ "localhost:4222" ]  // broker地址。
  },
  "redis": {  // Redis配置，与cache中的redis不同。如忽略此项，不创建redis连接。
    "address": "localhost:6379",  // Redis地址。
    "db": 0,  // Redis库。
    "password": "",  // Redis连接密码，默认为空。
    "max_retries":  3,  // 重试次数，默认为3，-1为关闭重试。
    "pool_size":  10,  // 连接池大小，默认为10.
  },
  "mongo": {  // MongoDB配置，如忽略此项，不创建mongo连接。
    "dsn": "mongodb://root:123456@localhost:27017/zenkoo"
  },
  "database": {  // 数据库配置，这里使用uptrace/bun。如忽略此项，不创建数据库连接。
    "driver": "postgres",  // 支持postgres，mysql，mssql，sqlite。
    "dsn": "postgres://postgres:123456@localhost:5432/zenkoo?sslmode=disable"
  },
  "logger": {  // 日志配置。
    "debug": true,  // 是否将日志级别从info设置为debug，默认为false。
    "silence": false,  // 是否关闭stdout和stderr回显，默认问为false。
    "discard_disk": false,  // 是否关闭写盘，默认为false。
    "base_path": "./svc_logs"  // 日志落盘路径，默认为./logs。
  },
  "fiber": {  // Fiber HTTP配置，如忽略此项，不创建HTTP服务。
    "address": "127.0.0.1:8080",  // HTTP监听地址，默认为:9990。
    "strict_routing": false,  // 严格匹配路由，当配置为true时，/foo和/foo/将被识别为不同的路由，默认为false。
    "case_sentitive": true,  // 路由中是否区分大小写，默认为false。
    "etag": false,  // 是否启用header中的etag generator，默认为false。
    "body_limit": 100000,  // 请求body的最大值，默认为4MB。
    "concurrency": 100000,  // 并发请求数量，默认为256K。
    "read_buffer_size": 100000,  // 读缓存大小。
    "write_buffer_size": 100000,  // 写缓存大小。
    "disable_keep_alive": false,  // 关闭Keep-Alive，默认为false。
    "enable_swagger": true,  // 启用/docs为SwaggerUI，默认为false。
    "enable_stack_trace": false  // 启用go stack trace，默认为false。
  }
}
```

### ORM Model

### 日志

### Proto

### 错误码

## 部署
