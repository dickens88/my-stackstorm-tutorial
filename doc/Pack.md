# Packs

Packs是package的缩写，可以理解为组织发布StackStorm插件的一个集合，pack里面可以包含和发布以下内容：

- actions
- workflows
- rules
- sensors
- triggers
- Python `requirements.txt` 
- Additional content (think Jinja templates, Ansible playbooks, etc)

Packs实际就是`git`仓库，所以你可以通过`git`仓库的url直接安装packs，当然也可以到[StackStorm exchange](https://exchange.stackstorm.org/)下载安装，StackStorm Exchange类似一个工具市场

在本例中，我们将会安装一个 `rabbitmq`的pack，并且通过StackStorm post消息到队列中

## 环境准备
为了演示这个场景，我们需要先准备一个rabbitmq环境，最简单的做法是使用docker搭建。下述语句会从下载并创建一个带管理console的rabbitmq通过http://locahost:15672 访问控制台
```
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 --hostname myRabbit  -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin rabbitmq:management
```

创建队列，这里可以通过RabbitMQ容器内的命令行创建，也可以通过RabbitMQ的console创建

```
rabbitmqadmin declare exchange name=demo type=topic durable=false
rabbitmqadmin declare queue name=demoqueue
rabbitmqadmin declare binding source=demo destination=demoqueue routing_key=demokey
rabbitmqadmin declare queue name=pyohio
rabbitmqadmin declare binding source=demo destination=pyohio routing_key=pyohio_key
rabbitmqadmin declare queue name=stackstorm
rabbitmqadmin declare binding source=demo destination=stackstorm routing_key=stackstorm_key
```

## 安装RabbitMQ Packs

从 [StackStorm exchange](https://exchange.stackstorm.org/) 安装RabbitMQ packs

```
st2 pack install rabbitmq
```

测试从RabbitMQ Packs发消息

```
st2 run rabbitmq.publish_message host=127.0.0.1 exchange=demo exchange_type=topic routing_key=demokey message="test"
```

从MQ查看消息

```
rabbitmqadmin get queue=demoqueue count=99
```

