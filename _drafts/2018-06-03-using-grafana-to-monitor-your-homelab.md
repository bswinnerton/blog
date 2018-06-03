---
layout: post
title: Monitoring your home lab with Grafana in Docker
author: Brooks Swinnerton
---

I have a rather complex mix of physical hosts, KVM guests, and Docker containers in my home lab that I'd like to monitor the health of.

I use Grafana for this, and after taking the steps in this blog post, the final result will look something like this:

![Grafana]({{ site.baseurl }}/public/images/grafana-docker.png)

## Setting up Grafana

Grafana is an open source metrics dashboard. I've chosen to run Grafana as a Docker container to reduce the memory footprint and avoid the maintenance overhead of having another host just for this application. Grafana itself is a standalone application that requires little to no configuration, other than how to fetch the metrics data. We'll be storing the metrics in proper data stores later in this blog post.

We're going to start by creating two Docker volumes. One for Grafana's configuration file, and another for its persistent data (for example the dashboards). Volumes are handy because they counteract the ephemeral nature of containers; when a container is destroyed but brought back up with the same volume, everything will feel the same.

```
sudo docker volume create grafana-config
sudo docker volume create grafana-data
```

This will result in two new folders on your Docker host: `/var/lib/docker/volumes/grafana-config/_data/` and `/var/lib/docker/volumes/grafana-data/_data/`. They'll start out empty, but once we get Grafana running, you'll notice data will start to appear in them.

After that, we can run the following command to add the container:

```
sudo docker run -d \
  --name="grafana" \
  --restart=always \
  --expose 3000 \
  -v grafana-data:/var/lib/grafana \
  -v grafana-config:/etc/grafana \
  -e "GF_SECURITY_ADMIN_PASSWORD=hKg9szzXaQ7NkiRbT97n" \
  grafana/grafana
```

This command is relatively intuitive, but to break it down piece by piece:

- `sudo docker run`: Docker needs to run as root, so we preface the command with `sudo`.
- `-d`: For Grafana, we'll want this container to run in the background as a "daemon".
- `--restart=always`: This flag denotes that we want the system to always reboot the container in the event of a failure, as well as automatically when Docker starts.
- `--expose 3000`: This will expose the internal port 3000 on the host's port 3000. This is the port that Grafana runs on, and how we'll access the web interface.
- `-v grafana-data:/var/lib/grafana`: This mounts the `grafana-data` volume that we created earlier in container's filesystem at `/var/lib/grafana`, which is defined in the [`Dockerfile`](https://github.com/grafana/grafana-docker/blob/89b7c50c1e69ba0c9902ef90b33e98b3d73bbf47/Dockerfile#L9) as where it will look for the Grafana data.
- `-v grafana-config:/etc/grafana`: This mounts the `grafana-config` volume that we created earlier in the container's filesystem at `/etc/grafana`, which is defined in the [`Dockerfile`](https://github.com/grafana/grafana-docker/blob/89b7c50c1e69ba0c9902ef90b33e98b3d73bbf47/Dockerfile#L8) as where it will look for the Grafana configuration.
- `-e "GF_SECURITY_ADMIN_PASSWORD=hKg9szzXaQ7NkiRbT97n"`: This sets an environment variable that will be used by the container to set the default admin user's password. **You should change this value**.

Now Grafana should be running on port `3000` of your Docker host.

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
