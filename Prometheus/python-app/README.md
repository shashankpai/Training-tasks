# Client libraries - Adding instrumentation to Python Application

Before you can monitor your services, you need to add instrumentation to their code via one of the Prometheus client libraries. These implement the Prometheus metric types.

Choose a Prometheus client library that matches the language in which your application is written. This lets you define and expose internal metrics via an HTTP endpoint on your application’s instance:

Markup : * Go
         * Java or Scala
         * Python
         * Ruby


# Simple python http server

We will create http server whic will be available on 8000 port and expose the metrics on 8001 port 

We will achive it using docker and docker-compose 


python app code 

```
import http.server
from prometheus_client import start_http_server

APP_PORT = 8000
METRICS_PORT = 8001

class HandleRequests(http.server.BaseHTTPRequestHandler):

    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()
        self.wfile.write(bytes("<html><head><title>First Application</title></head><body style='color: #333; margin-top: 30px;'><center><h2>Welcome to our first Prometheus-Python application.</center></h2></body></html>", "utf-8"))
        self.wfile.close()

if __name__ == "__main__":
    start_http_server(METRICS_PORT)
    server = http.server.HTTPServer(('0.0.0.0', APP_PORT), HandleRequests)
    server.serve_forever()
```

docker file

```
FROM python:3.8

ENV SRC_DIR /usr/app/src

RUN pip install prometheus_client

COPY app1.py ${SRC_DIR}/

WORKDIR ${SRC_DIR}/

ENV PYTHONUNBUFFERED=1

EXPOSE 8000

RUN chmod u+x app1.py

CMD [ "python", "app1.py"]
```

docker-compose file - part 1 

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
  http-server:
    container_name: http-server
    image: http-server
    ports:
    - 8000:8000
    - 8001:8001
    build:
      context: .
    networks:
      - prom
networks:
  prom:
    name: prom
```


run by doing a docker-compose up -d


This is the basic way through which we will expose the app metrics to prometheus. we will build on going forward the app code and dockerfile for different metrics type


# 1. Exposing counter type metrics

Now we will expose the counter type metrics , here we will calculate the total number of http requests made to the client


```
from prometheus_client import start_http_server, Counter

REQUEST_COUNT = Counter('app_requests_count', 'total app http request count')

```

in the above code we imported `Counter` from `prometheus_client` and created a metric named `app_requests_count` with description for that metric type

some theory on `metric names` : 
Markup : * metric names should start with letter and can be followed with number or letters , numbers and underscores
         * they should have unique names or client libraries would report an error
         * if applicable, when exposing time series for counter type metric, a `_total` suffix is added automatically

Full app code for counter metric
```

import http.server
from prometheus_client import start_http_server, Counter

REQUEST_COUNT = Counter('app_requests_count', 'total app http request count')

APP_PORT = 8000
METRICS_PORT = 8001

class HandleRequests(http.server.BaseHTTPRequestHandler):

    def do_GET(self):
        REQUEST_COUNT.inc()
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()
        self.wfile.write(bytes("<html><head><title>First Application</title></head><body style='color: #333; margin-top: 30px;'><center><h2>Welcome to our first Prometheus-Python application.</center></h2></body></html>", "utf-8"))
        self.wfile.close()

if __name__ == "__main__":
    start_http_server(METRICS_PORT)
    server = http.server.HTTPServer(('0.0.0.0', APP_PORT), HandleRequests)
    server.serve_forever()
```

now a new metric has been added if we check the expression browser , as we hit the url more , the count will increase


`app_requests_count_total`

`app_requests_count_total{instance="http-server:8001",job="http-server"}	12`

also try rate function which is specifically for counters

`rate(app_requests_count_total[10m])`

You can also increment the counter by more than 1 , Prom allows you to increment by any non negative number.

Lets add a new conter with random count


full code for random count metrics

```
import http.server 
import random                     ## import random
from prometheus_client import start_http_server, Counter

REQUEST_COUNT = Counter('app_requests_count', 'total app http request count')
RANDOM_COUNT = Counter('app_random_count', 'increment counter by random value')     ## new metric named RANDOM_COUNT 


APP_PORT = 8000
METRICS_PORT = 8001

class HandleRequests(http.server.BaseHTTPRequestHandler):

    def do_GET(self):
        REQUEST_COUNT.inc()
        random_val = random.random()*10       ## random_val var 
        RANDOM_COUNT.inc(random_val)          ## increment by random value
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()
        self.wfile.write(bytes("<html><head><title>First Application</title></head><body style='color: #333; margin-top: 30px;'><center><h2>Welcome to our first Prometheus-Python application.</center></h2></body></html>", "utf-8"))
        self.wfile.close()

if __name__ == "__main__":
    start_http_server(METRICS_PORT)
    server = http.server.HTTPServer(('0.0.0.0', APP_PORT), HandleRequests)
    server.serve_forever()
```

save cahnges and then do a `docker-compose up --build -d`


check the newly created metrics in expression browser and check the increase in count with random values 

# Adding labels to exposed metrics

It will be better if we get metrics based on endpoints, pages , So we will build the code by adding labels to appname and endpoints 

```

import http.server
import random
from prometheus_client import start_http_server, Counter

REQUEST_COUNT = Counter('app_requests_count', 'total app http request count',['app_name','endpoint'])   ## new labels with app_name and endpoint
RANDOM_COUNT = Counter('app_random_count', 'increment counter by random value')


APP_PORT = 8000
METRICS_PORT = 8001

class HandleRequests(http.server.BaseHTTPRequestHandler):

    def do_GET(self):
        REQUEST_COUNT.labels('prom_python_app', self.path).inc()        ## mapping app_name as prom_python_app and endpoint to path
        random_val = random.random()*10
        RANDOM_COUNT.inc(random_val)
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()
        self.wfile.write(bytes("<html><head><title>First Application</title></head><body style='color: #333; margin-top: 30px;'><center><h2>Welcome to our first Prometheus-Python application.</center></h2></body></html>", "utf-8"))
        self.wfile.close()

if __name__ == "__main__":
    start_http_server(METRICS_PORT)
    server = http.server.HTTPServer(('0.0.0.0', APP_PORT), HandleRequests)
    server.serve_forever()
```



run the build again ``

Check the new metrics in expression browser by calling different paths 

`app_requests_count_total`

```
app_requests_count_total{app_name="prom_python_app",endpoint="/",instance="http-server:8001",job="http-server"}	12
app_requests_count_total{app_name="prom_python_app",endpoint="/abc",instance="http-server:8001",job="http-server"}	1
app_requests_count_total{app_name="prom_python_app",endpoint="/favicon.ico",instance="http-server:8001",job="http-server"}	12
app_requests_count_total{app_name="prom_python_app",endpoint="/newz",instance="http-server:8001",job="http-server"}	1
app_requests_count_total{app_name="prom_python_app",endpoint="/red",instance="http-server:8001",job="http-server"}	1
app_requests_count_total{app_name="prom_python_app",endpoint="/xyz",instance="http-server:8001",job="http-server"}
```

`sum(app_requests_count_total) by(app_name)`