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
    - instance specific configuration using instance image logic
    - instance specific secrets using instance image logic
6. Rollback support for image upgrades


## Cluster Setup 


![ZookeeperSet](pictures/zookeeperset.png "ZookeeperSet")


## Basic Configuration

The ZookeeperSet deploys SERVERS="replicas" number of instances.

$SERVERS is available at start time for all instances .

All zookeeper instances receive hostnames to be zk-{$ORD} (like zk-0, zk-1 ) where $ORD is 0..$SERVERS-1.
All zookeeper instances have the same local dns domain.
The instances can access the host name and the domain in the following way:
```
HOST=`hostname -s`
DOMAIN=`hostname -d`  # identical for all
```
With this values the image can construct the zookeeper config file. 


## Ordered Automatic Rollout

![Ordered Rollout](pictures/orderedrollout.png "Ordered ROllout")

## Deploying the Operator

```
# deploy the latest version 
make deploy
# alternatively a specific version: 
# make docker-build docker-push deploy IMAGE_TAG_BASE='jankul02/zookeeperset' VERSION='0.0.151" 
```
check if the operator is running:
```
kubectl get pods -n zookeeperset-system
kubectl logs zookeeperset-controller-manager-6dcd55bf77-xj2rj -n zookeeperset-system manager

```

### Deploy a ZookeeperSet (Basics)

Create the resource definitions (a ZookeeperSet CR) 

A file like zookeeperset-lab.yaml:

```yaml
apiVersion: dataproxy.jankul02/v1alpha1
kind: ZookeeperSet
metadata:
  name: zookeeperset-lab
  namespace: default
spec:
  replicas: 3
  app: "zk"
  image: jankul02/kubernetes-zookeeper:0.0.25
  data:
    cfg: |
      LOG_LEVEL=INFO
      WELCOME_MESSAGE="Hey Hey for all"
      ADDITIONAL_PARAM=" Value"
```

Apply the CR defintion.

```
kubectl apply -f zookeeperset-lab.yaml

# observe the deployment
kubectl get pods -w 

```

## Showcases

Assumed:

1. 4 nodes cluster (minikube start --nodes 4)
2. 3 zookeeper replicas (replicas: 3)

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

# kubectl exec zk-0 -- zookeeper-shell localhost:2181 create /mykey "Hello World"

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
remove the check to force fail the check
```
kubectl exec zk-0 -- rm /etc/confluent/docker/zookeeper-ready
kubectl get pod -w  
```

### Deployment: No Single Point of Failure

Instances are automatically deployed on different nodes of the cluster
There is no collocation on one node
affinity / antiaffinity settings

```
for i in 0 1 2; do kubectl get pod zk-$i --template {{.spec.nodeName}}; echo ""; done
```

### Protection against Node outage

![Node Outage](pictures/zookeepersetnodeoutage.png "Node Outage")

```
kubectl get pdb zk-pdb
# show the pod/node assoc
for i in 0 1 2; do kubectl get pod zk-$i --template {{.spec.nodeName}}; echo ""; done
# drain zk-2 node
kubectl drain $(kubectl get pod zk-2 --template {{.spec.nodeName}}) --ignore-daemonsets --force --delete-emptydir-data

kubectl get pods -w -l # watch

# kubectl drain succeeds and the zk-2 is rescheduled to another node
```


### Protection against maintenance failures

Disruption protection:

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

Change the image name in the zookeeperset resource file and then apply it and observe

```
kubectl apply -f config/samples/dataproxy_v1alpha1_zookeeperset.yaml

kubectl get pods -w -l # to observe the rollout 
```

### Instance Configuration
 
1. change Configuration of the zk-2 instance in the zookeeperset resource file

```
kubectl apply -f config/samples/dataproxy_v1alpha1_zookeeperset.yaml

kubectl get pods -w -l # to observe the restart zk-2 only 

# view the beginning of the log 
kubectl logs zk-2

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

## Advanced Configuration

The defaults for all instances ("global") are by defined under:
```yaml
data:
    cfg: |
      LOG_LEVEL=INFO
      WELCOME_MESSAGE="Hey Hey for all"
      ADDITIONAL_PARAM=" Value"
``` 
and will be mounted under:
```
/mnt/zk-global
```

By conventions the instance specific configuration should be named ${ORD}-1 and specified under spec.data like

```yaml
  spec:
    data:
      ZOOKEEPER_TOOLS_LOG4J_LOGLEVEL=DEBUG
      ZOOKEEPER_LOG4J_ROOT_LOGLEVEL=DEBUG
      WELCOME_MESSAGE="Hey Hey for all"
      ADDITIONAL_PARAM=" Value"
    0: |
      ZOOKEEPER_TOOLS_LOG4J_LOGLEVEL=INFO
      ZOOKEEPER_LOG4J_ROOT_LOGLEVEL=INFO
      WELCOME_MESSAGE="Hula hop hej hej 0"
      ADDITIONAL_PARAM=" 231122"
    2: |
      ZOOKEEPER_TOOLS_LOG4J_LOGLEVEL=WARN
      ZOOKEEPER_LOG4J_ROOT_LOGLEVEL=WARN
      WELCOME_MESSAGE="Hi Hi Ho 2"
      ADDITIONAL_PARAM=" 23134122" 
```
The configuration will be mounted as a file with the name of ${ORD}-1 under /mnt/zk-global/:
```
cat /mnt/zk-config/0 | grep WELCOME_MESSAGE
WELCOME_MESSAGE="Hula hop hej hej 0"
```

It is the responisibility of the instance to use the needed configuration in a secure way (like avoiding shell sourcing). 

By conventions the instance specific secrets should be named zksecrets for common secrets and zksecrets0, zksecrets1 ... for instance specific secrets  and specified under spec.data like:
```yaml
spec:
  secrets:
  - secretname: zksecrets
    items:
    - key: username
      path: zksecrets 
    - key: password
      path: zksecrets 
  - secretname: zksecrets0
    items:
    - key: username
      path: zksecrets0 
    - key: password
      path: zksecrets0 
```
The secrets will be mounted as files with the name as their secretname under /mnt/:
```
ls /mnt/zksecrets0
```

## Advanced Configuration Deployment

Create resource definitions:

- all needed secrets
- a ZookeeperSet CR 

zksecrets.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: zksecrets
  namespace: default
  labels:
    app: zk
type: Opaque
data:
  username: YWRtaW4=
  password: ZGZhcXFkcXFhcXEzNDUyOA==  
```
zksecrets2.yaml 
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: zksecrets2
  namespace: default
  labels:
    app: zk
type: Opaque
data:
  username: YWRtaW4=
  password: YXFxZHFxYXFxMzQ1Mjg=
  api: aHR0cHM6Ly9hcGkuZXhhbXBsZS5jb20=
```

zksecrets0.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: zksecrets0
  namespace: default
  labels:
    app: zk
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

 and a file like zookeeperset-lab.yaml
```yaml
apiVersion: dataproxy.jankul02/v1alpha1
kind: ZookeeperSet
metadata:
  name: zookeeperset-lab
  namespace: default
spec:
  replicas: 3
  app: "zk"
  image: jankul02/kubernetes-zookeeper:0.0.25
  data:
    cfg: |
      ZOOKEEPER_TOOLS_LOG4J_LOGLEVEL=DEBUG
      ZOOKEEPER_LOG4J_ROOT_LOGLEVEL=DEBUG
      WELCOME_MESSAGE="Hey Hey for all"
      ADDITIONAL_PARAM=" Value"
    0: |
      ZOOKEEPER_TOOLS_LOG4J_LOGLEVEL=INFO
      ZOOKEEPER_LOG4J_ROOT_LOGLEVEL=INFO
      WELCOME_MESSAGE="Hula hop hej hej 0"
      ADDITIONAL_PARAM=" 231122"
    2: |
      ZOOKEEPER_TOOLS_LOG4J_LOGLEVEL=WARN
      ZOOKEEPER_LOG4J_ROOT_LOGLEVEL=WARN
      WELCOME_MESSAGE="Hi Hi Ho 2"
      ADDITIONAL_PARAM=" 23134122" 
  secrets:
  - secretname: zksecrets
    items:
    - key: username
      path: zksecrets 
    - key: password
      path: zksecrets 
  - secretname: zksecrets0
    items:
    - key: username
      path: zksecrets0 
    - key: password
      path: zksecrets0 
  - secretname: zksecrets2
    items:
    - key: username
      path: zksecrets2
    - key: password              
      path: zksecrets2
    - key: api              
      path: zksecrets2
```

Deploy the secrets and the ZookeeperSet
```
kubectl apply -f zksecrets.yaml
kubectl apply -f zksecrets0.yaml
kubectl apply -f zksecrets2.yaml


kubectl apply -f zookeeperset-lab.yaml

# observe the deployment
kubectl get pods -w 

```

## Scaling **not implemented**

(Re)scalling is **not implemented** and there is no protection against changing the value of replicas.
The results of scalling are unknown and currently unpredictable.


```
#cert manager
curl -L -o kubectl-cert-manager.tar.gz https://github.com/jetstack/cert-manager/releases/latest/download/kubectl-cert_manager-linux-amd64.tar.gz
tar xzf kubectl-cert-manager.tar.gz
sudo mv kubectl-cert_manager /usr/local/bin

kubectl cert-manager x install
````