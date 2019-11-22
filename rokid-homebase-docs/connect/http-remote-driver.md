# HTTP 远程驱动（旧版协议）

HTTP 远程驱动是接入 Homebase 推荐的方式，云对云驱动也使用此协议。

你可以通过 开发者驱动， 或 [rhome 命令][rhome] 来开发和调试 HTTP Driver

开发一个 HTTP 远程驱动你可能需要以下 3 个接口

- `/list` 获取设备列表
- `/execute` 在设备上执行动作
- `/command` （可选）用于处理用户授权，绑定，初始化等命令；相对于使用这个授权方式，我们更加推荐使用[标准 OAuth2.0 接入](rfc6749.md)


## 通用消息格式

HTTP 远程驱动使用 JSON 作为数据交换格式， 执行成功会返回如下字段：

### **成功返回**

- `status` {int} 成功返回必须为 0
- `data` {any} 执行结果, 根据接口类型返回不同的结果


```json
{
  "status": 0,
  "data": {
    "some": "data"
  }
}
```

### 错误返回

- `status` {int} 必须 大于0 的正整数
- `errorName` {string} 可选， 若琪定义的错误名
- `vendorErrorCode` {string} 可选， 厂家自有错误码
- `message` {string} status > 0 时必选；错误描述, 对错误的简短描述
- `debugInfo` {string} 可选， 调试信息

```json
{
  "status": 1,
  "errorName": "E_DRIVER_ERROR",
  "vendorErrorCode": "10001",
  "message": "time out",
  "debugInfo": "some info for debug"
}
```

## 标准接口

### 设备搜索

`POST /list` 

搜索给定用户账号下的所有设备， 包括虚拟设备， 子设备

#### 参数

- userAuth {Object} 用户授权信息

```json
{
  "userAuth": {
    "userId": "",
    "userToken": ""
  }
}
```

#### 返回值

标准化后的 [Device][device] 列表 { Array }：

示例

```json
{
  "status": 0,
    "data": [
      {
        "type": "light",
        "deviceId": "demoDevcie1",
        "name": "灯灯灯灯",
        "roomName": "客厅",
        "homeName": "我的家",
        "actions": {
          "switch": ["on", "off"]
        },
        "state": {
          "switch": "off"
        }
      },
      {
        "type": "light",
        "deviceId": "demoDevcie2",
        "name": "灯灯灯灯",
        "roomName": "办公室",
        "homeName": "公司",
        "actions": {
          "switch": ["on", "off"]
        },
        "state": {
          "switch": "off"
        }
      }
    ]
}
```

### 执行操作指令

`POST /execute` 

设备执行操作， 将返回最新状态

#### 参数

- device `{Object}`
- device.deviceId  `{String}`, 厂商设备ID， 可以包含不可变数据， 与厂商ID一起，可以唯一确认一台设备
- device.state  `{Object}`, 获取设备当前状态
- device.userAuth `{Object}`, 用户授权信息
- action `{Object}` 需要执行的操作， 参考文档 [actions and state](../v1/device/actions-and-state.md)


request

```json
{
  "device": {
     "deviceId": "xxx",
     "state": {
        "switch": "on"
     },
     "userAuth": {
        "userId": "",
        "userToken": ""
     }
  },
  "action": {
    "property": "switch",
    "name": "on"
  }
}
```

response

```json

{
  "status": 0,
  "data": {
    "switch": "on"
  }
}

```

#### 返回值

Object 设备最新状态， 如果无法确定设备的状态， 返回字段可以置空， 返回的设备状态将被缓存到虚拟设备中去

### 非标准授权指令

`POST /command` 

command 接口主要面向非标准的授权方式， 如 用户名密码登录， 非标准 Oauth 授权目前支持三种授权方式

- 用户名，密码登录： 提供 `login` command
- 非标准Oauth 授权： 提供 OAuth, OAuthGetToken, OAuthRefresh 接口
- 非标准OAuth2.0 授权: 提供 OAuth, OAuthGetCode, OAuthGetToken, OAuthRefresh 接口

#### 参数

- command `{String}`
- params  `{Object}`

request sample

```json
{
  "command": "login",
  "params": {
    "username": "superman",
    "password": "xxxxxx"
  }
}
```

#### 返回

返回 data 的数据格式和请求对应

```json

{
  "status": 0,
  "data": {
    "userId": "superman",
    "userToken": "DFGHJKLCVBNMFGHJKL"
  }
}

```


### 常用指令 Command

##### Command "OAuth"

OAuth 授权方式必须提供 OAuth Command， 输入回调地址， 返回 OAuth 登陆地址

- 返回的地址必须可以在浏览器里面打开
- 地址里面包含 callbackURL ， 注意需要 encode
- 用户授权完成， 需要重定向到 `callbackURL`, url 中添加 OAuthRefresh

params:

- `callbackURL` 授权成功后的回调页面

request

```json
{
  "command": "OAuth",
  "params": {
    "callbackURL": "https://homebase.rokid.com/bar?a=1"
  }
}

```

returns { String } 登录跳转 URL

response
```json
{
  "status": 0,
  "data": "https://oauth.yourbrand.com?redirect=http%3A%2F%2Fhomebase.rokid.com%2Fbar%3Fa%3D1"
}
```

##### Command "OAuthRefresh"

OAuth 授权方式必须要有， 输入回调地址， 返回 OAuth 登陆地址

params:

- `userId` 用户Id
- `userToken` 用户Token
- `refreshToken` 用户的 refreshToken， 可选

```json
{
    "userId": "superman",
    "userToken": "fjfjfjfjfjfj"
}
```

returns result <Object>

- result.userId {可选}
- result.userToken 新的用户Token
- result.expiredTime 更新过期时间， 秒为单位， UTC 时间戳
- result.refreshToken  可选， 更新 refreshToken

```json
{
  "status": 0,
  "data": {
    "userToken": "",
    "expiredTime": ""
  }
}
```

##### Command "login"

params:

- username 用户Id
- password 密码

```json
{
    "username": "superman",
    "password": "superman_password"
}
```

returns result <Object>

- result.userToken {string}
- result.userId {string} // 可选

```json
{
  "status": 0,
  "data": {
    "userToken": "userToken"
  }
}
```



## 如何描述你的设备？

请参考： [设备定义][device]


[device]: ../v1/device/device.md
[rhome]: ../tools/rhome.md

##### 可以参考我们的示例程序

[Rokid/homebase-nodejs-http-remote-driver-sample](https://github.com/Rokid/homebase-nodejs-http-remote-driver-sample)