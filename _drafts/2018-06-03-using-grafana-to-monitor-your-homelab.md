---
layout: post
title: Monitoring your homelab with Grafana in Docker
author: Brooks Swinnerton
---

I have a rather complex mix of physical hosts, KVM guests, and Docker containers that I'd like to monitor the health of.

I use Grafana for this, and after taking the steps in this blog post, the final result looks something like this:

![Grafana]({{ site.baseurl }}/public/images/grafana-docker.png)

## Node Exporter

```
wget node_exporter
tar xzf
sudo cp node_exporter /usr/local/bin
```

`/etc/systemd/system/node_exporter.service`

```
[Unit]
Description=Node Exporter

[Service]
User=root
ExecStart=/usr/local/bin/node_exporter $OPTIONS

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl enable node_exporter.service
sudo systemctl start node_exporter.service
```

## Docker Setup

### cadvisor

```
sudo docker run \
    --restart=always \
    --volume=/:/rootfs:ro \
    --volume=/var/run:/var/run:rw \
    --volume=/sys:/sys:ro \
    --volume=/var/lib/docker/:/var/lib/docker:ro \
    --volume=/dev/disk/:/dev/disk:ro \
    --publish=8080:8080 \
    --detach=true \
    --name=cadvisor \
    google/cadvisor:latest
```

### Grafana

```
sudo docker volume create grafana-config
sudo docker volume create grafana-data
```

```
sudo docker run -d \
    --name="grafana" \
    --restart=always \
    --expose 3000 \
    -v grafana-data:/var/lib/grafana \
    -v grafana-config:/etc/grafana \
    -e VIRTUAL_HOST=graphs.brooks.network \
    -e VIRTUAL_PORT=3000 \
    -e LETSENCRYPT_HOST=graphs.brooks.network \
    -e LETSENCRYPT_EMAIL=bswinnerton@gmail.com \
    -e "GF_SECURITY_ADMIN_PASSWORD=ujETk4BHUdcFp4MTrkr3B95n" \
    grafana/grafana
```

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

### Prometheus

```
sudo docker volume create prometheus-config
sudo docker volume create prometheus-data
```

TODO: Add link to where you found this info:

```
sudo chown -R nobody /var/lib/docker/volumes/prometheus-config
sudo chown -R nobody /var/lib/docker/volumes/prometheus-data
```

```
sudo docker run -d \
    --name=prometheus \
    --restart=always \
    -p 9090:9090 \
    -v prometheus-config:/etc/prometheus \
    -v prometheus-data:/prometheus \
    prom/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus
```

`/var/lib/docker/volumes/prometheus-config/_data/prometheus.yml`

```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['containers.brooks.network:8080']
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['containers.brooks.network:9100', 'nuc7i5.brooks.network:9100', 'unifi.brooks.network:9100', 'homebridge.brooks.network:9100']
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

- [ ] Add links to dashboards
- [ ] Add screenshots
