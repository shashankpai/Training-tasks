#  Exporters 

Exporter is a sw or number of libraries and servers that help in exporting existing metrics from third-party systems like linux  and windows in same format as
of prometheus


you’ll learn how to run Prometheus and Node Exporter as Docker containers on a Linux machine, with the containers managed by Docker Compose. You’ll mount the relevant host directories into the Node Exporter and Prometheus containers, and configure Prometheus to scrape Node Exporter metrics

```
version: "3.9"
services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - 9100:9100
    networks:
      - prom
  prometheus:
    image: prom/prometheus:v2.22.0
    container_name: prometheus
    ports:
    - 9090:9090
    command:
    - --config.file=/etc/prometheus/prometheus.yaml
    volumes:
    - ./prometheus.yaml:/etc/prometheus/prometheus.yaml:ro
    networks:
     - prom
networks:
  prom:
    name: prom
```

```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        #- job_name: 'csvserver'
        #static_configs:
        #- targets: ['csvserver:9300']
  - job_name: "node"
    static_configs:
    - targets: ["node-exporter:9100"]
```

now `docker compose up -d` will start the stack

check `docker ps` if containers are up for prometheus and node exporter


play around with different metrics available 

