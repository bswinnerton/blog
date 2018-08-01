---
layout: post
title: Monitoring your SNMP devices with Grafana in Docker
author: Brooks Swinnerton
---

## Monitoring Other Devices with SNMP

### InfluxDB

```
sudo docker volume create influxdb-config
sudo docker volume create influxdb-data
```

```
sudo su -
docker run --rm influxdb influxd config > /var/lib/docker/volumes/influxdb-config/_data/influxdb.conf
exit
```

```
sudo docker run -d \
    --name="influxdb" \
    --restart=always \
    -p 8086:8086 \
    -v influxdb-data:/var/lib/influxdb \
    -v influxdb-config:/etc/influxdb \
    influxdb -config /etc/influxdb/influxdb.conf
```

```
sudo docker exec -it influxdb influx
USE _internal
CREATE DATABASE telegraf
```

### Telegraf

```
sudo docker volume create telegraf-config
```

```
sudo docker run -d \
    --name telegraf \
    --restart=always \
    -v telegraf-config:/etc/telegraf \
    -v /var/run/docker.sock:/var/run/docker.sock \
    bswinnerton/telegraf-snmp-unifi
```
