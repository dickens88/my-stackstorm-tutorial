# Sensors

Sensor用来与外部系统和事件集成，Sensor是一些python的代码片段，可以周期性的从外部拉取信息，或者被动等待来自外部的事件。Sensor可以产生trigger，trigger可以匹配rules进而执行其他action

Sensor由python编写，而且必须遵循StackStorm定义的sensor接口要求

## 配置Sensor

本例我们将复用`RabbitMQ` pack里面的sensor `rabbitmq.queues_sensor`，这个sensor需要读取`/opt/stackstorm/configs/rabbitmq.yaml`配置文件，这个配置里包含了`queues`参数，定义sensor要监听哪个队列。作为初始化步骤，我们需要

复制 `/opt/stackstorm/packs/rabbitmq/rabbitmq.yaml.example` 到 `/opt/stackstorm/configs/rabbitmq.yaml`

并且修改配置文件的内容：

```yaml
---
sensor_config:
  host: "10.0.2.15"
  username: "admin"
  password: "admin"
  rabbitmq_queue_sensor:
    queues:
      - "demoqueue"
    deserialization_method: "json"
```

接下来要注册config

```
st2ctl reload --register-configs
```

## 测试Sensor

发布一个新消息到MQ

```
st2 run tutorial.nasa_apod_rabbitmq_publish date="2018-07-04" message="hey sensor"
```

查看trigger是否被创建

```
$ st2 trigger-instance list --trigger rabbitmq.new_message
+--------------------------+----------------+-----------------+-----------+
| id                       | trigger        | occurrence_time | status    |
+--------------------------+----------------+-----------------+-----------+
| 5b5dce8e587be00afa97911f | rabbitmq.new_m | Sun, 29 Jul     | processed |
|                          | essage         | 2018 14:26:22   |           |
|                          |                | UTC             |           |
| 5b5dce8e587be00afa979120 | rabbitmq.new_m | Sun, 29 Jul     | processed |
|                          | essage         | 2018 14:26:22   |           |
|                          |                | UTC             |           |
| 5b5dce8e587be00afa97912b | rabbitmq.new_m | Sun, 29 Jul     | processed |
|                          | essage         | 2018 14:26:22   |           |
|                          |                | UTC             |           |
+--------------------------+----------------+-----------------+-----------+
```

查看消息内容，可以看到MQ队列中的信息已经被Sensor读取出来，并且创建了trigger，接下来将对获取的信息做进一步处理

```
$ st2 trigger-instance get 5b5dce8e587be00afa97912b
+-----------------+-----------------------------+
| Property        | Value                       |
+-----------------+-----------------------------+
| id              | 5b5dce8e587be00afa97912b    |
| trigger         | rabbitmq.new_message        |
| occurrence_time | 2018-07-29T14:26:22.482000Z |
| payload         | {                           |
|                 |     "queue": "demoqueue",   |
|                 |     "body": "test sensor"   |
|                 | }                           |
| status          | processed                   |
+-----------------+-----------------------------+
```

> 在docker-compose部署的场景下，可以查看通过``docker logs -f st2-docker_st2sensorcontainer_1``查看sensor执行的日志，本地部署的场景下，日志文件保存在``/var/log/st2/*``