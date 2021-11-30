# 1. Running containers in pods and deployments

Kubernetes runs containers for your application workloads, but the containers themselves are not objects you need to work with. Every container belongs to a Pod, which is a Kubernetes object for managing one or more containers, and Pods, in turn, are managed by other resources

# 2. Basic building blocks of Kubernetes: Pods which run containers and deployments which manage pods

# 3. How kubernetes runs and manages containers

A container is a virtualized environment that typically runs a single application component. Kubernetes wraps the container in another virtualized environment: the Pod. A Pod is a unit of compute, which runs on a single node in the cluster. The Pod has its own virtual IP address, which is managed by Kubernetes, and Pods in the cluster can communicate with other Pods over that virtual network, even if they’re running on different nodes.

You normally run a single container in a Pod, but you can run multiple containers in one Pod, which opens up some interesting deployment options.

All the containers in a Pod are part of the same virtual environment, so they share the same network address and can communicate using localhost

# 4. Run a simple pod using kuberntes command line without yaml


`kubectl run hello-kiamol --image=kiamol/ch02-hello-kiamol --restart=Never`

wait for pod to be ready 
`kubectl wait --for=condition=Ready pod hello-kiamol`

list all the Pods in the cluster:
`kubectl get pods`

 show detailed information about the Pod
 `kubectl describe pod hello-kiamol`

# 5. how are containers managed in kubernetes

Kubernetes doesn’t really run containers, though—it passes the responsibility for that to the container runtime installed on the node, which could be Docker or containerd  That’s why the Pod is an abstraction: it’s the resource that Kubernetes manages, whereas the container is managed by something outside of Kubernetes.

get the basic information about the Pod:
`kubectl get pod hello-kiamol`

specify custom columns in the output, selecting network details:
`kubectl get pod hello-kiamol --output custom-columns=NAME:metadata.name,NODE_IP:status.hostIP,POD_IP:status.podIP `

specify a JSONPath query in the output,
selecting the ID of the first container in the Pod:
`kubectl get pod hello-kiamol -o jsonpath='{.status.containerStatuses[0].containerID}'`

# 6. Key takeaways uptil now 
   
   # 6.1 Kubernetes does not run containers—the container ID in the Pod is a reference to another system that runs containers. 
   # 6.2 Pods are allocated to one node when they’re created, and it’s that node’s responsibility to manage the Pod and its containers. It does that by working with the container runtime using a known API called the Container Runtime Interface (CRI).


# 7 This exercise shows you how a Kubernetes node keeps its Pod containers running, but you’ll only be able to follow it if you’re using Docker as your container runtime. In our case we are using kind and it uses containerd as CRI

find the Pod’s container:
`docker container ls -q --filter label=io.kubernetes.container.name=hello-kiamol`

now delete that container:
`docker container rm -f $(docker container ls -q --filter label=io.kubernetes.container.name=hello-kiamol)`

check the Pod status:
`kubectl get pod hello-kiamol`

and find the container again:
`docker container ls -q --filter label=io.kubernetes.container.name=hello-kiamol`


Kubernetes reacted when I deleted my Docker container. For an instant, the Pod had zero containers, but Kubernetes immediately created a replacement to repair the Pod and bring it back to the correct state.


following the above exercise using crictl in containerd 

`kubectl run hello-kiamol --image=kiamol/ch02-hello-kiamol --restart=Always`

```
kubectl get pods -owide
NAME           READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
hello-kiamol   1/1     Running   0          46s   10.244.2.2   kind-worker   <none>           <none>
```

`docker exec -it kind-worker bash`

`crictl ps --label io.kubernetes.container.name=hello-kiamol`

`crictl rm $(crictl ps --label io.kubernetes.container.name=hello-kiamol)`

or try on worker node where the pod is scheduled

```
oot@kind-worker:/# crictl ps --label io.kubernetes.container.name=hello-kiamol
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID
b9663149439bc       8050f6e540117       13 minutes ago      Running             hello-kiamol        1                   2983352021392
root@kind-worker:/# crictl rm -f b9663149439bc
b9663149439bc
root@kind-worker:/# crictl ps --label io.kubernetes.container.name=hello-kiamol
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID
root@kind-worker:/# crictl ps --label io.kubernetes.container.name=hello-kiamol
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID
ba1c8c6cac11e       8050f6e540117       1 second ago        Running             hello-kiamol        2                   2983352021392
root@kind-worker:/# crictl ps --label io.kubernetes.container.name=hello-kiamol
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID
ba1c8c6cac11e       8050f6e540117       3 seconds ago       Running             hello-kiamol        2                   2983352021392

```

A failed container is a temporary fault; the Pod still exists, and the Pod can be brought back up to spec with a new container. 


# 8 Kubectl port-foward

Kubectl can forward traffic from a node to a Pod, which is a quick way to communicate with a Pod from outside the cluster. You can listen on a specific port on your machine—which is the single node in your cluster—and forward traffic to the application running in the Pod.

```
# listen on port 8080 on your machine and send traffic
# to the Pod on port 80:
kubectl port-forward pod/hello-kiamol 8080:80
```


# 9. Running pods with controllers (deployment controller)

Pods are too simple to be useful on their own; they are isolated instances of an application, and each Pod is allocated to one node. If that node goes offline, the Pod is lost, and Kubernetes does not replace it. You could try to get high availability by running several Pods, but there’s no guarantee Kubernetes won’t run them all on the same node. Even if you do get Pods spread across several nodes, you need to manage them yourself. Why do that when you have an orchestrator that can manage them for you?


That’s where controllers come in. A controller is a Kubernetes resource that manages other resources. It works with the Kubernetes API to watch the current state of the system, compares that to the desired state of its resources, and makes any changes necessary. Kubernetes has many controllers, but the main one for managing Pods is the Deployment, which solves the problems I’ve just described. If a node goes offline and you lose a Pod, the Deployment creates a replacement Pod on another node; if you want to scale your Deployment, you can specify how many Pods you want, and the Deployment controller runs them across many nodes.


Now we create another instance of the web application, this time using a Deployment.

```
# create a Deployment called "hello-kiamol-2", running the same web app:
kubectl create deployment hello-kiamol-2 --image=kiamol/ch02-hello-kiamol
deployment.apps/hello-kiamol-2 created

# list all the Pods:
kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
hello-kiamol                      1/1     Running   2          32m
hello-kiamol-2-7ddd98ffb5-qb5v2   1/1     Running   0          17s

```


# important takeaway

you created the Deployment, but you did not directly create the Pod. The Deployment specification described the Pod you wanted, and the Deployment created that Pod. The Deployment is a controller that checks with the Kubernetes API to see which resources are running, realizes the Pod it should be managing doesn’t exist, and uses the Kubernetes API to create it

# 10. Labels in deployments

How the deployment keeps track of its resources does matter, though, because it’s a pattern that Kubernetes uses a lot. Any Kubernetes resource can have labels applied that are simple key-value pairs. You can add labels to record your own data. For example, you might add a label to a Deployment with the name release and the value 20.04 to indicate this Deployment is from the 20.04 release cycle

```
# print the labels that the Deployment adds to the Pod:
kubectl get deploy hello-kiamol-2 -o jsonpath='{.spec.template.metadata.labels}'

# list the Pods that have that matching label:
kubectl get pods -l app=hello-kiamol-2
```

In this case, the Deployment adds a label called app with the value hello-kiamol-2 to the Pod. Querying Pods that have a matching label returns the single Pod managed by the Deployment.

# 11. Deployment recognises a pod by its label

The Deployment doesn’t have a direct relationship with the Pod it created; it only knows there needs to be one Pod with labels that match its label selector. If you edit the labels on the Pod, the Deployment no longer recognizes it.

```
# list all Pods, showing the Pod name and labels:
kubectl get pods -o custom-columns=NAME:metadata.name,LABELS:metadata.labels
NAME                              LABELS
hello-kiamol                      map[run:hello-kiamol]
hello-kiamol-2-7ddd98ffb5-qb5v2   map[app:hello-kiamol-2 pod-template-hash:7ddd98ffb5]

# update the "app" label for the Deployment’s Pod:
kubectl label pods -l app=hello-kiamol-2 --overwrite app=hello-kiamol-x
od/hello-kiamol-2-7ddd98ffb5-qb5v2 labeled

# fetch Pods again:
kubectl get pods -o custom-columns=NAME:metadata.name,LABELS:metadata.labels
NAME                              LABELS
hello-kiamol                      map[run:hello-kiamol]
hello-kiamol-2-7ddd98ffb5-mnsfb   map[app:hello-kiamol-2 pod-template-hash:7ddd98ffb5]
hello-kiamol-2-7ddd98ffb5-qb5v2   map[app:hello-kiamol-x pod-template-hash:7ddd98ffb5]

```


you will observe from the above overwriting the label of pod that Pod label effectively removes the Pod from the Deployment. At that point, the Deployment sees that no Pods that match its label selector exist, so it creates a new one. The Deployment has done its job, but by editing the Pod directly, you now have an unmanaged Pod.

`Useful for debugging ` : This can be a useful technique in `debugging—removing` a Pod from a controller so you can connect and investigate a problem, while the controller starts a replacement Pod, which keeps your app running at the desired scale. You can also do the opposite: editing the labels on a Pod to fool a controller into acquiring that Pod as part of the set it manages.

Lets try by setting the label back to its original one (that was set by deployment)

```
# list all Pods with a label called "app," showing the Pod name and
# labels:
kubectl get pods -l app -o custom-columns=NAME:metadata.name,LABELS:metadata.labels
NAME                              LABELS
hello-kiamol                      map[run:hello-kiamol]
hello-kiamol-2-7ddd98ffb5-mnsfb   map[app:hello-kiamol-2 pod-template-hash:7ddd98ffb5]
hello-kiamol-2-7ddd98ffb5-qb5v2   map[app:hello-kiamol-x pod-template-hash:7ddd98ffb5]

# update the "app" label for the the unmanaged Pod:
kubectl label pods -l app=hello-kiamol-x --overwrite app=hello-kiamol-2
pod/hello-kiamol-2-7ddd98ffb5-qb5v2 labeled

# fetch the Pods again:
kubectl get pods -l app -o custom-columns=NAME:metadata.name,LABELS:metadata.labels

NAME                              LABELS
hello-kiamol                      map[run:hello-kiamol]
hello-kiamol-2-7ddd98ffb5-qb5v2   map[app:hello-kiamol-2 pod-template-hash:7ddd98ffb5]
```

Now when the Deployment controller checks with the API, it sees two Pods that match its label selector. It’s supposed to manage only a single Pod, however, so it deletes one (using a set of deletion rules to decide which one)


# Port forward on deployment

You can configure the port forward on the Deployment resource, and the Deployment selects one of its Pods as the target.

`kubectl port-forward deploy/hello-kiamol-2 8080:80`


# 12. Doing deployments with application manifests

Manifests can be written in JSON or YAML; JSON is the native language of the Kubernetes API, but YAML is preferred for manifests because it’s easier to read

A simple app manifest of pod 

```
apiVersion: v1
kind: Pod
metadata:
  name: hello-kiamol-3
spec:
  containers:
    - name: web
      image: kiamol/ch02-hello-kiamol
```

```
kubectl apply -f pod.yaml 
pod/hello-kiamol-3 created
```

```
kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
hello-kiamol                      1/1     Running   2          99m
hello-kiamol-2-7ddd98ffb5-qb5v2   1/1     Running   0          67m
hello-kiamol-3                    1/1     Running   0          42s
```


A deployment manifest

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kiamol-4
spec:
  selector:
    matchLabels:
      app: hello-kiamol-4
  template:
    metadata:
      labels:
        app: hello-kiamol-4
    spec:
      containers:
        - name: web
          image: kiamol/ch02-hello-kiamol
```

```
kubectl apply -f deployment.yaml 
deployment.apps/hello-kiamol-4 created
```

# 13 Working with application in pods

You can run commands inside containers with kubectl and connect a terminal session, so you can connect into a Pod’s container as though you were connecting to a remote machine.

```
kubectl exec -it hello-kiamol-3 -- sh
/ # hostname -i
10.244.2.3
/ # wget -O - http://127.0.0.1 | head -n 4
Connecting to 127.0.0.1 (127.0.0.1:80)
writing to stdout
-                    100% |********************************************************************************************************************************************|   353  0:00:00 ETA
written to stdout
<html>
  <body>
    <h1>
      Hello from Chapter 2!
/ # 
```

# 14 Read logs of pods and containers using kubectl

`kubectl logs --tail=2 hello-kiamol`

```
kubectl logs --tail=2 hello-kiamol
127.0.0.1 - - [30/Nov/2021:10:38:24 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.45 Safari/537.36" "-"
127.0.0.1 - - [30/Nov/2021:10:38:54 +0000] "GET / HTTP/1.1" 200 353 "-" "Wget/1.21" "-"
```

same can be done for deployments 

You can run commands in Pods that are managed by a Deployment without knowing the Pod name, and you can view the logs of all Pods that match a label selector.

```
# make a call to the web app inside the container for the 
# Pod we created from the Deployment YAML file: 
kubectl exec deploy/hello-kiamol-4 -- sh -c 'wget -O - http://localhost > /dev/null'

# and check that Pod’s logs:
kubectl logs --tail=1 -l app=hello-kiamol-4
```


# 15 Kubectl copy 

```
# create the local directory:
mkdir -p /tmp/kiamol/ch02

# copy the web page from the Pod:
kubectl cp hello-kiamol:/usr/share/nginx/html/index.html /tmp/kiamol/ch02/index.html

# check the local file contents:
cat /tmp/kiamol/ch02/index.html
```


# 16 kubernetes resource deletion using kubectl


You can easily delete a Kubernetes resource using kubectl, but the resource might not stay deleted. If you created a resource with a controller, then it’s the controller’s job to manage that resource. It owns the resource life cycle, and it doesn’t expect any external interference. If you delete a managed resource, then its controller will create a replacement.


```
kubectl get pods

# delete all Pods:
kubectl delete pods --all

# check again:
kubectl get pods
```

Two of those Pods were created directly with the run command and with a YAML Pod specification. They don’t have a controller managing them, so when you delete them, they stay deleted. The other two were created by Deployments, and when you delete the Pod, the Deployment controllers still exist. They see there are no Pods that match their label selectors, so they create new ones.

If you want to delete a resource that is managed by a controller, you need to delete the controller instead. Controllers clean up their resources when they are deleted, so removing a Deployment is like a cascading delete that removes all the Deployment’s Pods, too.


```
# view Deployments:
ubectl get deploy
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hello-kiamol-2   1/1     1            1           105m
hello-kiamol-4   1/1     1            1           31m

# delete all Deployments:
kubectl delete deploy --all

# view Pods:
kubectl get pods
NAME                              READY   STATUS        RESTARTS   AGE
hello-kiamol-2-7ddd98ffb5-m5zt7   0/1     Terminating   0          3m31s
hello-kiamol-4-86b77c47dc-rgwmc   0/1     Terminating   0          3m31s

# check all resources:
kubectl get all
```











