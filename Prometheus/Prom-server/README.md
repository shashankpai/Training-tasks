# Simple way to get started with prometheus basics 


we are going to spawn a container using docker-compose and a sample prometheus.yaml. going forward we will build on it 

docker-compose.yaml

```
version: "3.9"
services:
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

prometheus.yaml

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

```
run `docker-compose up -d` to get started

# Some pointers on prometheus

# Basic terminologies in Prometheus
```
1 . Monitoring - systematic process of collecting and recording activities taking place in target project 
2. Alerting - an alert is outcome of an alerting rule in prometheus that is actively firing. Alert are sent  from prom to alertmanager
3. Alertmanager - it takes alert from prom server , aggreagates them into groups , deduplicates, silences , throttles and then sends out notifications 
   to email, pagerduty, slack etc
4. target - a target is definition of an object to scrape , target is an object whose metrics are to be monitored 
5. instance - in promehteus terms, an endpoint you can scrape is called an instance
6. job - job is a collection of targets/instances with same purpose

7. sample - a sample is a single value at a point in time in a time series 
   ex : http_requests_total{method="get"}     32
```


# 3 Prometheus Architecture

https://images.prismic.io/gspann/478b51d2-5b2d-4ca2-9957-8848b782123a_Image+1_+Prometheus+Architecture.jpg?auto=compress,format

![Alt text](https://images.prismic.io/gspann/478b51d2-5b2d-4ca2-9957-8848b782123a_Image+1_+Prometheus+Architecture.jpg?auto=compress,format "Prometheus Architecture")
