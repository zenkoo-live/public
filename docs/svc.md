# 服务端开发

---

## 服务列表

服务ID|服务名|说明|负责人
---|---|---|---
8001|svc.web.viewer|直播用户端BFF|王龙
8002|svc.web.streamer|主播端BFF|王龙
8003|svc.web.dashboard|管理后台BFF|商伟
1001|svc.infra.setting|配置服务|王龙
1002|svc.infra.static|静态资源存储|战旗团队
1003|svc.infra.notifier|通知服务（邮件/短信/第三方推送）|战旗团队
3001|svc.biz.account|账号服务|张浩
3002|svc.biz.gift|礼物服务|王雷云
3003|svc.biz.room|房间服务|王磊云
3004|svc.biz.org|组织|张浩
3005|svc.biz.relation|关系|王雷云
3006|svc.biz.log|操作日志|王磊云
3101|svc.biz.asset|资产服务|张志华
3102|svc.biz.trade|交易服务|张志华

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

        micro.Name(AppName+"::"+SvcName+runtime.AppendEnv()),
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

```go
type Manager struct {
    bun.BaseModel `bun:"table:account_managers"`
    
    ID         uuid.UUID       `bun:"id,pk,type:uuid,default:uuid_generate_v4()" json:"id"`
    MerchantID uuid.UUID       `bun:"merchant_id,type:uuid" json:"merchant_id"`
    Name       string          `bun:"name" json:"name"`
    Mobile     string          `bun:"mobile,unique" json:"mobile"`
    Email      string          `bun:"email,unique" json:"email"`
    Password   string          `bun:"password" json:"password"`
    Salt       string          `bun:"salt" json:"-"`
    Additions  json.RawMessage `bun:"additions,type:jsonb" json:"additions"`
    
    CreatedAt time.Time `bun:"created_at,nullzero,notnull,default:CURRENT_TIMESTAMP" json:"created_at"`
    UpdatedAt time.Time `bun:"updated_at,nullzero,notnull,default:CURRENT_TIMESTAMP" json:"updated_at"`
    DeletedAt time.Time `bun:"deleted_at,soft_delete,nullzero" json:"-"`
}
```

内部串联ID统一使用uuid。

如果需要自动创建数据表，在model文件中添加方法：

```go
func InitManager(ctx context.Context, dropTable bool) error {
    if dropTable {
        _, err := runtime.DB().NewDropTable().Model((*Manager)(nil)).Exec(ctx)
        if err != nil {
            logger.Error(err)
            
            return err
        }
    }
    
    _, err := runtime.DB().NewCreateTable().Model((*Manager)(nil)).Exec(ctx)
    if err != nil {
        logger.Error(err)
        
        return err
    }
    
    _, err = runtime.DB().NewCreateIndex().
    Model((*Manager)(nil)).
    Index("idx_account_managers_merchant_id").
    Column("merchant_id").Exec(ctx)
    if err != nil {
        logger.Error(err)
        
        return err
    }
    
    _, err = runtime.DB().NewCreateIndex().
    Model((*Manager)(nil)).
    Index("idx_account_managers_created_at").
    Column("created_at").Exec(ctx)
    if err != nil {
        logger.Error(err)
        
        return err
    }
    
    _, err = runtime.DB().NewCreateIndex().
    Model((*Manager)(nil)).
    Index("idx_account_managers_updatted_at").
    Column("updated_at").Exec(ctx)
    if err != nil {
        logger.Error(err)
        
        return err
    }
    
    _, err = runtime.DB().NewCreateIndex().
    Model((*Manager)(nil)).
    Index("idx_account_managers_deleted_at").
    Column("deleted_at").Exec(ctx)
    if err != nil {
        logger.Error(err)
        
        return err
    }
    
    logger.Info("Table <account_managers> created")
    
    // Insert first record
    salt := utils.RandomString(defs.SaltLength)
    admin := &Manager{
        MerchantID: defs.PlatformMerchantID,
        Name:       "zenkoo",
        Password:   utils.CryptoPassword("zenkoo", salt),
        Salt:       salt,
    }
    _, err = runtime.DB().NewInsert().Model(admin).Exec(ctx)
    if err != nil {
        logger.Error(err)
    } else {
        logger.Info("Administrator created")
    }
    
    return err
}
```

### 日志

### Proto

### 错误码

## 部署

### develop环境

Drone CI已配置，当push代码到develop分支时，drone将正常使用golang:alpine镜像编译，并将二进制文件scp到开发服务器上的/opt/zenkoo/*/bin中。

Drone在正确复制二进制制品文件后，远程使用开发服务器上的systemd重载对应的服务，如正确，在consul中可以看到对应服务。

一个unit文件的实例：

```properties
[Unit]
Description=zenkoo_svc.biz.account
After=network-online.target

[Service]
Type=simple
WorkingDirectory=/opt/zenkoo/svc.biz.account
Environment=ZENKOO_CONFIG_CONSUL_PREFIX=/zenkoo/config/svc.biz.account
ExecStart=/opt/zenkoo/svc.biz.account/bin/svc
PIDFile=/run/zenkoo/svc.biz.account.pid
Restart=always
User=ci
Group=ci

[Install]
WantedBy=multi-user.target
```

Unit文件位于开发服务器的/etc/systemd/system中，如果手动修改了unit文件，需要执行：

```bash
sudo systemctl daemon-reload
```

重载。

Develop环境中的服务，可以通过VPN直接访问。

### testing环境

Drone CI已配置，当push代码到testing分支时，drone将正常使用golang:alpine镜像编译，并使用项目根目录中定义的Dockerfile构建docker镜像，并推送到似有registry中（腾讯云香港）。

Drone在推送镜像后，尝试远程连接测试服务器，刷新swarm中的zenkoo stack（这里是要刷新zenkoo中的所有service）。

Testing环境中的服务，除bff外，不能直接远程访问。

### 调试

在开发服务器上，可以直接通过VPN互联的方式，调试本地启动的服务。如，直接在开发者本地的设备上启动服务：

```bash
ZENKOO_CONFIG_CONSUL_PREFIX=/zenkoo/config/svc.biz.account ZENKOO_CONFIG_CONSUL_ADDRESS=192.168.200.1:8500 bin/svc
```

在确保vpn连通且192.168.200.1（开发服务器的虚拟IP）可访问时，本地电脑的服务尝试连接远程的consul获取配置并启动服务。

这里也可以使用本地配置文件：

```bash
ZENKOO_CONFIG_FILE_PATH=config.json bin/svc
```

此时，需要正确配置base中的registry：

```json
{
    "registry": {
        "driver": "consul",
        "address": [ "192.168.200.1:8500" ]
    }
}
```

这样，服务就可以被正确注册到开发服务器的consul上。其他开发者可以正确调用到。

当本地电脑上有不止一个ip地址（连接vpn后这简直是一定的）时，go-micro可能会获取并注册了错误的服务地址，这时可以强制指定一个：

```bash
ZENKOO_CONFIG_CONSUL_PREFIX=/zenkoo/config/svc.biz.account ZENKOO_CONFIG_CONSUL_ADDRESS=192.168.200.1 MICRO_SERVER_ADDRESS=192.168.200.101:8888 bin/svc
```

指定正确的地址。需要注意的是，这个地址需要被consul和其他开发者直接访问到。

### 多环境

svc.base响应环境变量 **ZENKOO_ENV**，用来指定当前启动的服务环境副本。默认环境为空。环境会在服务注册时，附加在服务名后面，如：

```bash
ZENKOO_ENV=dev01 bin/svc
```

且服务入口中mirco.Name定义为：

```go
micro.Name(AppName+"::"+SvcName+runtime.AppendEnv())
```

将会创建服务名为“zenkoo::svc.biz.account::dev01”，并尝试注册。同时，调用方也响应同样的环境变量，当调用方的ZENKOO_ENV也定义为dev01时，调用svc.biz.account中的服务时，会尝试访问zenkoo::svc.biz.account::dev01，当调用方的ZENKOO_ENV为空时，尝试访问zenkoo::svc.biz.account。当调用方的ZENKOO_ENV为dev02时，将出现调用失败。

可以使用这个特性，在同一个consul上注册一个服务的多个副本，用于同时保持稳定版本的服务和调试中的服务共存。

这里推荐方式为，develop分支直接在开发服务器上启动一个稳定副本，并在需要时使用：

```bash
ZENKOO_ENV=dev ZENKOO_CONFIG_CONSUL_PREFIX=/zenkoo/config/svc.biz.account ZENKOO_CONFIG_CONSUL_ADDRESS=192.168.200.1:8500 MICRO_SERVER_ADDRESS=192.168.200.101:8888 bin/svc
```

临时启动本地电脑上的副本用于联调。由于使用了同样的ZENKOO_CONFIG_CONSUL_PREFIX，所以和稳定副本使用的数据库等是相同的。

## Tips

### VPN

开发服务器使用WireGuard实现开发者间的互联。Peer配置文件示例：

```properties
[Interface]
Address = 192.168.200.101/24
PrivateKey = {{你的私钥}}

[Peer]
PublicKey = LWt89hs5qAu1asksDFbYDvMG2endvw9VdpWndqPz9F8=  ## 这个是开发服务器的公钥
Endpoint = 82.156.35.99:37201
AllowedIPs = 192.168.200.0/24, 10.2.20.0/24
```

192.168.200.101是当前Peer希望连接到开发服务器后的内部IP，注意互相不要冲突。

在服务器上，添加对应peer的公钥：

```properties
[Peer]
# zhanghao
PublicKey = pvMwwwIX+xR9sx6/vILe802sX1sdT3vg9OauS1mz5g8=  ## 这是peer的公钥
AllowedIPs = 192.168.200.101/32
PersistentKeepalive = 25
```

生成密钥对的方式：

```bash
# 生成私钥
wg genkey > privatekey

# 计算公钥
wg pubkey < privatekey > publickey

# 同时生成
wg genkey | tee privatekey | wg pubkey > publickey
```

### go work

当有稳定的项目依赖，如服务依赖svc.proto和svc.base时，为方便本地开发调试，可以使用go work：

```bash
go work init
go work use .
go work use ../svc.base
go work use ../svc.proto
```

此时，调用对应包时，会选择go.work中定义的代码目录，而不是mod cache。但是，此时go.mod中对应的版本还是生效的。

不要将go.work和go.work.sum文件提交至git repo。但是这个时候，要注意更新go.mod中对应的包版本，否则ci可能会失败。

### 更新go包

```bash
go get -u github.com/zenkoo-live/svc.proto@latest
go get -u github.com/zenkoo-live/svc.base@latest
go mod tidy
```

然后再提交代码。
