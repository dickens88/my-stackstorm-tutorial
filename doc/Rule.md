# Rules

StackStorm用Rule和Workflow来将操作转变为自动化，Rules将trigger映射到action(或workflow)，应用匹配到的criteria并且映射trigger payload给action输入

## 配置Rule

本例中我们匹配`rabbitmq.new_message` trigger返回的信息，并且调用action将``demoqueue``队列中输入的消息转移到另外一个MQ队列

> 查看并编辑 `/opt/stackstorm/packs/tutorial/rules/post_rabbitmq_to_othermq.yaml`

```yaml
---
name: "post_rabbitmq_to_othermq"
description: "Post RabbitMQ message to another queue."
enabled: true

trigger:
  type: "rabbitmq.new_message"
  parameters: {}

action:
  ref: "tutorial.post_rabbitmq_to_multimq"
  parameters:
    queue: "{{ trigger.queue }}"
    body: "{{ trigger.body }}"
```

为此我们需要再创建一组action-workflow ``tutorial.post_rabbitmq_to_multimq``，将消息发送到其他队列，具体过程会在下个章节详述

> 查看并编辑
>
>  `/opt/stackstorm/packs/tutorial/action/post_rabbitmq_to_multimq.yaml`
>
> `/opt/stackstorm/packs/tutorial/action/workflow/post_rabbitmq_to_multimq.yaml`

注册rules：

```
st2ctl reload --register-rules
```

## 测试Rule

发布一个消息，查看消息是否被发布到``pyohio``队列

```
st2 run tutorial.nasa_apod_rabbitmq_publish date="2018-07-04" message="sensor to #pyohio"
```

## 创建Action和Workflow

这里Action就是一个Workflow，收到一条来自rabbitmq的消息，workflow会检验消息体并且决定要将消息转发到哪个队列，如果消息中包含`#pyohio`则将消息发送到队列`pyohio`，如果消息体包含`stackstorm`则发送消息到队列`stackstorm`

> 查看并编辑`/opt/stackstorm/packs/tutorial/action/post_rabbitmq_to_multimq.yaml`

```yaml
---
name: post_rabbitmq_to_multimq
pack: tutorial
description: "Post a RabbitMQ message to Slack"
runner_type: "orquesta"
enabled: true
entry_point: workflows/post_rabbitmq_to_multimq.yaml
parameters:
  queue:
    type: string
    description: "Queue the message was received on"
  body:
    type: string
    description: "Body of the message"
```

> 查看并编辑`/opt/stackstorm/packs/tutorial/action/workflow/post_rabbitmq_to_multimq.yaml`
```yaml
---
version: '1.0'
input:
  - queue
  - body
tasks:
  channel_branch:
    action: core.noop
    next:
      - when: "{{ '#pyohio' in ctx().body }}"
        publish:
          - chat_message: "Received a message on RabbitMQ queue {{ ctx().queue }}\n {{ ctx().body }}"
        do:
          - post_to_pyohio
      - when: "{{ '#stackstorm' in ctx().body }}"
        publish:
          - chat_message: "Received a message on RabbitMQ queue {{ ctx().queue }}\n {{ ctx().body }}"
        do:
          - post_to_stackstorm
  post_to_pyohio:
    action: rabbitmq.publish_message
    input:
      host: "10.0.2.15"
      port: 5672
      username: "admin"
      password: "admin"
      exchange: "demo"
      exchange_type: "topic"
      routing_key: "pyohio_key"
      message: '{{ ctx().chat_message }}'
  post_to_stackstorm:
    action: rabbitmq.publish_message
    input:
      host: "10.0.2.15"
      port: 5672
      username: "admin"
      password: "admin"
      exchange: "demo"
      exchange_type: "topic"
      routing_key: "stackstorm_key"
      message: "{{ ctx().chat_message }}"

```

注册相关组件：

```
st2ctl reload --register-all
```

## 测试工作流

分别发送消息到两个队列
```
st2 run tutorial.nasa_apod_rabbitmq_publish date="2020-12-05" message="sensor to #pyohio"
```

```
root@185f6453e40a:/opt/stackstorm/packs/tutorial/rules# st2 run tutorial.nasa_apod_rabbitmq_publish date="2020-12-05" message="sensor to #pyohio"
..
id: 5ff566f09f87412a6014b025
action.ref: tutorial.nasa_apod_rabbitmq_publish
parameters: 
  date: '2020-12-05'
  message: 'sensor to #pyohio'
status: succeeded
start_timestamp: Wed, 06 Jan 2021 07:29:52 UTC
end_timestamp: Wed, 06 Jan 2021 07:29:56 UTC
result: 
  output:
    url: https://apod.nasa.gov/apod/image/2012/MonsRumker_Letellier.jpg
+--------------------------+------------------------+-------------------+-------------------+-----------------+
| id                       | status                 | task              | action            | start_timestamp |
+--------------------------+------------------------+-------------------+-------------------+-----------------+
| 5ff566f1745ff832fa674645 | succeeded (2s elapsed) | get_apod_url      | tutorial.nasa_apo | Wed, 06 Jan     |
|                          |                        |                   | d                 | 2021 07:29:53   |
|                          |                        |                   |                   | UTC             |
| 5ff566f3745ff832fa674655 | succeeded (1s elapsed) | publish_to_rabbit | rabbitmq.publish_ | Wed, 06 Jan     |
|                          |                        | mq                | message           | 2021 07:29:55   |
|                          |                        |                   |                   | UTC             |
+--------------------------+------------------------+-------------------+-------------------+-----------------+
```

