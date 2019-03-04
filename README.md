# filebeat-docker docker版本轻量型日志采集器
## 目录结构
```\
├── docker-compose.yml filebeat 容器编排文件
├── filebeat
│   ├── Dockerfile  filebeat 构建文件
│   └── filebeat.yml filebeat 配置文件
├── LICENSE 
├──README.md 项目说明文件
└──env-example 配置文件
```
## 快速使用
1. 本地安装`git`、`docker`和`docker-compose`。
2. `clone`项目：
```
$git clone https://github.com/thinksvip/filebeat-docker.git
```   
3. 如果不是`root`用户，还需将当前用户加入`docker`用户组：
```
    $ sudo gpasswd -a ${USER} docker
```
4. 启动：
```
    $ cd filebeat-docker
    $ cp env-example .env
    $ docker-compose up
```
## 配置日志路径

1. 本项目支持直接修改env-example挂载 `nginx` `mysql` `laravel` `system` `php` 日志。如需添加其他类型日志,请将路径添加加在`docker-compose.yml`。
2. 需注意`./filebeat/filebeat.yml`中`paths`路径对应为`docker-compose.yml`中filebeat容器挂载路径。

## filebeat配置

```yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/nginx.log # 文件名和目录名都可以用通配符，不过 path/*/*.log 不包括 path 根目录的文件
    fields: # 添加额外的字段
      log_topic: "nginx-test"
      service: "backend-nginx"
    fields_under_root: true # field 字段会放在根索引下，否则会放在 fields 字段下
    ignore_older: 24h # 忽略 24 小时前的文件
    scan_frequency: 11s # 设置不同的时间，这样可以错开扫描高峰
    max_backoff: 11s
    backoff: 11s
    harvester_buffer_size: 51200 # 采集的 buffer 大小
    # close_timeout: 1h # 因为我这里的文件是一个小时产生一个，所以直接设置采集器默认 1 小时关闭
    clean_inactive: 25h # 需要大于 ignore_older + scan_frequency，有效地清理可以减小 registry 文件的大小和当中记录的文件条目数量
    harvester_limit: 10 # 限制最多爬取个数，默认不限制，如果碰到文件数很多一开始会占用大量 cpu
## Kafka Output 相关属性设置: https://www.elastic.co/guide/en/beats/filebeat/current/kafka-output.html
output.kafka:
  enabled: true
  hosts: ["x.x.x.x:9092"]
  topic: '%{[log_topic]}' ##匹配fileds字段下的logtopic
  codec.format:
    string: '%{[message]}' #'%{[beat][hostname]} %{[service]} %{[message]}' # 传给 logstash 时可以用 grok 过滤插件设置 `match => { "message" => "^%{DATA:hostname} %{DATA:service} (?<message>.*)"}` 解析
    # 如果设成 ‘%{[message]}’，则是原样转发消息
  partition.round_robin:
    reachable_only: false
  required_acks: 1
  compression: none # 默认是 gzip，如果想节省开销可以设成 none
  bulk_max_size: 100
  max_message_bytes: 1000000 # 不要超过 kafka server 端设置的 message.max.size，否则超过的部分会被丢弃

```

## License
MIT
