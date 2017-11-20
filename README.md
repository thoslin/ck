# deploy consul
Per instructions: https://github.com/kelseyhightower/consul-on-kubernetes

```
$ kubectl get pods
NAME                    READY     STATUS    RESTARTS   AGE
consul-0                1/1       Running   0          54m
consul-1                1/1       Running   0          51m
consul-2                1/1       Running   0          51m
```

# use consul as cilium kv store
```
          - "--kvstore"
          - "consul"
          - "--kvstore-opt"
          - "consul.address=100.125.138.131:8500"

```

# deploy cilium
```
$ kubectl get pods -n kube-system
NAME                                                    READY     STATUS    RESTARTS   AGE
calico-node-2z8sb                                       2/2       Running   0          4h
calico-node-7zl1w                                       2/2       Running   0          4h
calico-node-972pg                                       2/2       Running   0          4h
calico-node-cbrcx                                       2/2       Running   0          4h
calico-policy-controller-3446986630-8ff2k               1/1       Running   0          4h
cilium-72l81                                            1/1       Running   0          4m
cilium-9j6rk                                            1/1       Running   0          4m
cilium-dbz98                                            1/1       Running   0          4m
cilium-t0nr4                                            1/1       Running   0          4m
dns-controller-2354318375-vf4zh                         1/1       Running   0          4h
etcd-server-events-ip-172-20-55-217.ec2.internal        1/1       Running   0          4h
etcd-server-ip-172-20-55-217.ec2.internal               1/1       Running   0          4h
kube-apiserver-ip-172-20-55-217.ec2.internal            1/1       Running   14         4h
kube-controller-manager-ip-172-20-55-217.ec2.internal   1/1       Running   10         4h
kube-dns-1311260920-hh4kw                               3/3       Running   0          3h
kube-dns-1311260920-tf211                               3/3       Running   0          3h
kube-dns-autoscaler-1818915203-b26kp                    1/1       Running   0          4h
kube-proxy-ip-172-20-38-83.ec2.internal                 1/1       Running   0          4h
kube-proxy-ip-172-20-48-118.ec2.internal                1/1       Running   0          4h
kube-proxy-ip-172-20-53-198.ec2.internal                1/1       Running   0          4h
kube-proxy-ip-172-20-55-217.ec2.internal                1/1       Running   0          4h
kube-scheduler-ip-172-20-55-217.ec2.internal            1/1       Running   10         4h
```

# restart kube-dns pods
```
$ kubectl delete pod kube-dns-1311260920-hh4kw kube-dns-1311260920-tf211 -n kube-system
pod "kube-dns-1311260920-hh4kw" deleted
pod "kube-dns-1311260920-tf211" deleted
```

# deploy apps
http://cilium.readthedocs.io/en/latest/gettingstarted/

```
$ kubectl create -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/demo.yaml
service "app1-service" created
deployment "app1" created
deployment "app2" created
deployment "app3" created

$ kubectl get pods -w
NAME                    READY     STATUS    RESTARTS   AGE
app1-3720119688-0bnbm   1/1       Running   0          59s
app1-3720119688-hcw3w   1/1       Running   0          59s
app2-1798985037-3rh1b   1/1       Running   0          58s
app3-2097142386-1tcxq   1/1       Running   0          58s
consul-0                1/1       Running   0          20m
consul-1                1/1       Running   0          18m
consul-2                1/1       Running   0          17m
```

# check cilium endpoint list
```
$ kubectl exec cilium-72l81 -n kube-system -- cilium endpoint list
ENDPOINT   POLICY        IDENTITY   LABELS (source:key[=value])               IPv6                    IPv4           STATUS
           ENFORCEMENT                                                                                                        
14594      Disabled      257        k8s:id=app2                               f00d::6460:200:0:3902   100.96.2.120   ready
                                    k8s:io.kubernetes.pod.namespace=default                                                   
20084      Disabled      259        k8s:id=app1                               f00d::6460:200:0:4e74   100.96.2.150   ready
                                    k8s:io.kubernetes.pod.namespace=default

$ kubectl exec cilium-9j6rk -n kube-system -- cilium endpoint list
ENDPOINT   POLICY        IDENTITY   LABELS (source:key[=value])               IPv6                    IPv4           STATUS   
           ENFORCEMENT                                                                                                        
19080      Disabled      259        k8s:id=app1                               f00d::6460:300:0:4a88   100.96.3.188   ready    
                                    k8s:io.kubernetes.pod.namespace=default

$ kubectl exec cilium-dbz98 -n kube-system -- cilium endpoint list
ENDPOINT   POLICY        IDENTITY   LABELS (source:key[=value])               IPv6                    IPv4          STATUS   
           ENFORCEMENT                                                                                                       
12075      Disabled      258        k8s:id=app3                               f00d::6460:100:0:2f2b   100.96.1.79   ready    
                                    k8s:io.kubernetes.pod.namespace=default

$ kubectl exec cilium-t0nr4 -n kube-system -- cilium endpoint list
ENDPOINT   POLICY        IDENTITY   LABELS (source:key[=value])   IPv6   IPv4   STATUS   
           ENFORCEMENT                                                                   
```

# cilium log output
https://pastebin.com/ZL2sKJej

# apply network policies
```
$ kubectl create -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/l3_l4_policy.yaml
networkpolicy "access-backend" created

$ kubectl exec cilium-72l81 -n kube-system -- cilium endpoint list
ENDPOINT   POLICY        IDENTITY   LABELS (source:key[=value])               IPv6                    IPv4           STATUS   
           ENFORCEMENT                                                                                                        
14594      Disabled      257        k8s:id=app2                               f00d::6460:200:0:3902   100.96.2.120   ready    
                                    k8s:io.kubernetes.pod.namespace=default                                                   
20084      Enabled       259        k8s:id=app1                               f00d::6460:200:0:4e74   100.96.2.150   ready    
                                    k8s:io.kubernetes.pod.namespace=default

$ kubectl exec cilium-9j6rk -n kube-system -- cilium endpoint list
ENDPOINT   POLICY        IDENTITY   LABELS (source:key[=value])               IPv6                    IPv4           STATUS   
           ENFORCEMENT                                                                                                        
19080      Enabled       259        k8s:id=app1                               f00d::6460:300:0:4a88   100.96.3.188   ready    
                                    k8s:io.kubernetes.pod.namespace=default
```

$ kubectl exec cilium-dbz98 -n kube-system -- cilium endpoint list
ENDPOINT   POLICY        IDENTITY   LABELS (source:key[=value])               IPv6                    IPv4          STATUS   
           ENFORCEMENT                                                                                                       
12075      Disabled      258        k8s:id=app3                               f00d::6460:100:0:2f2b   100.96.1.79   ready    
                                    k8s:io.kubernetes.pod.namespace=default     
# test L3/L4 policies
$ APP2_POD=$(kubectl get pods -l id=app2 -o jsonpath='{.items[0].metadata.name}')
$ SVC_IP=$(kubectl get svc app1-service -o jsonpath='{.spec.clusterIP}')
$ kubectl exec $APP2_POD -- curl -s $SVC_IP
<html><body><h1>It works!</h1></body></html>
$ APP3_POD=$(kubectl get pods -l id=app3 -o jsonpath='{.items[0].metadata.name}')
$ kubectl exec $APP3_POD -- curl -s $SVC_IP

Request is hanged... Test passed.

# now test (L7 test is not passed, /private is hanged)
```
$ kubectl exec $APP2_POD -- curl -s http://${SVC_IP}/private
^C
```

And test after applying L7 policy also failed.

```
$ kubectl exec $APP2_POD -- curl -s http://${SVC_IP}/private
^C

$ kubectl exec $APP2_POD -- curl -s http://${SVC_IP}/public
^C


$ kubectl describe ciliumnetworkpolicies rule1
Name:         rule1
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cilium.io/v2
Kind:         CiliumNetworkPolicy
Metadata:
  Cluster Name:        
  Creation Timestamp:  2017-11-20T12:40:34Z
  Generation:          0
  Resource Version:    30702
  Self Link:           /apis/cilium.io/v2/namespaces/default/ciliumnetworkpolicies/rule1
  UID:                 00c3e71b-cdf0-11e7-91fe-0290b3808ecc
Spec:
  Endpoint Selector:
    Match Labels:
      Any : Id:  app1
  Ingress:
    From Endpoints:
      Match Labels:
        Any : Id:  app2
    To Ports:
      Ports:
        Port:      80
        Protocol:  TCP
      Rules:
        Http:
          Method:  GET
          Path:    /private
Status:
  Nodes:
    Ip - 172 - 20 - 38 - 83 . Ec 2 . Internal:
      Last Updated:  2017-11-20T12:40:34.684443167Z
      Ok:            true
    Ip - 172 - 20 - 48 - 118 . Ec 2 . Internal:
      Last Updated:  2017-11-20T12:40:34.688359575Z
      Ok:            true
    Ip - 172 - 20 - 53 - 198 . Ec 2 . Internal:
      Last Updated:  2017-11-20T12:40:34.683781341Z
      Ok:            true
    Ip - 172 - 20 - 55 - 217 . Ec 2 . Internal:
      Last Updated:  2017-11-20T12:40:34.679804446Z
      Ok:            true
Events:              <none>
```

Will try to do a retest of L7.

# Test kafka

```
$ kubectl create -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/kubernetes-kafka/kafka-sw-app.yaml
deployment "kafka-broker" created
deployment "zookeeper" created
service "zook" created
service "kafka-service" created
deployment "empire-hq" created
deployment "empire-outpost-8888" created
deployment "empire-outpost-9999" created
deployment "empire-backup" created

$ kubectl get pods,svc
NAME                                      READY     STATUS    RESTARTS   AGE
po/app1-3720119688-4p4mv                  1/1       Running   0          48m
po/app1-3720119688-ksn0f                  1/1       Running   0          48m
po/app2-1798985037-q548w                  1/1       Running   0          48m
po/app3-2097142386-wqx31                  1/1       Running   0          45m
po/consul-0                               1/1       Running   0          2h
po/consul-1                               1/1       Running   0          2h
po/consul-2                               1/1       Running   0          2h
po/empire-backup-1952792476-080cb         1/1       Running   0          1m
po/empire-hq-1131382786-t6gbf             1/1       Running   0          1m
po/empire-outpost-8888-2341181502-kg0wc   1/1       Running   0          1m
po/empire-outpost-9999-1113184742-cb2z1   1/1       Running   0          1m
po/kafka-broker-274039942-bwgm2           1/1       Running   2          1m
po/zookeeper-4191820466-qhddb             1/1       Running   0          1m

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                            AGE
svc/app1-service    ClusterIP   100.69.43.223   <none>        80/TCP                                                                             2h
svc/consul          ClusterIP   None            <none>        8500/TCP,8443/TCP,8400/TCP,8301/TCP,8301/UDP,8302/TCP,8302/UDP,8300/TCP,8600/TCP   4h
svc/kafka-service   ClusterIP   100.66.223.91   <none>        9092/TCP                                                                           1m
svc/kubernetes      ClusterIP   100.64.0.1      <none>        443/TCP                                                                            6h
svc/zook            ClusterIP   100.64.207.21   <none>        2181/TCP                                                                           1m

```

Test not passed. Probably something is wrong with the demo.

# Test with my own demo

## install kafka and zookeeper
```
$ kubectl get pods
NAME                    READY     STATUS    RESTARTS   AGE
app1-3720119688-4p4mv   1/1       Running   0          1h
app1-3720119688-ksn0f   1/1       Running   0          1h
app2-1798985037-q548w   1/1       Running   0          1h
app3-2097142386-wqx31   1/1       Running   0          1h
consul-0                1/1       Running   0          2h
consul-1                1/1       Running   0          2h
consul-2                1/1       Running   0          2h
kafka-0                 0/1       Running   3          2m
zk-0                    1/1       Running   0          3m
zk-1                    1/1       Running   0          2m
zk-2                    1/1       Running   0          1m
```

## deploy order service
```
```
