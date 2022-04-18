# istio-service-mesh
istio-service-mesh


Kubernetes is essentially about application lifecycle management through declarative configuration, while a service mesh is essentially about providing inter-application traffic, security management and observability. If you have already built a stable application platform using Kubernetes, how do you set up load balancing and traffic control for calls between services? This is where a service mesh comes into the picture.

**Kubernetes vs Service Mesh**

The following diagram shows the service access relationship in Kubernetes and service mesh (one sidecar per pod model).

![image](https://user-images.githubusercontent.com/74225291/163704081-41602c16-19f7-41e3-a0c8-c9a394e627da.png)


<img width="1366" alt="image" src="https://user-images.githubusercontent.com/74225291/163709184-271be4aa-31a1-48e1-bc88-5e907d03867f.png">

**Create nginx ingress controller**

    # k apply -f mandatory.yaml 

    # k apply -f ingress-service.yaml 

    # k  get pods -n ingress-nginx
    NAME                                        READY   STATUS    RESTARTS   AGE
    nginx-ingress-controller-65886f4f5d-4l7tz   1/1     Running   0          31s
    [root@ip-172-31-18-204 nginx-ingress-controller]# k get svc -n ingress-nginx
    NAME            TYPE           CLUSTER-IP     EXTERNAL-IP                                                               PORT(S)                      AGE
    ingress-nginx   LoadBalancer   10.100.20.83   abf852d2471084b5c91c8315b3c804ea-1090491327.us-east-2.elb.amazonaws.com   80:31978/TCP,443:31823/TCP   31s

**Deploy the sample application**


    # kubectl apply -f yelb_initial_deployment.yaml

    # kubectl get svc,pods -n yelb
    NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)        AGE
    service/redis-server     ClusterIP      10.100.163.225   <none>                                                                    6379/TCP       35s
    service/yelb-appserver   ClusterIP      10.100.251.130   <none>                                                                    4567/TCP       35s
    service/yelb-db          ClusterIP      10.100.111.138   <none>                                                                    5432/TCP       35s
    service/yelb-ui          LoadBalancer   10.100.129.55    a90cbe2d4db58410ba3a6004cea4e89e-2122463079.us-east-2.elb.amazonaws.com   80:30138/TCP   35s

    NAME                                 READY   STATUS    RESTARTS   AGE
    pod/redis-server-74556bbcb7-9rg8x    1/1     Running   0          35s
    pod/yelb-appserver-d584bb889-b2qdd   1/1     Running   0          35s
    pod/yelb-db-694586cd78-2x4xr         1/1     Running   0          35s
    pod/yelb-ui-798667d648-qz88h         1/1     Running   0          35s
    
    # k get ing -n yelb
    NAME           CLASS    HOSTS   ADDRESS                                                                   PORTS   AGE
    yelb-ingress   <none>   *       abf852d2471084b5c91c8315b3c804ea-1090491327.us-east-2.elb.amazonaws.com   80      139m

Now you can try to access App using ingress endpoint.

http://abf852d2471084b5c91c8315b3c804ea-1090491327.us-east-2.elb.amazonaws.com

<img width="1527" alt="image" src="https://user-images.githubusercontent.com/74225291/163771659-75beed95-e8fa-4744-8abe-552b8c2b7846.png">


**Install istio to kubernetes:**

Here we are installing it using istioctl, we can install it using helm as well. Will try that too afetr this one.

**Download Istio**

Go to the Istio release page to download the installation file for your OS, or download and extract the latest release automatically (Linux or macOS):

    curl -L https://istio.io/downloadIstio | sh -
    cd istio-1.13.2
    export PATH=$PWD/bin:$PATH

**Install Istio**

    $ istioctl install --set profile=demo -y
    
      ✔ Istio core installed
      ✔ Istiod installed
      ✔ Egress gateways installed
      ✔ Ingress gateways installed
      ✔ Installation complete

Add a namespace label to instruct Istio to automatically inject Envoy sidecar proxies when you deploy your application later:

    kubectl label namespace yelb istio-injection=enabled

Now as we have installed istio and enabled istio-injection lable for yelb namespace as well. We need to delete the extisting pods, so that new pods will spawn with envoy sidecar attached to it.

    # k delete pod redis-server-74556bbcb7-9rg8x yelb-appserver-d584bb889-b2qdd yelb-db-694586cd78-2x4xr yelb-ui-798667d648-qz88h   -n yelb
    pod "redis-server-74556bbcb7-9rg8x" deleted
    pod "yelb-appserver-d584bb889-b2qdd" deleted
    pod "yelb-db-694586cd78-2x4xr" deleted
    pod "yelb-ui-798667d648-qz88h" deleted
    
    # k get pods -n yelb
    NAME                             READY   STATUS    RESTARTS   AGE
    redis-server-74556bbcb7-wj5x7    2/2     Running   0          45s
    yelb-appserver-d584bb889-tx68m   2/2     Running   0          45s
    yelb-db-694586cd78-g2f7n         2/2     Running   0          45s
    yelb-ui-798667d648-6g5ct         2/2     Running   0          45s

Now we need to create gateway and virtualservice for it, so that we can access it using istio ingress.


      # k apply -f yelb-gateway.yaml  -n yelb
      gateway.networking.istio.io/yelb-gateway created
      virtualservice.networking.istio.io/yelb created
      
      # k get gateway -n yelb
      NAME           AGE
      yelb-gateway   6s

      # k get vs -n yelb
      NAME   GATEWAYS           HOSTS   AGE
      yelb   ["yelb-gateway"]   ["*"]   6s

Kiali Dashboard:

<img width="1531" alt="image" src="https://user-images.githubusercontent.com/74225291/163760694-3f7ca0a8-1685-4c66-aabe-66f07b9b951b.png">
