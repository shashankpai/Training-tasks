# Monitoring applications and Kubernetes with Prometheus

Exercises here are performed from the book `Kubernetes in a month of lunches` link https://www.manning.com/books/learn-kubernetes-in-a-month-of-lunches 

# How Prometheus monitors Kubernetes workloads

Metrics in Prometheus are completely generic: each component you want to monitor has an HTTP endpoint, which returns all the values that are important to that component. A web server includes metrics for the number of requests it serves, and a Kubernetes node includes metrics for how much memory is available. Prometheus doesn’t care what’s in the metrics; it just stores everything the component returns. What’s important to Prometheus is a list of targets it needs to collect from

# We will get prometheus running first 

we will apply the manifests in prometheus folder 

```
kubectl apply -f prometheus/
namespace/kiamol-ch14-monitoring created
configmap/prometheus-config created
service/prometheus created
deployment.apps/prometheus created
serviceaccount/prometheus created
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRole is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRole
clusterrole.rbac.authorization.k8s.io/prometheus created
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRoleBinding
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
```
check if the pods are starting up

`kubectl wait --for=condition=ContainersReady pod -l app=prometheus -n kiamol-ch14-monitoring`

```
kubectl get svc prometheus -n kiamol-ch14-monitoring
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
prometheus   LoadBalancer   10.96.174.86   172.18.255.200   9090:31480/TCP   5m11s
```

Prometheus calls metrics collection scraping. When you browse to the Prometheus UI, you’ll see there are no scrape targets, although there is a category called test-pods, which lists zero targets.

The test-pods name comes from the Prometheus configuration you deployed in a ConfigMap, which the Pod reads from.

some part of the config 

```
global:
      scrape_interval: 30s
     
    scrape_configs:
      - job_name: 'test-pods'
        kubernetes_sd_configs:
        - role: pod 
```


Prometheus uses jobs to define a related set of targets to scrape, which could be multiple components of an application. The scrape configuration can be as simple as a static list of domain names, which Prometheus polls to grab the metrics, or it can use dynamic service discovery. Listing below shows the beginning of the test-pods job configuration, which uses the Kubernetes API for service discovery.

```
scrape_configs:                # This is the YAML inside the ConfigMap.
 - job_name: 'test-pods'      # Used for test apps
   kubernetes_sd_configs:     # Finds targets from the Kubernetes API
   - role: pod                # Searches for Pods
   relabel_configs:           # Applies these filtering rules
   - source_labels:          
       - __meta_kubernetes_namespace
     action: keep             # Includes Pods only where the namespace
     regex: kiamol-ch14-test  # is the test namespace for this chapter
```

It’s the relabel_configs section that needs explanation. Prometheus stores metrics with labels, which are key-value pairs that identify the source system and other relevant information. You’ll use labels in queries to select or aggregate metrics, and you can also use them to filter or modify metrics before they are stored in Prometheus

egular expressions rear their unnecessarily complicated heads in Prometheus, too, but it’s rare that you need to make changes. The pipeline you set up in the relabeling phase should be generic enough to work for all your apps. The full pipeline in the configuration file applies the following rules:

- Include Pods only from the namespace kiamol-ch14-test.
- Use the Pod name as the value of the Prometheus instance label.
- Use the app label in the Pod metadata as the value of the Prometheus job label.
- Use optional annotations in the Pod metadata to configure the scrape target.


This approach is convention-driven—as long as your apps are modeled to suit the rules, they’ll automatically be picked up as monitoring targets. Prometheus uses the rules to find Pods that match, and for each target, it collects metrics by making an HTTP GET request to the /metrics path. Prometheus needs to know which network port to use, so the Pod spec needs to explicitly include the container port. That’s a good practice anyway because it helps to document your application’s setup. Let’s deploy a simple app to the test namespace and see what Prometheus does with it.


Deploy the timecheck application to the test namespace. The spec matches all the Prometheus scrape rules, so the new Pod should be found and added as a scrape target.

```
# create the test namespace and the timecheck Deployment:
kubectl apply -f timecheck/ 

# wait for the app to start:
kubectl wait --for=condition=ContainersReady pod -l app=timecheck -n kiamol-ch14-test
```


Prometheus saw the timecheck Pod being created, and it matched all the rules in the relabel stage, so it was added as a target. The Prometheus configuration is set to scrape targets every 30 seconds. The timecheck app has a /metrics endpoint, which returns a count for how many timecheck logs it has written. When I queried that metric in Prometheus, the app had written 22 log entries


You should realize two important things here: the application itself needs to provide the metrics because Prometheus is just a collector, and those metrics represent the activity for one instance of the application. The timecheck app isn’t a web application—it’s just a background process—so there’s no Service directing traffic to it. Prometheus gets the Pod IP address when it queries the Kubernetes API, and it makes the HTTP request directly to the Pod. You can configure Prometheus to query Services, too, but then you’d get a target that is a load balancer across multiple Pods, and you want Prometheus to scrape each Pod independently.


You’ll use the metrics in Prometheus to power dashboards showing the overall health of your apps, and you may aggregate across all the Pods to get the headline values. You need to be able to drill down, too, to see if there are differences between the Pods. That will help you identify if some instances are performing badly, and that will feed back into your health checks. We can scale up the timecheck app to see the importance of collecting at the individual Pod level.

Add another replica to the timecheck app. It’s a new Pod that matches the Prometheus rules, so it will be discovered and added as another scrape target.


```
# scale the Deployment to add another Pod:
kubectl scale deploy/timecheck --replicas 2 -n kiamol-ch14-test

# wait for the new Pod to spin up:
kubectl wait --for=condition=ContainersReady pod -l app=timecheck -n kiamol-ch14-test

# back in Prometheus, check the target list, and in the graph page,
# execute queries for timecheck_total and dotnet_total_memory_bytes
```

Prometheus finds the new Pod and starts scraping it. Both Pods record the same metrics, and the Pod name is set as a label on each metric. The query for the timecheck_total metric now returns two results—one for each Pod



# Monitoring apps built with Prometheus client libraries

The components of that app are in Java, Go, and Node.js, and they each use a Prometheus client library to expose run-time and application metrics.

Deploy the APOD app to the test namespace, and confirm that the three components of the app are added as Prometheus targets.

```
# deploy the app:
kubectl apply -f apod/

# wait for the main component to start:
kubectl wait --for=condition=ContainersReady pod -l app=apod-api -n kiamol-ch14-test

# get the app URL:
kubectl get svc apod-web -n kiamol-ch14-test
```

`Important:` First, the Pod specs all include a container port, which states that the application container is listening on port 80, and that’s how Prometheus finds the target to scrape. The Service for the web UI actually listens on port 8014, but Prometheus goes directly to the Pod port. Second, the API target isn’t using the standard /metrics path, because the Java client library uses a different path. I’ve used an annotation in the Pod spec to state the correct path.

Convention-based discovery is great because it removes a lot of repetitive configuration and the potential for mistakes, but not every app will fit with the conventions. The relabeling pipeline we’re using in Prometheus gives us a nice balance. The default values will work for any apps that meet the convention, but any that don’t can override the defaults with annotations.

below shows how the override is configured to set the path to the metrics.

```
- source_labels:   # This is a relabel configuration in the test-pods job.

 - __meta_kubernetes_pod_annotationpresent_prometheus_io_path
 - __meta_kubernetes_pod_annotation_prometheus_io_path

regex: true;(.*)   # If the Pod has an annotation named prometheus.io/path . . .

target_label:  __metrics_path__  # sets the target path from the annotation.
```

The rule says: if the Pod has an annotation called `prometheus.io/path`, then use the value of that annotation as the metrics path. Prometheus does it all with labels, so every Pod annotation becomes a label with the name `meta_kubernetes_pod_annotation_<annotation-name>`, and there’s an accompanying label called `meta_kubernetes_pod_annotationpresent_<annotation-name>`, which you can use to check if the annotation exists. Any apps that use a custom metrics path need to add the annotation. 


example annotation in api spec 

```
template:                 # This is the pod spec in the Deployment.
 metadata:
   labels:
     app: apod-api       # Used as the job label in Prometheus
   annotations:
     prometheus.io/path: "/actuator/prometheus"   # Sets the metrics path
```

# Introducing grafana

The UI is great for running queries and quickly seeing the results, but you can’t use it to build a dashboard showing all the key metrics for your app in a single screen. For that you can use Grafana, another open source project in the container ecosystem, which comes recommended by the Prometheus team

Deploy Grafana with ConfigMaps that set up the connection to Prometheus, and include a dashboard for the APOD app.


```
# deploy Grafana in the monitoring namespace:
kubectl apply -f grafana/

# wait for it to start up:
kubectl wait --for=condition=ContainersReady pod -l app=grafana -n kiamol-ch14-monitoring

# get the URL for the dashboard:
kubectl get svc grafana -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:3000/d/kb5nhJAZk' -n kiamol-ch14-monitoring

# browse to the URL; log in with username kiamol and password kiamol
```




