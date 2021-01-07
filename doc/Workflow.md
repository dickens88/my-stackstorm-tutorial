# Workflows

Workflows用于将action串联起来，可以通过条件判断逻辑来编排action，实际上Workflows也是一种Action，只是`runner_type`会有不同。根据v3.3官方文档的描述，当前版支持两种workflow：[Orquesta](https://docs.stackstorm.com/orquesta/index.html)和[ActionChain](https://docs.stackstorm.com/actionchain.html)，mistral-v2已经不再支持，并且后续StackStorm将只支持Orquesta，所以本例我们将使用Orquesta脚本

本例我们将演示创建一个workflow来获取NASA APOD图片URL然后将其发送到RabbitMQ队列

## 创建workflow action元数据

Workflow Action的元数据文件与其他Action的元数据文件没有太大不同，这里`runner_type: orquesta`，入参与python action类似，`entry_point`配置成workflow定义的YAML文件(相对于pack的actions的路径)

> 查看或修改`/opt/stackstorm/packs/tutorial/actions/nasa_apod_rabbitmq_publish.yaml` 

``` yaml
---
name: nasa_apod_rabbitmq_publish
pack: tutorial
description: "Queries NASA's APOD (Astronomy Picture Of the Day) API to get the link to the picture of the day, then publishes that link to a RabbitMQ queue"
runner_type: orquesta
enabled: true
entry_point: workflows/nasa_apod_rabbitmq_publish.yaml
parameters:
  date:
    type: string
    description: "The date [YYYY-MM-DD] of the APOD image to retrieve."
  message:
    type: string
    default: "halo"
    description: "Extra message to publish with the URL"
  host:
    type: string
    default: "10.0.2.15"
  port:
    type: integer
    default: 5672
  exchange:
    type: string
    default: "demo"
    description: "Name of the RabbitMQ exchange"
  exchange_type:
    type: string
    default: "topic"
    description: "Type of the RabbitMQ exchange"
  exchange_durable:
    type: boolean
    default: false
  routing_key:
    type: string
    default: "demokey"
    description: "Name of the RabbitMQ routing key"
  username:
    type: string
    default: "admin"
    description: "user name of rabbitmq"
  password:
    type: string
    default: "admin"
    description: "password of the rabbitmq"

```

最后要把元数据文件注册一下：
```shell
st2ctl reload --register-actions
```

## 创建workflow

这里我们使用Orchestra写workflow [Orchestra](https://github.com/StackStorm/orchestra).


在这个工作流中，我们要调用 `tutorial.nasa_apod` 来获取图片URL，然后发布到RabbitMQ的队列中

**注意** workflow文件的名字**必须**与action元数据的文件的名字一样

> 查看并编辑`/opt/stackstorm/packs/tutorial/actions/workflows/nasa_apod_rabbitmq_publish.yaml`

``` yaml
version: '1.0'

description: nasa image to rabbitmq demo.

input:
  - date
  - message
  - host
  - exchange
  - exchange_type
  - routing_key
  - username
  - password
  - port
  - exchange_durable
  
vars:
  - apod_url: null
  - flag: null

tasks:
    get_apod_url:
      action: tutorial.nasa_apod date=<% ctx().date %>
      next:
      - when: <% succeeded() %>
        publish: apod_url=<% task('get_apod_url').result.result.url %>, flag=<% ctx(message) %>
        do: publish_to_rabbitmq
    
    publish_to_rabbitmq:
      action: rabbitmq.publish_message
      input:
        host: <% ctx(host) %>
        exchange: <% ctx(exchange) %>
        exchange_type: <% ctx(exchange_type) %>
        routing_key: <% ctx(routing_key) %>
        message: "<% ctx(apod_url) %>,<% ctx(flag) %>"
        username: <% ctx(username) %>
        password: <% ctx(password) %>
        port: <% ctx(port) %>
        exchange_durable: <% ctx(exchange_durable) %>

output:
  - url: <% ctx(apod_url) %>

```

## 测试

测试发布一个消息：

``` shell
st2 run tutorial.nasa_apod_rabbitmq_publish date="2018-07-04"
```

查看队列：

```shell
rabbitmqadmin get queue=demoqueue count=99
```

> 这里有几个辅助工具：
> - [orquestaconvert](https://github.com/StackStorm/orquestaconvert)：可以将Mistral workflows 转换为 Orquesta workflows
> - [rehearsal](https://github.com/trstruth/rehearsal) 可以根据输入的orquesta文件画一个可视化的流程图，在当前版本没有图形编排工具的情况下可以辅助开发