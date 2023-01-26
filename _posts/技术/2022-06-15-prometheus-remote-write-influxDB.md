---
layout: post
title: prometheus remote write to influx-DB
category: 技术
---

# remote write to influx-db

# prepare influx-db

influx-db v2 should use telegraph

influx-db v1.8 原生支持prometheus remote write协议

```jsx
sudo docker run -p 8086:8086 \
      --name influxdb \
      -v influxdb:/opt/pulsar/influxdb \
      influxdb:1.8
```

## influx-db 创建测试数据库

```jsx
wsg@ansible-control:~$ sudo docker exec -it influxdb bash
root@f46e6bcea141:/# ls
bin   dev         etc   init-influxdb.sh  lib64  mnt  proc  run   srv  tmp  var
boot  entrypoint.sh  home  lib             media  opt  root  sbin  sys  usr
root@f46e6bcea141:/# influx
Connected to http://localhost:8086 version 1.8.10
InfluxDB shell version: 1.8.10
> show databases;
name: databases
name
----
_internal
> show users;
user admin
---- -----
> create database prom;
> show databases;
name: databases
name
----
_internal
prom
>
```

# config

## prometheus config

修改prometheus.yml

```jsx
---
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s # By default, scrape targets every 15 seconds.
  # scrape_timeout is set to the global default (10s).
  external_labels:
    cluster: 'cluster-rack'
    monitor: "prometheus"
    mylabel: "hello-world"
    app_id: 001461

# Load and evaluate rules in these files every 'evaluation_interval' seconds.
# rule_files:

remote_write:
- url: "http://10.2.0.10:8086/api/v1/prom/write?db=prom"
  write_relabel_configs:
  - source_labels: [job]
    regex: '(.*)'
    replacement: 'cluster1-${1}'
    target_label: unit_id
    action: replace
- url: "http://10.2.0.10:8087/api/v1/prom/write?db=prometheus"
```

可以remote write到多个influx db中

### 补充： 华泰需要新增一个 ip的label，调整的配置如下

```jsx
remote_write:
- url: "http://10.2.0.10:8086/api/v1/prom/write?db=prom"
  write_relabel_configs:
  - source_labels: [job]
    regex: '(.*)'
    replacement: 'cluster1-${1}'
    target_label: unit_id
    action: replace
  - source_labels: [instance]
    #regex: '(.*)'
    regex: '(.*):(.*)'
    replacement: '${1}'
    target_label: ip
    action: replace
- url: "http://10.2.0.10:8087/api/v1/prom/write?db=prometheus"
```

instance的取值格式如下（以：号间隔）

![Untitled](../../images/remote%20write%20to%20influx-db%2061aba584a5ad412bb8d26ff03514853b/Untitled.png)

## grafana config

### add datasource

![Untitled](../../images/remote%20write%20to%20influx-db%2061aba584a5ad412bb8d26ff03514853b/Untitled%201.png)

![Untitled](../../images/remote%20write%20to%20influx-db%2061aba584a5ad412bb8d26ff03514853b/Untitled%202.png)

![Untitled](../../images/remote%20write%20to%20influx-db%2061aba584a5ad412bb8d26ff03514853b/Untitled%203.png)

### 查询influx db中的数据

![Untitled](../../images/remote%20write%20to%20influx-db%2061aba584a5ad412bb8d26ff03514853b/Untitled%204.png)

### 检查指标的 lable信息

![Untitled](../../images/remote%20write%20to%20influx-db%2061aba584a5ad412bb8d26ff03514853b/Untitled%205.png)

#
