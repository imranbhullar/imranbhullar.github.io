---
layout: post
title: "Modern Linux Monitoring: Prometheus, node_exporter, and Grafana"
date: Nov-29-2012
categories: [Linux, DevOps]
tags: [linux, monitoring, prometheus, grafana, observability]
image: /assets/img/prometheus-grafana-server-room.jpg
---

Nagios and its plugin ecosystem can tell you a host is down. They're much clumsier at answering "how has memory pressure trended over the last three weeks" or "graph this alongside that." Prometheus and Grafana, together, are the stack most teams reach for when they outgrow check-based monitoring.

## Pull vs Push: Why Prometheus Is Different

Older agent-based tools push metrics to a central collector; Prometheus does the opposite, it scrapes targets on a schedule.

```text
Push model:    host -> agent pushes metrics -> central collector
Pull model:    Prometheus -> scrapes /metrics on each target on an interval
```

*Tip:* The pull model means Prometheus itself knows immediately if a target stopped responding, that's a monitoring signal in its own right (`up == 0`), without any extra agent logic.

## Installing node_exporter

node_exporter is the agent that exposes OS-level metrics in the plain-text format Prometheus scrapes.

```bash
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz
tar xzf node_exporter-1.8.0.linux-amd64.tar.gz
sudo mv node_exporter-1.8.0.linux-amd64/node_exporter /usr/local/bin/
```

*Tip:* node_exporter ships as a single static binary with no runtime dependencies. There's no package manager wrangling required, just drop it in `/usr/local/bin` and run it.

## Running node_exporter as a systemd Service

A long-running exporter belongs under systemd, not a stray background process that dies on reboot.

```ini
# /etc/systemd/system/node_exporter.service
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin node_exporter
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
curl -s localhost:9100/metrics | head
```

*Tip:* Run it as a dedicated unprivileged user. node_exporter reads a lot from `/proc` and `/sys` but has no legitimate need to write anywhere or run as root.

## What Metrics It Exposes

The `/metrics` endpoint is just text, and it's worth actually reading once to know what's available.

```bash
curl -s localhost:9100/metrics | grep -E "^node_(cpu|memory|filesystem|network)" | head -20

# A few of the most commonly graphed:
# node_cpu_seconds_total
# node_memory_MemAvailable_bytes
# node_filesystem_avail_bytes
# node_network_receive_bytes_total
```

*Tip:* These are counters and gauges, not percentages. A counter like `node_cpu_seconds_total` only makes sense graphed as a rate over time, which is exactly what PromQL's `rate()` is for.

## A Minimal prometheus.yml

Prometheus itself needs to know what to scrape and how often.

```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "node"
    static_configs:
      - targets: ["localhost:9100", "db01:9100", "web01:9100"]
```

*Tip:* `static_configs` is fine for a handful of hosts. Past a dozen or so, look at file-based service discovery or a cloud provider's SD integration instead of hand-maintaining a target list.

## Running Prometheus

Same pattern as node_exporter: a static binary, run under systemd, pointed at its config.

```bash
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz
tar xzf prometheus-2.53.0.linux-amd64.tar.gz
cd prometheus-2.53.0.linux-amd64
./prometheus --config.file=/etc/prometheus/prometheus.yml
```

*Tip:* Prometheus's web UI on port 9090 includes a "Targets" page under Status. Check it first whenever a graph looks empty, a target can be scraping successfully, failing, or simply never configured, and that page tells you which.

## Basic PromQL Queries

PromQL is how you turn raw time series into the number or rate you actually care about.

```promql
# CPU usage rate over the last 5 minutes, per core
rate(node_cpu_seconds_total{mode="idle"}[5m])

# Available disk space on the root filesystem, in GB
node_filesystem_avail_bytes{mountpoint="/"} / 1024 / 1024 / 1024

# Is a target currently reachable?
up{job="node"}
```

*Tip:* `rate()` requires a counter and a range (`[5m]`), never apply it to a gauge like memory usage, which is already an instantaneous value and needs no rate calculation.

## Connecting Grafana as a Data Source

Grafana doesn't store metrics itself; it queries Prometheus and renders the result.

```bash
sudo apt install -y grafana
sudo systemctl enable --now grafana-server
# Then in the UI: Connections -> Data sources -> Add data source -> Prometheus
# URL: http://localhost:9090
```

*Tip:* Point Grafana at Prometheus's address, not at node_exporter directly. Grafana never talks to exporters; it only ever talks to the Prometheus server that already scraped and stored their data.

## Importing a Community Dashboard

Writing dashboards from scratch is rarely necessary for something as standard as node_exporter metrics.

```text
Grafana UI -> Dashboards -> Import -> paste dashboard ID 1860
("Node Exporter Full", one of the most widely used community dashboards)
```

*Tip:* Importing a known-good community dashboard first, then trimming panels you don't need, is almost always faster than building a monitoring dashboard panel by panel.

## A Simple Alerting Rule

Alerting belongs in Prometheus itself, evaluated against the same data the graphs are built from.

```yaml
# /etc/prometheus/rules/disk.yml
groups:
  - name: disk
    rules:
      - alert: LowDiskSpace
        expr: node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} < 0.10
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Root filesystem below 10% free on {{ $labels.instance }}"
```

*Tip:* The `for: 10m` clause matters as much as the threshold. Without it, a single noisy scrape crossing the line fires an alert; with it, the condition has to hold steady before anyone gets paged.
