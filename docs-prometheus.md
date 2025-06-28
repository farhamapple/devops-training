# Prometheus

## Install Prometheus
Download prometheus package
```
wget https://github.com/prometheus/prometheus/releases/download/v2.53.3/prometheus-2.53.3.linux-amd64.tar.gz
```

Extract prometheus package
```
tar -zxf prometheus-2.53.3.linux-amd64.tar.gz
mv -v prometheus-2.53.3.linux-amd64 /etc/prometheus
```

Running and configuration prometheus
```
cd /etc/prometheus
vim prometheus.yml
./prometheus --config.file=prometheus.yml
```

Access prometheus ui
```
http://localhost:9090/
```

Query prometheus
```
prometheus_target_interval_length_seconds
```

## Setup Exporter
### Node Exporter
Download node exporter package
```
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
```

Extract node exporter package
```
tar -zxf node_exporter-1.8.2.linux-amd64.tar.gz
mv -v node_exporter-1.8.2.linux-amd64 /etc/node_exporter
```

Start Service
```
cd /etc/node_exporter
./node_exporter --web.listen-address 172.23.x.x:9100
```

Access node metric
```
http://ip-address:port/metrics
```

Config prometheus
```
vim prometheus.yml
```

Add Targets
```
- job_name: "Server monitoring"
    scrape_interval: 5s
    static_configs:
      - targets: ["172.23.x.x:9100", "172.23.x.x:9100", "172.23.x.x:9100"]
        labels:
          group: 'devops'
          os: 'linux'
```

Start service
```
./prometheus --config.file=prometheus.yml
```

Access prometheus ui
```
http://172.23.x.x:9090/
```

Query metric
```
rate(node_cpu_seconds_total{mode="system"}[1m])
```
### Docker Cadvisor Exporter
Running cAdvisor in a Docker Container
```
sudo docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  --privileged \
  --device=/dev/kmsg \
  gcr.io/cadvisor/cadvisor:v0.49.1
```
cAdvisor is now running (in the background) on
```
http://172.23.x.x:8080/
```

Config prometheus
```
vim prometheus.yml
```

Add targets
```
- job_name: "docker monitoring"
    scrape_interval: 5s
    static_configs:
      - targets: ["172.23.x.x:8080"]
        labels:
          group: 'docker'
```

Start Service
```
./prometheus --config.file=prometheus.yml
```

Access prometheus ui
```
http://172.23.x.x:9090/
```

Query metric
```
rate(container_cpu_usage_seconds_total{name="redis"}[1m])
```
### Nginx Exporter
Edit nginx config
```
vim default.conf
```

Add metrics 
```
server {
    listen       80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }    
    // add this
    location /metrics {
        stub_status;
    }
}
```

Deploy nginx exporter
```
docker container run -d --rm -p 9113:9113 --name nginx-exporter nginx/nginx-prometheus-exporter -nginx.scrape-uri http://172.23.x.x/metrics
```

Config prometheus
```
vim prometheus.yml
```

Add targets
```
- job_name: "nginx monitoring"
    scrape_interval: 5s
    static_configs:
      - targets: ["172.23.x.x:9113"]
        labels:
          group: 'nginx'
```

Start Service
```
./prometheus --config.file=prometheus.yml
```

Access prometheus ui
```
http://172.23.x.x:9090/
```

Query metric
```
nginx_connections_active
```
## Setup rule
### Setup Record Rule
```
vim prometheus.rules.yml
```

```
groups:
  - name: cpu-node
    rules:
    - record: node_cpu
      expr: avg by (job, instance, mode) (rate(node_cpu_seconds_total[5m]))
```

```
vim prometheus.yml
```

```
rule_files:
  - prometheus.rules.yml
```

```
./prometheus --config.file=prometheus.yml
```

## Setup Alert Rule
### Alert Server is down
```
vim server.rules.yml
```

```
groups:
- name: server_down
  rules:
  - alert: server_down
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: Server are down.
```

```
vim prometheus.yml
```

```
rule_files:
  - prometheus.rules.yml
  - server.rules.yml
```

```
./prometheus --config.file=prometheus.yml
```

### Alert CPU and memory > 50%
```
vim cpu.rules.yml
```

```
groups:
  - name: example_alerts
    rules:
    - alert: HighCPUUtilization
      expr: avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[1m]) * 100) < 50
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: High CPU utilization on host {{ $labels.instance  }}
        description: The CPU utilization on host {{ $labels.instance }} has exceeded 50 for 1 minutes.
```

```
vim prometheus.yml
```

```
rule_files:
  - prometheus.rules.yml
  - server.rules.yml
  - cpu.rules.yml
```

```
./prometheus --config.file=prometheus.yml
```
