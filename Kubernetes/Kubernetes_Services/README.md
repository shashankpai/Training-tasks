# 1.  Connecting pods over network with services

For example, you may have a website Pod and an API Pod, or you may have dozens of Pods in a microservice architecture. They all need to communicate, and Kubernetes supports the standard networking protocols, `TCP and UDP`. Both protocols use IP addresses to route traffic, but IP addresses change when Pods are replaced, so Kubernetes provides a network address discovery mechanism with `Services`.

Services are flexible resources that support routing traffic between Pods, into Pods from the world outside the cluster, and from Pods to external systems


How Kubernetes routes network traffic
You learned two important things about Pods in the previous chapter: a Pod is a virtual environment that has an IP address assigned by Kubernetes, and Pods are disposable resources whose lifetime is controlled by another resource. If one Pod wants to communicate with another, it can use the IP address. That’s problematic for two reasons, however: first, the IP address changes if the Pod is replaced, and second, there’s no easy way to find a Pod’s IP address—it can be discovered only using the Kubernetes API.

Exercise

If you deploy two Pods, you can ping one Pod from the other, but you first need to find its IP address.

```
kubectl apply -f sleep1.yaml -f sleep2.yaml 
deployment.apps/sleep-1 created
deployment.apps/sleep-2 created
```

```
kubectl get pods --show-labels
NAME                      READY   STATUS    RESTARTS   AGE   LABELS
sleep-1-8f68bd59d-4s7j5   1/1     Running   0          60s   app=sleep-1,pod-template-hash=8f68bd59d
sleep-2-bd8bc78c7-c7xb2   1/1     Running   0          60s   app=sleep-2,pod-template-hash=bd8bc78c7
```

```
kubectl wait --for=condition=Ready pod -l app=sleep-2
pod/sleep-2-bd8bc78c7-c7xb2 condition met
```

check the IP address of the second Pod:
```
kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'
10.244.2.5
```
use that address to ping the second Pod from the first:

```
kubectl exec deploy/sleep-1 -- ping -c 2 $(kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}')
PING 10.244.2.5 (10.244.2.5): 56 data bytes
64 bytes from 10.244.2.5: seq=0 ttl=62 time=0.179 ms
64 bytes from 10.244.2.5: seq=1 ttl=62 time=0.132 ms

--- 10.244.2.5 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.132/0.155/0.179 ms
```

These Pods are managed by Deployment controllers. If you delete the second Pod, its controller will start a replacement with a new IP address.

```
# check the current Pod’s IP address:
kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'
10.244.2.5

# delete the Pod so the Deployment replaces it:
kubectl delete pods -l app=sleep-2
pod "sleep-2-bd8bc78c7-c7xb2" deleted

# check the IP address of the replacement Pod:
kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'
10.244.2.6
```


the replacement Pod has a different IP address, and if I tried to ping the old address, the command would fail.

The problem of needing a permanent address for resources that can change is an old one—the internet solved it using DNS (the Domain Name System), mapping friendly names to IP addresses, and Kubernetes uses the same system. A Kubernetes cluster has a DNS server built in, which maps Service names to IP addresses


`Important:` Pods communicate using domain names . DNS lookups are handled by the internal kubernetes DNS server. It returns the IP address of the `Service`
Creating a service effectivley registers it with the DNS server using IP address that is static for the life of the Service

This type of Service is an abstraction over a Pod and its network address, just like a Deployment is an abstraction over a Pod and its container. The Service has its own IP address, which is static. When consumers make a network request to that address, Kubernetes routes it to the actual IP address of the Pod. The link between the Service and its Pods is set up with a label selector, just like the link between Deployments and Pods.


YAML specification for a Service, using the app label to identify the Pod which is the ultimate target of the network traffic.

```
apiVersion: v1
kind: Service
metadata:
  name: sleep-2
spec:
  selector:
    app: sleep-2
  ports:
    - port: 80
```

```
kubectl apply -f sleep-service.yaml 
service/sleep-2 created
```

Now we will ping the sleep-2 service from sleep-1 pod that was created earlier

```
kubectl exec deploy/sleep-1 --  ping -c 2 sleep-2
PING sleep-2 (10.96.203.161): 56 data bytes

--- sleep-2 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
command terminated with exit code 1
```

the sleep-1 pod does a domain name lookup and receives the Service IP address from K8s dns server

`important`:the ping command fails because it uses ICMP protocol which kubernetes service dont support , services do support standard TCP and UDP protocol


# 2. Routing traffic between pods 

The default type of Service in Kubernetes is called `ClusterIP`. It creates a clusterwide IP address that Pods on any node can access. The IP address works only within the cluster, so ClusterIP Services are useful only for communicating between Pods




# Exercise

Run two Deployments, one for the web application and one for the API. This app has no Services yet, and it won’t work correctly because the website can’t find the API.

```
/Kubernetes_Services/numbers$ kubectl apply -f web.yaml -f api.yaml 
deployment.apps/numbers-web created
deployment.apps/numbers-api created
```

```
kubectl get deploy 
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
numbers-api   0/1     1            0           7s
numbers-web   0/1     1            0           7s
sleep-1       1/1     1            1           53m
sleep-2       1/1     1            1           53m

```

```
kubectl port-forward deploy/numbers-web 8080:80
```

it will fail when trying to access with `ERROR`

```
RNG service unavailable!

(Using API at: http://numbers-api/sixeyed/kiamol/master/ch03/numbers/rng)

```

the error from web app shows that it cant reach the API , the domain name its using for API is `numbers-api` a local domain name that doesnt exists in kubernetes DNS server 

Now lets create a service for API pod ,below shows a Service with the correct name and a label selector that matches the API Pod.

```
apiVersion: v1
kind: Service
metadata:
  name: numbers-api
spec:
  ports:
    - port: 80
  selector:
    app: numbers-api
  type: ClusterIP
```

```
kubectl apply -f api-service.yaml 
service/numbers-api created
infracloud@infracloud-ThinkPad-E14-Gen-3:~/Practice/Training/Training-tasks/Kubernetes/Kubernetes_Services/numbers$ kubectl port-forward deploy/numbers-web 8080:80
```

output shows the app working correctly, with the website displaying a random-number generated by the API.

The API Pod is managed by a Deployment controller, so you can delete the Pod and a replacement will be created. The replacement is also a match for the label selector in the API Service, so traffic is routed to the new Pod, and the app keeps working.

this can be tried and tested 

```
# check the name and IP address of the API Pod:
kubectl get pod -l app=numbers-api -o custom-columns=NAME:metadata.name,POD_IP:status.podIP

# delete that Pod:
kubectl delete pod -l app=numbers-api

# check the replacement Pod:
kubectl get pod -l app=numbers-api -o custom-columns=NAME:metadata.name,POD_IP:status.podIP 

# forward a port to the web app:
kubectl port-forward deploy/numbers-web 8080:80
```

# 3. Routing External traffic to pods 

type of a Service called `LoadBalancer`, which solves the problem of getting traffic to a Pod that might be running on a different node from the one that received the traffic


