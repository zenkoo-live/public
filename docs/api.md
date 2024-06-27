# API

## BFFs

* svc.web.viewer
* svc.web.streamer
* svc.web.dashboard

## 错误码

http Envelope中定义的错误码格式为：

```text
SSSS-HHH-DDD
```

其中前四位为服务编号，参考svc.md。中间三位对应HTTP Status,最后三位为序号。

code | static message | service | 含义 | 说明
---|---|---|---|---
0|OK|-|正常|-
9999400000|Parse HTTP body failed|-|HTTP请求body解析错误|-
9999400001|Validate HTTP request failed|-|HTTP入参数据验证错误|validator返回错误
9999401001|Auth failed|-|HTTP认证失败|可能发生在登录、或session middleware等
9999500001|Storage failed|-|存储错误|内部错误，可能是数据库或其它IO错误
9999999999|General failed|-|普通（不属于其它可描述的）错误|fallback，实在不知道报啥了
