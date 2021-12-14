


`helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`

```
helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
```


```
helm search repo kube-prometheus-stack 
NAME                                            CHART VERSION   APP VERSION     DESCRIPTION                                       
prometheus-community/kube-prometheus-stack      23.2.0          0.52.1          kube-prometheus-stack collects Kubernetes manif...
```


```
---
defaultRules:
  rules:
    etcd: true
    kubeScheduler: true
kubeControllerManager:
  enabled: true
kubeEtcd:
  enabled: true
kubeScheduler:
  enabled: true
prometheus:
  prometheusSpec:
    serviceMonitorSelector:
      matchLabels:
        prometheus: devops
commonLabels:
  prometheus: devops
grafana:
  adminPassword: test123
```

lets install with `helm`

`helm install monitoring prometheus-community/kube-prometheus-stack --values prometheus-values.yaml --version 16.10.0 --namespace monitoring --create-namespace`

now the helm stack is ready 

```
kubectl get all -n monitoring
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running   0          80s
pod/monitoring-grafana-8669bf97c9-22s2b                      2/2     Running   0          2m1s
pod/monitoring-kube-prometheus-operator-bb596cf69-nbvh2      1/1     Running   0          2m1s
pod/monitoring-kube-state-metrics-69b5565f46-lmv24           1/1     Running   0          2m1s
pod/monitoring-prometheus-node-exporter-2qh4k                1/1     Running   0          2m1s
pod/monitoring-prometheus-node-exporter-6msgt                1/1     Running   0          2m1s
pod/monitoring-prometheus-node-exporter-ll98c                1/1     Running   0          2m1s
pod/prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running   1          80s

NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                     ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   80s
service/monitoring-grafana                        ClusterIP   10.96.205.96    <none>        80/TCP                       2m1s
service/monitoring-kube-prometheus-alertmanager   ClusterIP   10.96.181.169   <none>        9093/TCP                     2m1s
service/monitoring-kube-prometheus-operator       ClusterIP   10.96.50.107    <none>        443/TCP                      2m1s
service/monitoring-kube-prometheus-prometheus     ClusterIP   10.96.115.172   <none>        9090/TCP                     2m1s
service/monitoring-kube-state-metrics             ClusterIP   10.96.148.74    <none>        8080/TCP                     2m1s
service/monitoring-prometheus-node-exporter       ClusterIP   10.96.243.58    <none>        9100/TCP                     2m1s
service/prometheus-operated                       ClusterIP   None            <none>        9090/TCP                     80s

NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/monitoring-prometheus-node-exporter   3         3         3       3            3           <none>          2m1s

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/monitoring-grafana                    1/1     1            1           2m1s
deployment.apps/monitoring-kube-prometheus-operator   1/1     1            1           2m1s
deployment.apps/monitoring-kube-state-metrics         1/1     1            1           2m1s

NAME                                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/monitoring-grafana-8669bf97c9                   1         1         1       2m1s
replicaset.apps/monitoring-kube-prometheus-operator-bb596cf69   1         1         1       2m1s
replicaset.apps/monitoring-kube-state-metrics-69b5565f46        1         1         1       2m1s

NAME                                                                    READY   AGE
statefulset.apps/alertmanager-monitoring-kube-prometheus-alertmanager   1/1     80s
statefulset.apps/prometheus-monitoring-kube-prometheus-prometheus       1/1     80s
```

we will port forward the prom service on 9090 port 

`kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090 -n monitoring`


# 2 Now lets create a service monitor for a app that exposes prometheus metrics 

lets deploy on of the deployments exposing it through service

```
kubectl apply -f timecheck-monitoring.yaml
```

```
kubectl apply -f timecheck-service.yaml
```

```
kubectl get all -n mon1
NAME                             READY   STATUS    RESTARTS   AGE
pod/timecheck-559654746d-bk29p   1/1     Running   0          10m

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/timecheck   ClusterIP   10.96.135.188   <none>        8080/TCP   10m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/timecheck   1/1     1            1           10m

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/timecheck-559654746d   1         1         1       10m
```

we will now create a `service-monitor-deploy.yaml` for the app that we just deployed


To check `serviceMonitorSelector`

`kubectl get prometheus monitoring-kube-prometheus-prometheus -n monitoring -o yaml`

```
serviceMonitorSelector:
    matchLabels:
      release: monitoring
```

make sure you are aware of the serviceMonitorSelector and then proceed creating the `service-monitor-deploy.yaml`

 then update the values in it by describing the ep in namespace mon1 

 ```
 kubectl describe  ep -n mon1
Name:         timecheck
Namespace:    mon1
Labels:       kiamol=ch14
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2021-12-14T18:49:32Z
Subsets:
  Addresses:          10.244.1.33
  NotReadyAddresses:  <none>
  Ports:
    Name          Port  Protocol
    ----          ----  --------
    http-metrics  8080  TCP

Events:  <none>
```




```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: timecheck
  namespace: mon1
  labels:
    release: monitoring
spec:
  endpoints:
  - port: http-metrics
    interval: 60s
    scrapeTimeout: 30s
  namespaceSelector:
    matchNames:
    - mon1
  selector:
    matchLabels:
      #app.kubernetes.io/instance: postgres
      kiamol: ch14
```

kubectl apply -f `service-monitor-deploy.yaml`






## Some useful docs and links for understanding and troubleshooting

https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/troubleshooting.md#troubleshooting-servicemonitor-changes
https://github.com/antonputra/tutorials/tree/main/lessons/072
https://rtfm.co.ua/en/kubernetes-a-clusters-monitoring-with-the-prometheus-operator/


