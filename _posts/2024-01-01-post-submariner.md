---
title: "Multi Cluster Networking with ACM - Submariner"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - ACM
  - Submariner
  - Network
---

When it comes to multi cluster networking in Kubernetes there are some alternatives out there.
I will try to cover [Submariner](https://submariner.io/) in this post. 

### Why you would need to connect multiple cluster at all?
 
There can be multiple use cases for multi cluster networking, the one which comes to mind easily is a Hybrid Cloud environment where some services are on Public cloud and some services are on premise clusters for different reasons. Another use case can be disaster recovery with ODF.

### What is Submariner?
>Submariner enables direct networking between Pods and Services in different Kubernetes clusters, either on-premises or in the cloud.

### What does Submariner provides?

Cross-cluster L3 connectivity with encryption.
> You can easily create secure tunnels between clusters.

Service Discovery accross clusters.
> Submariner allows service discovery between multiple clusters with Lighthouse dns.

### Getting started with Submariner

Before getting started with Submariner there are some prerequisites you should provide.

You should be providing IP Connectivity between gateway nodes. This can be public or private connection. In my setup I will be providing a private network for my two clusters. 
More information can be found from Submariner official documentation which gives examples about routing. [https://submariner.io/operations/nat-traversal/](https://submariner.io/operations/nat-traversal/)

If you are using OVN Kubernetes, OpenShift cluster version should be 4.12 or later.

If you are using OpenShift SDN, 4800 UDP port should be open between all cluster nodes.

When your clusters pod and service networks overlap with each other which is a common scenario, you should be using Globalnet, which handles overlapping CIDRs. Globalnet assigns additional CIDR for your clusters. It also creates additional ovn rules or iptable rules for SNAT and DNAT. Please see the documentation for more information about Globalnet. [https://submariner.io/getting-started/architecture/globalnet/](https://submariner.io/getting-started/architecture/globalnet/)


### Installing the Submariner Plugin

This procedure assumes  you to created a Cluster set on ACM with clusters you want to connect together.

My lab setup network configuration is below. Since pod and service network overlaps each other I need to use Globalnet controller. 

| Cluster Name  | Pod Network   | Service Network |
|---------------|---------------|-----------------|
| local-cluster | 10.128.0.0/14 | 172.30.0.0/16   |
| ocp5          | 10.128.0.0/14 | 172.30.0.0/16   |

You can find out what networks are in use for your cluster.
```yaml
$ oc get network cluster -o yaml
```

The below configuration install submariner add-on on cluster named ocp5 and local-cluster.  

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: submariner
  namespace: local-cluster
spec:
  installNamespace: submariner-operator
---
apiVersion: submarineraddon.open-cluster-management.io/v1alpha1
kind: SubmarinerConfig
metadata:
  name: submariner
  namespace: local-cluster
spec:
  gatewayConfig:
    gateways: 2 #You can increase the GW pod count which runs as active - passive mode.
  IPSecNATTPort: 4500 #Default port for IPSec NAT traversal.
  airGappedDeployment: false
  NATTEnable: true
  cableDriver: libreswan #default driver creates 4 IPSec tunnels. vxlan driver is unencrypted and not recommended.
  globalCIDR: ""
---
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: submariner
  namespace: ocp5
spec:
  installNamespace: submariner-operator
---
apiVersion: submarineraddon.open-cluster-management.io/v1alpha1
kind: SubmarinerConfig
metadata:
  name: submariner
  namespace: ocp5
spec:
  gatewayConfig:
    gateways: 2 #You can increase the GW pod count which runs as active - passive mode.
  IPSecNATTPort: 4500 #Default port for IPSec NAT traversal.
  airGappedDeployment: false
  NATTEnable: true
  cableDriver: libreswan #default driver creates 4 IPSec tunnels. vxlan driver is unencrypted and not recommended.
  globalCIDR: ""
---
apiVersion: submariner.io/v1alpha1
kind: Broker
metadata:
  name: submariner-broker
  namespace: all-broker
  labels:
    cluster.open-cluster-management.io/backup: submariner
spec:
  globalnetEnabled: true
  globalnetCIDRRange: 242.0.0.0/8 #CIDR for Globalnet.

```

When you apply this configuration on hub cluster, It will create a subscription for Submariner operator on target clusters which will install all necessary components for submariner.

```yaml
$ oc get pods -n submariner-operator
NAME                                             READY   STATUS    RESTARTS   AGE
submariner-addon-5845d6d894-9nm96                1/1     Running   0          98s
submariner-gateway-q7tk7                         1/1     Running   0          43s
submariner-globalnet-hb59v                       1/1     Running   0          43s
submariner-lighthouse-agent-5965695647-9fl78     1/1     Running   0          43s
submariner-lighthouse-coredns-665f8cb494-gjclf   1/1     Running   0          42s
submariner-lighthouse-coredns-665f8cb494-r8qh4   1/1     Running   0          42s
submariner-metrics-proxy-z99dl                   2/2     Running   0          43s
submariner-operator-5b67b9d946-5zdjn             1/1     Running   0          47s
submariner-routeagent-7dbj4                      1/1     Running   0          43s
submariner-routeagent-7qltp                      1/1     Running   0          43s
submariner-routeagent-b6kfl                      1/1     Running   0          43s
submariner-routeagent-j9jhm                      1/1     Running   0          43s
submariner-routeagent-kxxdd                      1/1     Running   0          43s
submariner-routeagent-vl7wz                      1/1     Running   0          43s
```


When you install submariner add-on for the managed cluster, the core-dns configuration also changes to provide service discovery between clusters. Please note the configuration below, which states Lighthouse dns service for queries  clusterset.local domain.


```yaml
$ oc get cm dns-default -o yaml -n openshift-dns
apiVersion: v1
data:
  Corefile: |
    # lighthouse
    clusterset.local:5353 {
        prometheus 127.0.0.1:9153
        forward . 172.30.74.59 {
            policy random
        }
        errors
        log . {
            class error
        }
        bufsize 1232
        cache 900 {
            denial 9984 30
        }
    }
    .:5353 {
        bufsize 1232
        errors
        log . {
            class error
        }
        health {
            lameduck 20s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus 127.0.0.1:9153
        forward . /etc/resolv.conf {
            policy sequential
        }
        cache 900 {
            denial 9984 30
        }
        reload
    }
    hostname.bind:5353 {
        chaos
    }
kind: ConfigMap
metadata:
  labels:
    dns.operator.openshift.io/owning-dns: default
  name: dns-default
  namespace: openshift-dns
```

Lighthouse dns service in submariner namespace.

```yaml
$ oc get svc -n submariner-operator | grep submariner-lighthouse-coredns
submariner-lighthouse-coredns           ClusterIP   172.30.74.59     <none>        53/UDP     5m48s
submariner-lighthouse-coredns-metrics   ClusterIP   172.30.169.141   <none>        9153/TCP   5m48s

```

After a successful deployment you should see green ticks on ACM GUI , you can now use service discovery between clusters. We will create a sample application to test this out.

![](/assets/images/clusterset.png)


Let's create an nginx app in one of OpenShift clusters. 
```yaml
$ oc new-project subm
$ oc new-app --name=nginx-sample nginx:1.20-ubi7~https://github.com/sclorg/nginx-ex.git
$ subctl export service --namespace subm nginx-sample #create serviceexport object.
$ oc get serviceexport nginx-sample  -o yaml # you should see serviceexport object.
```


From other cluster try to reach serviceexport url.
Where url schema is: <service>.<namespace>.svc.clusterset.local

```yaml
$ oc exec nginx-sample-545b9db55c-x726j --  curl nginx-sample.subm.svc.clusterset.local:8080
```

We can see that we now can access the service from another cluster.


With this demonstration you have seen how to connect services from different clusters with each other. Thanks to Advanced Cluster Management for Kubernetes, deployment and configuration is fairly easy when you provide underlying network connectivity.

If you need further information you can reach ACM - Network documentation and Submariner official documentation.

[https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.9/html/networking/index](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.9/html/networking/index)

[https://submariner.io/](https://submariner.io/)
