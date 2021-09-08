# ZookeeperSet - operator

## Overview

The Zookeeperset Operator deploys a zookeeper cluster and automatically operates the cluster.

The Operator implements:

1. High Availability (HA) 
    - instance distribution to different nodes
    - automatic recovery for zookeeper instances
    - Quorum Protection (Maintenance Fault Protection)
    - liveness/health checks for instances
2. Data Persistance
3. Automatic network setup
4. Seameless upgrades/rollout with no disruption
5. Full Lifecycle management of the zookeeper cluster: 
    - automatic cluster deployment
    - autoconfiguration during cluster deployment
    - minor version upgrades for the cluster
    - instance specific configuration
    - instance specific secrets
6. Rollback support for image upgrades


## Cluster Setup 


![ZookeeperSet](pictures/zookeeperset.png "ZookeeperSet")


## Configuration
The ZookeeperSet deploys SERVERS="replicas" number of instances.

$SERVERS is available at start time.

All zookeeper instances receive hostnames to be zk-{$ORD} (like zk-0, zk-1 ) where $ORD is 0..$SERVERS.
All zookeeper instances have the same domain.
The instances can access the host name and the domain in the following way:
```
HOST=`hostname -s`
DOMAIN=`hostname -d`  # identical for all
```
### Global config

[TODO: global cluster config - start parameters to be implemented]
### Individual Configs for Zookeeper Instances


For each zookeeper instance an implicit dedicated initially empty ConfigMap is created and mounted as file at /mnt/`hostname -s` (like zk-0, zk-1 ).
It is the responsibility of the zookeeper image to create its own zoo.cfg from the image default parameters, global deployment parameters and from its specific ConfigMap.
It is the responsibility of the image to read its corresponding configuration from its file mnt/zk-{$ORD}, for example
```
source /mnt/zk-2
```

Applying ZkConfig (i.e. zk-2) redefines the content of the individual instance configuration. In such a case the instance (the pod i.e. zk-2) will be restarted if the zookeeper quorum would be preserved. Other instances will not be indluenced.

![ZkConfig](pictures/zkconfig.png "ZkConfig")

## Ordered Rollout

![Ordered Rollout](pictures/orderedrollout.png "Ordered ROllout")



## Deploying the operator
```
# deploy the latest version 
make deploy
# alternatively a specific version: 
# make deploy VERSION='0.0.32" 
```
check if the operator is running:
```
kubectl get pods -n zookeeperset-system
kubectl logs zookeeperset-controller-manager-6dcd55bf77-xj2rj -n zookeeperset-system manager
```

### Deploy a ZookeeperSet

Create two definitions:

- a ZookeeperSet CR and if required 
- the individual ZkConfigs for each instance

zookeeperset-lab.yaml
```yaml
apiVersion: dataproxy.jankul02/v1alpha1
kind: ZookeeperSet
metadata:
  name: zookeeperset-lab
  namespace: default
spec:
    replicas: 3
    image: jankul02/kubernetes-zookeeper:0.0.9
```

zkconfig-zk0.yaml
```yaml
apiVersion: dataproxy.jankul02/v1alpha1
kind: ZkConfig
metadata:
  name: zkconfig-0
  namespace: default
spec:
  zkid: "0"
  data:
    cfg: |
      LOG_LEVEL=DEBUG
      WELCOME_MESSAGE="buena vista Guten Abend hola mundo from zk-0"
      ADDITIONAL_PARAM="12354678"
```

zkconfig-zk2.yaml
```yaml
apiVersion: dataproxy.jankul02/v1alpha1
kind: ZkConfig
metadata:
  name: zkconfig-2
  namespace: default
spec:
  zkid: "2"
  data:
    cfg: |
      LOG_LEVEL=INFO
      WELCOME_MESSAGE="Dzien dobry from zk-2"
      ANOTHER_ADDITIONAL_PARAM="12354678"
```

Deploy the zkconfig resources

```
kubectl create -f zkconfig-zk2.yaml
kubectl create -f zkconfig-zk0.yaml
# the zk-1 instance will have no specific setting

kubectl get zkconfigs 
```
Deploy the ZookeeperSet
```
kubectl create -f zookeeperset-lab.yaml

# observe the deployment
kubectl get pods -w 

```







## Showcases

Assumed:
1. 4 nodes cluster
2. 3 zookeeper replicas


### Network setup
#### Show myid files:
```
for i in 0 1 2; do echo "myid zk-$i";kubectl exec zk-$i -- cat /var/lib/zookeeper/data/myid; done
```

#### Show fqdn

```
for i in 0 1 2; do kubectl exec zk-$i -- hostname -f; done
```

### functional case: write to zk-0 and get it back from zk-1
```
kubectl exec zk-0 zkCli.sh create /mykey "hello world"

kubectl exec zk-1 zkCli.sh get /mykey
```

### operational test: rollout and rollback

change a cpu assignment
```
kubectl patch sts zk --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/cpu", "value":"0.3"}]'

kubectl get pods -w # to observe the rollout 

kubectl rollout status sts/zk
```

### start a rollback:
```
kubectl rollout undo sts/zk

kubectl get pods -w # to observe the rollout 

kubectl rollout status sts/zk
```

## HA

### Recovering from a failure:
RestartPolicy=Always
```
kubectl exec zk-0 -- ps -ef

kubectl exec zk-0 -- pkill java

kubectl get pod -w -l 

```

### Liveness
remove check
```
kubectl exec zk-0 -- rm /usr/bin/zookeeper-ready
kubectl get pod -w -l 
```

### Deployment: No Single Point of Failure
Instances automatically deployed on different nodes
No collocation on one node
affinity / antiaffinity settings
```
for i in 0 1 2; do kubectl get pod zk-$i --template {{.spec.nodeName}}; echo ""; done
```


### Protection against maintenance failures
disruption protection

```
kubectl get pdb zk-pdb
# show the pod/node assoc
for i in 0 1 2; do kubectl get pod zk-$i --template {{.spec.nodeName}}; echo ""; done
# drain zk-0 node
kubectl drain $(kubectl get pod zk-0 --template {{.spec.nodeName}}) --ignore-daemonsets --force --delete-emptydir-data

kubectl get pods -w -l # watch


# kubectl drain succeeds and the zk-0 is rescheduled to another node


# drain another node
kubectl drain $(kubectl get pod zk-1 --template {{.spec.nodeName}}) --ignore-daemonsets --force --delete-emptydir-data "kubernetes-node-ixsl" cordoned

kubectl get pods -w -l # watch
# drain again 
kubectl drain $(kubectl get pod zk-2 --template {{.spec.nodeName}}) --ignore-daemonsets --force --delete-emptydir-data

# drain will be refused


# check zookeeper still working 
kubectl exec zk-0 zkCli.sh get /mykey

# uncordon
kubectl uncordon node-1231 

```

## Deployment and configuration

### Deployment
 
1. change the image in the zookeeperset resource file
2. apply:
```
kubectl apply -f config/samples/dataproxy_v1alpha1_zookeeperset.yaml

kubectl get pods -w -l # to observe the rollout 
```

### Instance Configuration
 
1. change Configuration of the zk-0 instance in the zkconfig resource file
2. apply

```
kubectl apply -f config/samples/dataproxy_v1alpha1_zkconfig.yaml

kubectl get pods -w -l # to observe the restart zk-0 only 

# view the beginning of the log 
kubectl logs zk-0

```

# Helpful oneliners

### Build, push, deploy the operator 
```
make docker-build docker-push deploy VERSION=0.0.38
```

### create a zookeeperset from sample 
```
kubectl apply -f config/samples/dataproxy_v1alpha1_zookeeperset.yaml

kubectl get pods -A

kubectl logs zookeeperset-controller-manager-6dcd55bf77-5pqps -n zookeeperset-system manager
```
### delete the zookeeperset created from sample 
```
kubectl delete -f config/samples/dataproxy_v1alpha1_zookeeperset.yaml
```

### undeploy the zookeeperset operator 
```
make undeploy
```