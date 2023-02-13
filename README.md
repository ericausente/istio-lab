# istio-lab

From https://istio.io/latest/docs/setup/getting-started/#download

Go to the Istio release page to download the installation file for your OS, or download and extract the latest release automatically (Linux or macOS)
Move to the Istio package directory. For example, if the package is istio-1.16.2:
Add the istioctl client to your path (Linux or macOS)
```
$ curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.16.2 TARGET_ARCH=x86_64 sh -
$ cd istio-1.16.2
$ export PATH=$PWD/bin:$PATH
$ cp bin/istioctl /usr/local/bin/
```

Install Istio
and add a namespace label to instruct Istio to automatically inject Envoy sidecar proxies when you deploy your application later:
```
istioctl install 
```

Output: 
```
root@eric-ubuntu-20:~/istio-1.16.2# istioctl install
This will install the Istio 1.16.2 default profile with ["Istio core" "Istiod" "Ingress gateways"] components into the cluster. Proceed? (y/N) y
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Installation complete                                                                                                                                      Making this installation the default for injection and validation.

Thank you for installing Istio 1.16.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/99uiMML96AmsXY5d6
```

You will notice that an istio-system namespace gets created: 
```
# kubectl get ns
NAME              STATUS   AGE
default           Active   7d22h
istio-system      Active   51s
kube-node-lease   Active   7d22h
kube-public       Active   7d22h
kube-system       Active   7d22h
```

Let's print out the pods (we have the IngressGateway and istiod pods [control plane which contains Pilot, Citadel and Galley]) 
```
# kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-6cf795f577-9mrjd   1/1     Running   0          82s
istiod-6746dbc8f6-f8c62                 1/1     Running   0          102s
```

Now, let's deploy a Microservices Application (from Google Cloud Repo)
They will all start in a 'default' namespace. 
A bunch of them will get created (will take 3-10 minutes for all the pods to come up and get into 'running' status). 
Take note that there is only 1 container for each pod that was created. 

```
kubectl create -f https://raw.githubusercontent.com/ericausente/google-microservices-demo/main/release/kubernetes-manifests.yaml
```

Output: 
```
# kubectl create -f https://raw.githubusercontent.com/ericausente/google-microservices-demo/main/release/kubernetes-manifests.yaml
deployment.apps/emailservice created
service/emailservice created
deployment.apps/checkoutservice created
service/checkoutservice created
deployment.apps/recommendationservice created
service/recommendationservice created
deployment.apps/frontend created
service/frontend created
service/frontend-external created
deployment.apps/paymentservice created
service/paymentservice created
deployment.apps/productcatalogservice created
service/productcatalogservice created
deployment.apps/cartservice created
service/cartservice created
deployment.apps/loadgenerator created
deployment.apps/currencyservice created
service/currencyservice created
deployment.apps/shippingservice created
service/shippingservice created
deployment.apps/redis-cart created
service/redis-cart created
deployment.apps/adservice created
service/adservice created

# kubectl get pods -o wide
NAME                                     READY   STATUS    RESTARTS   AGE   IP             NODE                 NOMINATED NODE   READINESS GATES
adservice-85464c648-dlggl                1/1     Running   0          11m   192.168.1.56   eric-ubuntu-worker   <none>           <none>
cartservice-59655cfc5f-p54jk             1/1     Running   0          11m   192.168.1.51   eric-ubuntu-worker   <none>           <none>
checkoutservice-b48888c64-9vqxj          1/1     Running   0          11m   192.168.1.46   eric-ubuntu-worker   <none>           <none>
currencyservice-75db674968-l5nwc         1/1     Running   0          11m   192.168.1.53   eric-ubuntu-worker   <none>           <none>
emailservice-768497cf5d-v9hjx            1/1     Running   0          11m   192.168.1.45   eric-ubuntu-worker   <none>           <none>
frontend-555565c6dd-lgqkw                1/1     Running   0          11m   192.168.1.47   eric-ubuntu-worker   <none>           <none>
loadgenerator-7668f8fcf7-hfh5f           1/1     Running   0          11m   192.168.1.52   eric-ubuntu-worker   <none>           <none>
paymentservice-99969bfd-lg68c            1/1     Running   0          11m   192.168.1.50   eric-ubuntu-worker   <none>           <none>
productcatalogservice-7ff5d874c6-7lbmf   1/1     Running   0          11m   192.168.1.49   eric-ubuntu-worker   <none>           <none>
recommendationservice-6d5554b8b-hmdhj    1/1     Running   0          11m   192.168.1.48   eric-ubuntu-worker   <none>           <none>
redis-cart-7f9cc97c69-5v6rb              1/1     Running   0          11m   192.168.1.55   eric-ubuntu-worker   <none>           <none>
shippingservice-5454bf7586-p5wqd         1/1     Running   0          11m   192.168.1.54   eric-ubuntu-worker   <none>           <none>

```

At this point, although Istio Core and Microservices and Running in our Kubernetes Cluster, Envoy Proxies are NOT YET injected. 
We did not yet explicitly tell ISIO to inject proxies into every pod that starts in the cluster. 
We actually have to configure that specifically and the configuration is very simple! 
What we do is basically LABEL A NAMESPACE with a LABEL called 'istio-injection=enabled'

```
kubectl label namespace default istio-injection=enabled
```

Output: 
```
# kubectl get ns --show-labels
NAME              STATUS   AGE     LABELS
default           Active   7d23h   istio-injection=enabled,kubernetes.io/metadata.name=default   --------->>> !!!!!
istio-system      Active   13m     kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   7d23h   kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   7d23h   kubernetes.io/metadata.name=kube-public
kube-system       Active   7d23h   kubernetes.io/metadata.name=kube-system
```

We can now shutdown all our microservices pods: 

```
kubectl delete -f https://raw.githubusercontent.com/ericausente/google-microservices-demo/main/release/kubernetes-manifests.yaml
```

Output: 
```
# kubectl delete -f https://raw.githubusercontent.com/ericausente/google-microservices-demo/main/release/kubernetes-manifests.yaml
deployment.apps "emailservice" deleted
service "emailservice" deleted
deployment.apps "checkoutservice" deleted
service "checkoutservice" deleted
deployment.apps "recommendationservice" deleted
service "recommendationservice" deleted
deployment.apps "frontend" deleted
service "frontend" deleted
service "frontend-external" deleted
deployment.apps "paymentservice" deleted
service "paymentservice" deleted
deployment.apps "productcatalogservice" deleted
service "productcatalogservice" deleted
deployment.apps "cartservice" deleted
service "cartservice" deleted
deployment.apps "loadgenerator" deleted
deployment.apps "currencyservice" deleted
service "currencyservice" deleted
deployment.apps "shippingservice" deleted
service "shippingservice" deleted
deployment.apps "redis-cart" deleted
service "redis-cart" deleted
deployment.apps "adservice" deleted
service "adservice" deleted

# kubectl get pods
No resources found in default namespace.

```

Let's re-apply it now. Instead of 1 container per pod, you will notice that will see 2 containers instead. 

```
# kubectl create -f https://raw.githubusercontent.com/ericausente/google-microservices-demo/main/release/kubernetes-manifests.yaml
deployment.apps/emailservice created
service/emailservice created
deployment.apps/checkoutservice created
service/checkoutservice created
deployment.apps/recommendationservice created
service/recommendationservice created
deployment.apps/frontend created
service/frontend created
service/frontend-external created
deployment.apps/paymentservice created
service/paymentservice created
deployment.apps/productcatalogservice created
service/productcatalogservice created
deployment.apps/cartservice created
service/cartservice created
deployment.apps/loadgenerator created
deployment.apps/currencyservice created
service/currencyservice created
deployment.apps/shippingservice created
service/shippingservice created
deployment.apps/redis-cart created
service/redis-cart created
deployment.apps/adservice created
service/adservice created

# kubectl get pods -o wide
NAME                                     READY   STATUS    RESTARTS   AGE   IP             NODE                 NOMINATED NODE   READINESS GATES
adservice-85464c648-xnxfh                2/2     Running   0          51s   192.168.1.68   eric-ubuntu-worker   <none>           <none>
cartservice-59655cfc5f-7nmjc             2/2     Running   0          52s   192.168.1.63   eric-ubuntu-worker   <none>           <none>
checkoutservice-b48888c64-fcxmt          2/2     Running   0          52s   192.168.1.57   eric-ubuntu-worker   <none>           <none>
currencyservice-75db674968-h5hn8         2/2     Running   0          52s   192.168.1.65   eric-ubuntu-worker   <none>           <none>
emailservice-768497cf5d-phvzd            2/2     Running   0          52s   192.168.1.60   eric-ubuntu-worker   <none>           <none>
frontend-555565c6dd-hx4kl                2/2     Running   0          52s   192.168.1.58   eric-ubuntu-worker   <none>           <none>
loadgenerator-7668f8fcf7-sdhhf           2/2     Running   0          52s   192.168.1.64   eric-ubuntu-worker   <none>           <none>
paymentservice-99969bfd-8lc6c            2/2     Running   0          52s   192.168.1.61   eric-ubuntu-worker   <none>           <none>
productcatalogservice-7ff5d874c6-mf9fp   2/2     Running   0          52s   192.168.1.62   eric-ubuntu-worker   <none>           <none>
recommendationservice-6d5554b8b-64sfs    2/2     Running   0          52s   192.168.1.59   eric-ubuntu-worker   <none>           <none>
redis-cart-7f9cc97c69-f8q2q              2/2     Running   0          51s   192.168.1.67   eric-ubuntu-worker   <none>           <none>
shippingservice-5454bf7586-j8g57         2/2     Running   0          51s   192.168.1.66   eric-ubuntu-worker   <none>           <none>

```

Let's see what's inside one of those pods. 
See that there is an INIT container which is ISTIO. 
That got automatically injected by Istio (it is not at all in Google's microservices yaml file. 
So that was done by Istio. 

```
# kubectl describe pod adservice-85464c648-xnxfh
Name:             adservice-85464c648-xnxfh
Namespace:        default
Priority:         0
Service Account:  default
Node:             eric-ubuntu-worker/10.201.10.178
Start Time:       Mon, 13 Feb 2023 22:18:38 +0700
Labels:           app=adservice
                  pod-template-hash=85464c648
                  security.istio.io/tlsMode=istio
                  service.istio.io/canonical-name=adservice
                  service.istio.io/canonical-revision=latest
Annotations:      cni.projectcalico.org/containerID: 568157e5b78a403740c66f2df7e985cd8356b378fa4c1aa964c44f67b1e19eb0
                  cni.projectcalico.org/podIP: 192.168.1.68/32
                  cni.projectcalico.org/podIPs: 192.168.1.68/32
                  kubectl.kubernetes.io/default-container: server
                  kubectl.kubernetes.io/default-logs-container: server
                  prometheus.io/path: /stats/prometheus
                  prometheus.io/port: 15020
                  prometheus.io/scrape: true
                  sidecar.istio.io/status:
                    {"initContainers":["istio-init"],"containers":["istio-proxy"],"volumes":["workload-socket","credential-socket","workload-certs","istio-env...
Status:           Running
IP:               192.168.1.68
IPs:
  IP:           192.168.1.68
Controlled By:  ReplicaSet/adservice-85464c648
Init Containers:
  istio-init:
    Container ID:  containerd://8e907d05a0c81a1632e236394f537694d893b819ca22a6b4ae8df4d4c712d364
    Image:         docker.io/istio/proxyv2:1.16.2
    Image ID:      docker.io/istio/proxyv2@sha256:e3122850ded1a844bd7d74afafbd3e388221c9e916c4e0ac47af8b644b939871
    Port:          <none>
    Host Port:     <none>
    Args:
      istio-iptables
      -p
      15001
      -z
      15006
      -u
      1337
      -m
      REDIRECT
      -i
      *
      -x

      -b
      *
      -d
      15090,15021,15020
      --log_output_level=default:info
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Mon, 13 Feb 2023 22:18:45 +0700
      Finished:     Mon, 13 Feb 2023 22:18:45 +0700
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     2
      memory:  1Gi
    Requests:
      cpu:        100m
      memory:     128Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-f2m2t (ro)
Containers:
  server:
    Container ID:   containerd://840d8007330548d0144427fa04d954f04065d22ef9da827eae58fac921f5c9e4
    Image:          gcr.io/google-samples/microservices-demo/adservice:v0.5.1
    Image ID:       gcr.io/google-samples/microservices-demo/adservice@sha256:645c53eab9c6b0f0a6604aaece6a21e9c2e952aec9c88765ede7a2f8f5015f5e
    Port:           9555/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 13 Feb 2023 22:18:46 +0700
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     300m
      memory:  300Mi
    Requests:
      cpu:      200m
      memory:   180Mi
    Liveness:   exec [/bin/grpc_health_probe -addr=:9555] delay=20s timeout=1s period=15s #success=1 #failure=3
    Readiness:  exec [/bin/grpc_health_probe -addr=:9555] delay=20s timeout=1s period=15s #success=1 #failure=3
    Environment:
      PORT:  9555
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-f2m2t (ro)
  istio-proxy:
    Container ID:  containerd://57c4b2bf91113814f5b92fd7a3d4f39ba3c4dff3a6cdb40c0781819ede61d79a
    Image:         docker.io/istio/proxyv2:1.16.2
    Image ID:      docker.io/istio/proxyv2@sha256:e3122850ded1a844bd7d74afafbd3e388221c9e916c4e0ac47af8b644b939871
    Port:          15090/TCP
    Host Port:     0/TCP
    Args:
      proxy
      sidecar
      --domain
      $(POD_NAMESPACE).svc.cluster.local
      --proxyLogLevel=warning
      --proxyComponentLogLevel=misc:error
      --log_output_level=default:info
      --concurrency
      2
    State:          Running
      Started:      Mon, 13 Feb 2023 22:18:47 +0700

# ... 

```

You can try to access your frontend website. But make sure to change the LoadBalancer Type into a Nodeport. 
kubectl edit svc frontend-external

```
# kubectl get svc
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
adservice               ClusterIP      10.104.146.185   <none>        9555/TCP       8m55s
cartservice             ClusterIP      10.98.110.3      <none>        7070/TCP       8m55s
checkoutservice         ClusterIP      10.97.119.82     <none>        5050/TCP       8m55s
currencyservice         ClusterIP      10.111.159.42    <none>        7000/TCP       8m55s
emailservice            ClusterIP      10.97.123.180    <none>        5000/TCP       8m55s
frontend                ClusterIP      10.98.13.228     <none>        80/TCP         8m55s
frontend-external       LoadBalancer   10.97.124.42     <pending>     80:31091/TCP   8m55s
kubernetes              ClusterIP      10.96.0.1        <none>        443/TCP        5h23m
paymentservice          ClusterIP      10.110.203.134   <none>        50051/TCP      8m55s
productcatalogservice   ClusterIP      10.98.177.66     <none>        3550/TCP       8m55s
recommendationservice   ClusterIP      10.101.36.45     <none>        8080/TCP       8m55s
redis-cart              ClusterIP      10.97.47.127     <none>        6379/TCP       8m55s
shippingservice         ClusterIP      10.97.189.86     <none>        50051/TCP      8m55s

# kubectl edit svc frontend-external

# kubectl get svc
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
adservice               ClusterIP   10.104.146.185   <none>        9555/TCP       9m35s
cartservice             ClusterIP   10.98.110.3      <none>        7070/TCP       9m35s
checkoutservice         ClusterIP   10.97.119.82     <none>        5050/TCP       9m35s
currencyservice         ClusterIP   10.111.159.42    <none>        7000/TCP       9m35s
emailservice            ClusterIP   10.97.123.180    <none>        5000/TCP       9m35s
frontend                ClusterIP   10.98.13.228     <none>        80/TCP         9m35s
frontend-external       NodePort    10.97.124.42     <none>        80:31091/TCP   9m35s   ----->>>>
kubernetes              ClusterIP   10.96.0.1        <none>        443/TCP        5h24m
paymentservice          ClusterIP   10.110.203.134   <none>        50051/TCP      9m35s
productcatalogservice   ClusterIP   10.98.177.66     <none>        3550/TCP       9m35s
recommendationservice   ClusterIP   10.101.36.45     <none>        8080/TCP       9m35s
redis-cart              ClusterIP   10.97.47.127     <none>        6379/TCP       9m35s
shippingservice         ClusterIP   10.97.189.86     <none>        50051/TCP      9m35s
```

# How about Monitoring, Data Visualization, Metrics Collection 
## This to see how are microservices are performing. 
### This is where ISTIO add-ons/integrations come in.See: 



