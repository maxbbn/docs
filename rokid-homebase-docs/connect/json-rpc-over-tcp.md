# Homebase over TCP

## 协议描述

通讯基于 [JSON-RPC 2.0](https://www.jsonrpc.org/specification)，使用 TCP 短连接

### 请求格式

```json
{"jsonrpc": "2.0", "method": "subtract", "params": [42, 23], "id": 1}
```

### 成功返回

- `jsonrpc` {string} 必须为 "2.0"
- `id` {string} 必须与请求对象中的id一致
- `result` {any} 执行结果, 根据接口类型返回不同的结果

示例如下：


```json
{ "jsonrpc": "2.0", "result": "some data", "id": "1" }
```

### 错误返回 

- `jsonrpc` {string} 必须为"2.0"
- `id` {string} 必须与请求对象中的id一致
- `error` {object} 错误对象
- `error.code` {int} 错误代码
- `error.message` {string} 错误描述
- `error.data` {string} 错误名称, (errorName)

错误名称需使用指定值，具体请参考 [标准错误](../v1/errors.md)， 示例如下：

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": 10001,
    "message": "error reason",
    "data": "E_DRIVER_ERROR"
  },
  "id": "1"
}
```

## 调用过程

- Homebase 获取到 TCP 驱动服务 IP地址 与 端口
- Homebase 建立与该驱动服务的连接
- Homebase 发送 jsonrpc 请求
- 设备收到 Homebase 发送 TCP 的 FIN 包代表 Homebase 的数据已全部发送完毕，可以开始解析数据并做相应的处理
- Homebase 接收响应， 并解析。接收超时时间 10s
- 断开本次连接

## 指令列表

### 设备搜索

方法名: `list`

获取设备列表

#### 参数说明

- params
  - {Object} userAuth
    - userId
    - userToken

#### 返回说明

返回设备列表， 标准设备接口参考 [Homebase 设备][device]

- {Array} result
  - {String} deviceId
  - {Object} deviceInfo
  - {String} state
  - actions
  

#### 示例：

接口请求：

```json
{"jsonrpc": "2.0", "method": "list", "params": {"userAuth":{ "userId": "hello1234" }}, "id": "1"}
```

接口响应：

```json
{
   "jsonrpc":"2.0",
   "result":[
      {
         "deviceId":"test-id",
         "name":"测试设备",
         "type":"light",
         "roomName": "办公室",
         "homeName": "公司",
         "actions":{
            "switch":[
               "on",
               "off"
            ]
         },
         "state":{
            "switch":"on"
         },
         "offline":false
      }
   ],
   "id":1
}
```

### 设备控制指令执行

方法名: `execute`



#### 参数说明

- params
  - userAuth， 可选
    - userId
    - userToken
  - device
    - deviceId
    - deviceInfo
    - state
  - action
    - property  eg: 'color'
    - name  eg: 'num'
    - value eg: 0xff0000

#### 返回说明

- 可以返回设备最新 state
- 返回 null 会触发一次 get 请求来更新最新设备状态
- result

示例：

接口请求

```json
{ "jsonrpc": "2.0", "method": "list", "params": {"userAuth":{ "userId": "hello1234" }, "device": {"deviceId": "abc"}, "action": {"property": "switch", "name": "on"}}, "id": "1" }
```


响应时返回设备最新 state, 可以更新设备状态

接口响应：

```json
{ "jsonrpc": "2.0", "result": {"switch": "on"}, "id": "1"}
```

返回设备描述查看如下文档


[device]: ../v1/device/device.md
