---
layout: post
title:  "设计优先的 Restful API 服务开发与服务解耦实践"
date:   2016-09-20 16:35:00
author: Akagi201
comments: true
---

最近公司需要一个 API 服务提供给用户用来查询直播系统的一些流相关数据接口。正好我们公司应用组的同事 [CatTail](https://github.com/CatTail) 分享了他们在这个季度将他们的 API 服务进行了重构, 使用了 swagger 这套 OpenAPI 标准工具。正好借此机会, 学来用用。

## Swagger 框架的选择

应用组这边用的是 Node.js 开发的 API 服务, 我这边涉及到数据的上报与高并发处理, 使用 Go 开发。所以, 需要选择一个 Go 语言版的 Swagger API 框架

经过简单的调研比较后, 我选择了 [go-swagger](https://github.com/go-swagger/go-swagger) 这个框架。这个开源项目是由 VMware 赞助并维护的, 支持最新的 Swagger 2.0 标准。[项目的文档](https://goswagger.io)还不算太详细, 有的地方需要自己来摸索下。

我选择这个框架基于如下几点考虑:

* 支持标准的 http middleware, 方便集成 [Alice](https://github.com/justinas/alice) 插件管理工具, 可以直接使用大量与 `net/http` 兼容的各种 http middleware。
* 代码侵入性不大, 如果后期想改用其他框架, 迁移起来也不太麻烦。
* 知道 validator 功能, API spec 设计好后, 无需在代码中自己去写繁琐的边界校验功能。
* 可以通过代码生成 API 文档, 这样能 100% 的保证代码与文档保持一致。
* 由于使用 Swagger 的标准, 方便使用兼容这一标准的一整套工具链, 保证 API 监控, spec render 等, 也能复用应用组那边的资源, 哈哈。
* 项目也在发展中, 目前支持的 scheme 有 http 和 https, 后续会支持 ws 和 wss。我将来计划开发的 WebRTC 信令服务也可以考虑用这个框架来做。
* go-swagger 提供很多基础的 helper function 库实现: <https://github.com/go-openapi>

这个框架不太好的地方, 或者对于 Go 这种强类型语言不太好的地方, 就是 Swagger 基于 spec 定义的 definitions 生成的 model 与 数据库的 model 不能使用一套 struct, 因为涉及到一些 struct tag 的添加没法添加, 区分开后, 就涉及到两种结构之间的数据拷贝的问题, 代码显得有点冗长。暂时我还没想到更好的办法。

## 直播 API 服务架构设计

首先是整理需求, 和抽象需求的模型, 我这边收到几个类似的业务需求有 禁播/踢流 服务, 触发/定时截图, 触发/定时录制, 流事件回调与查询。都涉及到配置变更和推流与断流事件触发相应的业务逻辑。所以, 我抽象出了如下服务架构。

![origin_admin]({{ site.remoteurl }}/assets/origin_admin.png)

这个架构同时解决了两个问题:

* 与应用解耦, 在过去应用配置数据是通过 redis 数据结构与底层服务进行对接的, 当数据结构变更与扩展时, 都会发生各种各样繁琐的数据兼容与繁琐的流程。现在底层服务直接提供 Restful API 风格的 CRUD 操作, 底层服务自身来进行数据的持久化, 当需要存储结构调优和变更时, 就可以自己内部来调整, 而不用去动对外的接口。这在互联网环境, 需求与架构不断变化, 快速迭代开发的情况相匹配。
* 由于 CRUD 接口都在我的服务内部, 配置变更触发事件, 这个逻辑变得更加容易实现和控制。

统一的异步消息处理机制与事件回调机制:

* 配置变更 与 流事件触发, 可以用类似的异步逻辑来处理, 这里目前是通过 NSQ 消息队列, 来异步处理消费这些事件。
* 推流断流事件与配置变更事件可以联动触发其他服务的业务逻辑。这里我设计了统一的事件回调机制, 可以回调到录制服务, 截图服务, 踢流服务甚至客户自己的服务等。

## 说说技术选型
* swagger: OpenAPI 标准, 可以利用 swagger 一整套工具, 方便后续做 API 统计与监控。
* mq: NSQ, 简单, 够用, 稳定。
* db: mongodb, 文档存储, 支持大数据, 支持单个 field 的 CRUD 操作, 方便 scale, 够用的读写性能, 够用的查询功能。

## 通过 Swagger spec 来设计 API

这里截取一小部分来说明

```
swagger: "2.0"
info:
  contact:
    email: akagi201@gmail.com
    name: Akagi201
    url: http://akagi201.org
  description: UPYUN live streaming API service based on go-swagger
  title: UPYUN live streaming API service
  version: 0.1.0
# during dev, should point to your local machine
host: localhost:2201
# basePath will be prefixed to all paths
basePath: /api/v1
produces:
  - application/json
consumes:
  - application/json
schemes:
  - http
  - https
tags:
  - name: system
    description: 系统信息
  - name: stream
    description: 流信息
  - name: event
    description: 对内 事件 接口
  - name: config
    description: 对内 应用配置 接口
paths:
  /system/version:
    get:
      tags:
        - system
      summary: 获取版本信息
      operationId: getVersion
      responses:
        200:
          description: get versions
          schema:
            $ref: "#/definitions/version"
        default:
          description: error
          schema:
            $ref: "#/definitions/error_response"
definitions:
  version:
    type: object
    title: version_info
    required:
      - version
      - signature
    properties:
      version:
        type: string
        minLength: 1
      signature:
        type: string
        minLength: 1
  error_response:
    type: object
    title: error_response
    required:
      - code
      - data
    properties:
      code:
        type: integer
        format: int32
        description: 所有错误码统一定义
        example: 201
      data:
        type: string
        minLength: 1
        description: 详细错误信息
        example: Operator already exists
```

限制条件与文档说明, 例子都可以写到 spec 里面。完整的 spec 文档, 请参考 <http://swagger.io/specification/>

可以使用开源的 [swagger-ui](https://github.com/swagger-api/swagger-ui) 来 render 这份文档, 当然如果你觉得他太丑了, 可以自己写个 render。

![uplive_api]({{ site.remoteurl }}/assets/uplive_api.png)

## 使用 go-swagger 框架

目录结构

```
❯ tree -L 3
.
├── README.md
├── client // client SDK
│   ├── config
│   │   ├── config_client.go
│   │   ├── del_stream_config_parameters.go
│   │   ├── del_stream_config_responses.go
│   │   ├── get_stream_config_parameters.go
│   │   ├── get_stream_config_responses.go
│   │   ├── set_stream_config_parameters.go
│   │   └── set_stream_config_responses.go
│   ├── event
│   │   ├── event_client.go
│   │   ├── get_stream_events_parameters.go
│   │   ├── get_stream_events_responses.go
│   │   ├── stream_event_collector_parameters.go
│   │   └── stream_event_collector_responses.go
│   ├── origin_admin_client.go
│   ├── stream
│   │   ├── get_stream_status_parameters.go
│   │   ├── get_stream_status_responses.go
│   │   └── stream_client.go
│   └── system
│       ├── get_version_parameters.go
│       ├── get_version_responses.go
│       └── system_client.go
├── cmd // 最终生成的二进制
│   └── origin-admin-server
│       └── main.go
├── models // 根据 spec definition 生成的 model
│   ├── error_response.go
│   ├── result_response.go
│   ├── stream_action.go
│   ├── stream_event.go
│   ├── stream_events.go
│   ├── stream_info.go
│   ├── stream_status.go
│   └── version.go
├── restapi // api 处理逻辑
│   ├── configure_origin_admin.go // 自定义 API 逻辑入口文件
│   ├── controller.go // 我添加的针对每个 API 的具体处理 Handler 逻辑实现, 后期可以拆分成目录和多个文件的结构
│   ├── doc.go
│   ├── embedded_spec.go
│   ├── operations
│   │   ├── config
│   │   ├── event
│   │   ├── origin_admin_api.go
│   │   ├── stream
│   │   └── system
│   └── server.go
├── store // 与 mgo 交互的 model 定义
│   └── mongo
│       ├── event.go
│       └── stream_config.go
└── swagger.yml // spec 定义的地方
```

## 总结与体会

* 对于新人来说, 会有一定的学习成本, 包括开发理念的传递。不过 design first 这个理念是非常好的。
* 整体来说使用这套框架还是比较顺利, 没有遇到什么无法解决的问题, 也希望 [go-swagger](https://github.com/go-swagger/go-swagger) 能够不断完善起来, 另外, 大家用的多了, 自然功能也就完善了, 所以, 在此, 还是向大家隆重推荐一下。
