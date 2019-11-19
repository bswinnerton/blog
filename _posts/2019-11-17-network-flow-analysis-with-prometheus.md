--
layout: post
title: Network Flow Analysis With Prometheus
author: Brooks Swinnerton
---

Network analytics tools are a valuable way to analyze the traffic patterns of an autonomous system. They can be used to gain valuable insights into ingress and egress traffic, identify potential peering relationships, and help in troubleshooting a network.

In investigating potential network analysis tools for [Neptune Networks](https://neptunenetworks.org), I struggled to find something that was 1) up to date, and 2) wouldn’t break the bank. The first tool that I came across at the recommendation of a colleague was [AS Stats](https://github.com/manuelkasper/AS-Stats), a series of perl scripts that generate RRD-based graphs. I opted not to go down this path as it’s unmaintained and I wasn’t confident I’d be able to get it working given it hasn’t been supported for a few years. The second tool I explored was [Kentik](https://www.kentik.com/), which is a beautiful cloud hosted solution but after chatting with their sales team, I learned that it costs tens of thousands of dollars for its most basic package 😱. I was disappointed to hear they didn’t have a more cost effective solution for autonomous systems with smaller amounts of traffic. The last tool I tried which proved quite promising was [Elastiflow](https://github.com/robcowart/elastiflow). Elastiflow is open source and is built on top of ELK (Elasticsearch, LogStash, and Kibana). While I was able to get this working successfully, I found that the amount of hardware required to operate this was far too much and would end up breaking the [hardware] bank.

In this article I explore creating a network analytics system with [Prometheus](https://prometheus.io/) and [Grafana](https://grafana.com/). Prometheus is a time series database complete with a query language (PromQL), and Grafana is a way to visualize that data. The end result is a dashboard that looks like this:

![](https://user-images.githubusercontent.com/934497/67167662-85e57080-f36a-11e9-96e2-f6f5b3b7e5d0.png)

To accomplish this, the following tools are used:
- [pmacct](https://github.com/pmacct/pmacct) - A daemon to collect the network flow data
- [Kafka](https://kafka.apache.org/) - A distributed, immutable log that can handle all of the flow data to be analyzed
- [Flow Exporter](https://github.com/neptune-networks/flow-exporter) - A Kafka consumer to analyze the flow data
- [Prometheus](https://prometheus.io/) - A tool to store the collected metrics
- [Grafana](https://grafana.com/) - A web interface to visualize our Prometheus metrics in graphs

Before digging in, it’s worth briefly introducing the autonomous system I was building this system for. Neptune Networks has four border routers with full routing tables and all external traffic passes through them. Each of these routers are Linux-based virtual machines running the [BIRD routing daemon](https://bird.network.cz/). Most conventional hardware routers have the ability to export what are called “flows” (Netflow, sFlow, IPFIX) to a collector. Because Linux has no built-in support for this, the first step was capturing the data that was flowing through these routers.

### pmacct

[pmacct](https://github.com/pmacct/pmacct) is a set of network monitoring tools that can collect network traffic via libpcap and export it to a variety of places. To power our network analysis tool, we’ll be collecting this data on each of our border routers and sending it to Kafka for later analysis. The pmacctd daemon also peers with our border routers over BGP in order to get an up-to-date routing table to associate prefixes with autonomous system numbers.

Let’s start by installing the pmacct suite on our border routers:

```shell
sudo apt install build-essential libpcap-dev libtool pkg-config m4 automake librdkafka-dev libjansson-dev
cd /tmp
wget https://github.com/pmacct/pmacct/archive/v1.7.3.tar.gz
tar -xzf v1.7.3.tar.gz
cd /tmp/pmacct-1.7.3
./autogen.sh
./configure --enable-kafka --enable-jansson
make
sudo make install
```

And then let's set up a systemd service for pmacctd:

**`/etc/systemd/system/pmacctd.service`**

```ini
[Unit]
Description=pmacctd Netflow probe
After=network.target

[Service]
ExecStart=/usr/local/sbin/pmacctd -f /etc/pmacct/pmacctd.conf

[Install]
WantedBy=multi-user.target
```

Now that we have pmacct installed, we can start working on the configuration file that will monitor the network interfaces and send the data to Kafka.

**`/etc/pmacctd/pmacctd.conf`**
```yaml
daemonize: false
pcap_interfaces_map: /etc/pmacct/interfaces.map
pre_tag_map: /etc/pmacct/pretag.map
pmacctd_as: longest
pmacctd_net: longest
networks_file: /etc/pmacct/networks.lst
networks_file_no_lpm: true
sampling_rate: 1
aggregate: src_host, dst_host, src_port, dst_port, src_as, dst_as, label, proto
!
bgp_daemon: true
bgp_daemon_ip: 127.0.0.2
bgp_daemon_port: 180
bgp_daemon_max_peers: 10
bgp_agent_map: /etc/pmacct/peering_agent.map
!
plugins: kafka
kafka_output: json
kafka_broker_host: kafka-host.fqdn.com
kafka_refresh_time: 5
kafka_history: 5m
kafka_history_roundoff: m
```

Let’s break down some of these configuration options.

#### pmacctd base configuration

1. [**`daemonize`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L41-L43): I opt not to daemonize the process since it will be managed by systemd.
2. [**`pcap_interfaces_map`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L431-L437): Showcased below, this is a way to get a single pmacctd process to capture traffic from multiple interfaces.
    > **`/etc/pmacct/interfaces.map`**
    >
    > ```
    > ifindex=100  ifname=ens3
    > ifindex=200  ifname=ens4
    > ```

3. [**`pre_tag_map`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L1838-L1848): Also showcased below, this is a hook that allows us to tag captured traffic (flows). We will use this as a way to set the hostname so we know which router the data is coming from in Prometheus.
    > **`/etc/pmacct/pretag.map`**
    > ```
    > set_label=bdr1.neptunenetworks.org
    > ```

4. [**`pmacctd_as`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L1750-L1770), [**`pmacctd_net`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L1772-L1801), [**`networks_file`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L859-L868), [**`networks_file_no_lpm`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L878-L901): These options are used together to associate a particular prefix with an ASN. The rationale for this is outlined in [this mailing list](https://www.mail-archive.com/pmacct-discussion@pmacct.net/msg03864.html), but TL;DR it helps differentiate between AS0 and your own ASN.
    > **`/etc/pmacct/networks.lst`**
    > ```
    > 397143,23.157.160.0/24
    > 397143,2602:fe2e::/36
    > ```

5. [**`sampling_rate`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L1925-L1933): Enables packet sampling. In high traffic routers, you may not want to capture _all_ flows. The value is meant to be thought of as “one out of `x`” flows are captured, a value of `1` would mean that all flows are sampled.
6. [**`aggregate`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L45-L133): This is the data that you wish to capture about each flow. In the configuration above, we capture the address, port, and ASN of both the source and destination, the label we set in the `pre_tag_map`, and the protocol.

#### pmacctd BGP configuration

The next group of configuration options create a BGP session between pmacctd and our border router. The border router passes its full routing table to pmacctd so that it can determine the autonomous systems of each flow it captures.

7. [**`bgp_daemon_ip`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L2217-L2222): The address of pmacctd’s BGP instance. Because in my case [BIRD](https://bird.network.cz/) is already bound to `127.0.0.1`, we bind to a new address on a different port in the next configuration option. Be sure to set this address on the loopback interface (`sudo ip addr add 127.0.0.2/32 dev lo`).
8. [**`bgp_daemon_port`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L2238-L2247): Again, because we already have a BGP instance running on `127.0.0.1:179`, we have pmacctd bind to a different port: `180`. Be sure to poke a hole in your firewall for this if needed.
9. [**`bgp_agent_map`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L2444-L2454): This maps the BIRD router ID with source addresses in flows.
    > **`/etc/pmacct/peering_agent.map`**
    > ```
    > bgp_ip=23.157.160.1     ip=0.0.0.0/0
    > ```

Pausing for a moment, you may be wondering what the other side of this BGP configuration looks like. In BIRD, we configure the other end of the session like so:

```javascript
protocol bgp pmacctd {
  description "pmacctd";
  local 127.0.0.1 as 397143;
  neighbor 127.0.0.2 port 180 as 397143;
  rr client;
  hold time 90;
  keepalive time 30;
  graceful restart;

  ipv4 {
    next hop self;
    import filter { reject; };
    export filter { accept; };
  };

  ipv6 {
    next hop address 127.0.0.1;
    import filter { reject; };
    export filter { accept; };
  };
}
```

#### pmacctd Kafka configuration

And lastly, we have a series of configuration options to send the flow data to Kafka.

1. [**`kafka_broker_host`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L720-L730): The Kafka broker to send the flow data to (which we configure below).
3. [**`kafka_refresh_time`**](https://github.com/pmacct/pmacct/blob/661f9679b7de2a754482c823e5988e0b26ad941c/CONFIG-KEYS#L1165-L1175): The time interval in which to actually send the flows. pmacctd buffers all flows and flushes them to Kafka according to the value set here.

### Analysis with Kafka, Flow Exporter & Prometheus

Now that pmacctd is configured on our border routers, it can capture flows and send them to the Kafka host that we defined above.

Kafka is a distributed log that is designed to handle large amounts of incoming data. Data is sent to the Kafka "broker" and "consumers" read this data from the broker and do whatever they please with it.

We'll be using [Flow Exporter](https://github.com/neptune-networks/flow-exporter) as our Kafka consumer. It keeps a tally of all traffic categorized by ASN and figures out the name of the autonomous system from the RIR. It then takes this data and makes it available to Prometheus which scrapes it at a regular interval.

Let's stand up each of these services.

You have a few options for creating a Kafka broker. You can use an off-the-shelf Kafka host like [Heroku Kafka](https://www.heroku.com/kafka) or stand your own up in a Docker container. You should not need to concern yourself much with disk space, as the retention for Kafka will be quite short since the data will be stored in Prometheus. Kafka is just an intermediary.

Should you decide to use Docker, you can use the following Docker Compose file for Kafka, Flow Exporter, and Prometheus.

**`docker-compose.yml`**:

```yaml
version: '3'

services:
  zookeeper:
    image: wurstmeister/zookeeper
    container_name: zookeeper
    restart: always
    ports:
      - "2181:2181"
    networks:
      - kafka-internal

  kafka:
    image: wurstmeister/kafka
    container_name: kafka
    restart: always
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    networks:
      - kafka
      - kafka-internal
    environment:
      KAFKA_ADVERTISED_LISTENERS: INSIDE://:19092,OUTSIDE://kafka.fqdn.com:9092
      KAFKA_LISTENERS: INSIDE://:19092,OUTSIDE://:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INSIDE
      KAFKA_CREATE_TOPICS: "pmacct.acct:3:1"
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LOG_RETENTION_HOURS: 1
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  flow-exporter:
    image: docker.pkg.github.com/neptune-networks/flow-exporter/flow-exporter:latest
    container_name: flow-exporter
    restart: always
    command: --brokers=kafka:19092 --topic=pmacct.acct --asn=397143
    expose:
      - 9590
    networks:
      - monitor
      - kafka

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    restart: always
    command: --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/prometheus
    expose:
      - 9090
    networks:
      - monitor
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - data:/prometheus

networks:
  kafka-internal:
  kafka:
  monitor:

volumes:
  data:
```

Be sure to swap out some of the arguments to the `command`s. Specifically you'll need to specify the FQDN that you intend to have your Kafka broker on (`OUTSIDE://kafka.fqdn.com:9092`), and the ASN you are monitoring flows from (`--asn=397143`).

Also don't forget to open ports in your firewall so that your border routers can communicate with Kafka.

And lastly, you'll need to configure Prometheus to scrape the data coming out of Flow Exporter:

**`prometheus.yml`**:

```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s
scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'flow-exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['flow-exporter:9590']
```

Now that all of the configuration options are in place, we can start pmacctd:

- `sudo systemctl enable pmacctd`
- `sudo systemctl start pmacctd`

If you tail your pmacctd log, you should hopefully see some lines like this:

```shell
$ sudo journalctl -u pmacctd -f
```

```
Nov 17 23:00:44 bdr1 pmacctd[25162]: INFO ( default/core/BGP ): [127.0.0.1] BGP peers usage: 1/2
Nov 17 23:00:44 bdr1 pmacctd[25162]: INFO ( default/core/BGP ): [23.157.160.1] Capability: MultiProtocol [1] AFI [1] SAFI [1]
Nov 17 23:00:44 bdr1 pmacctd[25162]: INFO ( default/core/BGP ): [23.157.160.1] Capability: MultiProtocol [1] AFI [2] SAFI [1]
Nov 17 23:00:44 bdr1 pmacctd[25162]: INFO ( default/core/BGP ): [23.157.160.1] Capability: 4-bytes AS [41] ASN [397143]
Nov 17 23:00:44 bdr1 pmacctd[25162]: INFO ( default/core/BGP ): [23.157.160.1] BGP_OPEN: Local AS: 397143 Remote AS: 397143 HoldTime: 90
Nov 17 23:00:46 bdr1 pmacctd[25162]: INFO ( default_kafka/kafka ): *** Purging cache - START (PID: 25172) ***
Nov 17 23:00:48 bdr1 pmacctd[25162]: INFO ( default_kafka/kafka ): *** Purging cache - END (PID: 25172, QN: 640/640, ET: 0) ***
```

In the first 6 lines we can see pmacctd establishing a BGP connection with our border router, and in the last two lines we can see pmacctd flush the flow cache to Kafka, and then two seconds later complete it.

We can validate that this worked properly with the `kafkacat` tool. Similar to netcat, kafkacat will let you consume a given topic of a Kaffka broker:

```shell
$ kafkacat -b kafka-broker.neptunenetworks.org:9092 -C -t pmacct.acct -o -1000
```

```json
{"event_type": "purge", "label": "bdr2.neptunenetworks.org", "as_src": 60729, "as_dst": 397143, "ip_src": "185.220.102.6", "ip_dst": "23.157.160.138", "port_src": 443, "port_dst": 46255, "ip_proto": "tcp", "stamp_inserted": "2019-11-17 23:00:00", "stamp_updated": "2019-11-17 23:03:51", "packets": 248, "bytes": 1075281, "writer_id": "default_kafka/20567"}
{"event_type": "purge", "label": "bdr2.neptunenetworks.org", "as_src": 49981, "as_dst": 397143, "ip_src": "185.165.240.126", "ip_dst": "23.157.160.138", "port_src": 42012, "port_dst": 9050, "ip_proto": "tcp", "stamp_inserted": "2019-11-17 23:00:00", "stamp_updated": "2019-11-17 23:03:51", "packets": 42, "bytes": 20182, "writer_id": "default_kafka/20567"}
{"event_type": "purge", "label": "bdr1.neptunenetworks.org", "as_src": 397143, "as_dst": 15169, "ip_src": "2602:fe2e:42:8:9d35:dd53:2c65:fdfd", "ip_dst": "2607:f8b0:400d:c0c::bc", "port_src": 61983, "port_dst": 5228, "ip_proto": "tcp", "stamp_inserted": "2019-11-17 23:00:00", "stamp_updated": "2019-11-17 23:03:51", "packets": 1, "bytes": 60, "writer_id": "default_kafka/25365"}
```

Keep an eye out for the appropriate AS values in `as_src` and `as_dst`. A value of zero may take some tweaking to the `bgp_agent_map` part of your pmacct configuration.

### Grafana

Now the fun part, loading the data in Grafana.

If you already have a Grafana instance set up I've put together a shared dashboard [here](https://grafana.com/grafana/dashboards/11206). You should be able to add it by hovering over the `+` on the sidebar and choosing "import" and pasting in `11206` as the "Grafana.com Dashboard".

### Voila!

And that's it! You'll be able to filter the dashboard by AS and host, which can be valuable to determine potential peering relationships should you be present on an internet exchange or in the same datacenter for private peering.

I hope this is helpful. Feel free to open an issue in [https://github.com/neptune-networks/flow-exporter/issues](https://github.com/neptune-networks/flow-exporter/issues).
