# Create first chart

`helm ls`

# Command to create a chart

`helm create <chart-name>`

ex: `helm create firstchart`

```
helm create first-chart
Creating first-chart
```

A folder gets created name `first-chart`

```
ls -lrth
total 16K
-rw-r--r-- 1 infracloud infracloud 1.9K Dec  9 23:52 values.yaml
drwxr-xr-x 3 infracloud infracloud 4.0K Dec  9 23:52 templates
-rw-r--r-- 1 infracloud infracloud 1.2K Dec  9 23:52 Chart.yaml
drwxr-xr-x 2 infracloud infracloud 4.0K Dec  9 23:52 charts
```

- `chart.yaml` contains the metadata about the chart exists
- `charts` folder is where charts of dependent charts are stored
- `templates` here all the manifests are stored , like deployment yaml, ingress yaml , service yaml etc 
- `values.yaml` this file contains all the default values that need to go as values in template yaml file

Lets install the first-chart that we created

`helm install firstapp first-chart/`

```
helm install first-app first-chart/
NAME: first-app
LAST DEPLOYED: Fri Dec 10 00:02:31 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=first-chart,app.kubernetes.io/instance=first-app" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:$CONTAINER_PORT
```

