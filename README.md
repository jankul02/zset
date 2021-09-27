# ZookeeperSet - operator

## Overview

The Operator deploys a Kafka cluster with a zookeeper cluster  and automatically operates the cluster.

The Operator implements:

1. High Availability (HA)
    - instance distribution among the k8s nodes 
    - automatic recovery for kafka and zookeeper instances
    - Quorum Protection (Maintenance Fault Protection)
    - liveness/health checks for instances
2. Data Persistance
3. Automatic network setup
4. Seameless upgrades/rollout with no disruption
5. Full Lifecycle management of the zookeeper cluster: 
    - automatic cluster deployment
    - autoconfiguration during cluster deployment
    - minor version upgrades for the cluster
    - instance specific configuration incl. instance specific secrets using pod presets
    - clusterwide  configuration 


## Cluster Setup 


![ZookeeperSet](pictures/zookeeperset.png "ZookeeperSet")

## Basic Configuration

### Scaling and Autoscaling kafka
For the kafka cluster scalling and autoscalling are implemented.
kafka.spec.autoscale.behavior defines the scaledown/up policies (steps and time periods for metrics and reactions)

```yaml
    autoscale:
      minReplicas: 3
      maxReplicas: 5
      initial: 3
      targetCPUUtilizationPercentage: 30
      resources:
        requests: 
          # memory: "64Mi"
          cpu: "200m"
        limits:
          # memory: "128Mi"
          cpu: "300m"
      behavior:
        scaleDown: 
          policies: 
          - type: Pods 
            value: 1
            periodSeconds: 60 
          - type: Percent
            value: 10 
            periodSeconds: 60
          selectPolicy: Min 
          stabilizationWindowSeconds: 100
        scaleUp: 
          policies:
          - type: Pods
            value: 1
            periodSeconds: 70
          - type: Percent
            value: 12 
            periodSeconds: 80
          selectPolicy: Max
          stabilizationWindowSeconds: 100
```          

### Scaling kafka and PDB protection


1. For the zookeeper cluster autoscaling is **NOT implemented** (and potentially will not be).
2. The "kafka.spec.autoscale.initial" value is treated as the required number of replicas.
3. A Disruption Budget of instances is set for the zookeeper cluster to be maxUnavailable 1

The insatnces automatically calculate the servers settings during rollout (like for new image) 


## Ordered Automatic Rollout

Is implemented for both kafka and zookeeper clusters.


![Ordered Rollout](pictures/orderedrollout.png "Ordered ROllout")

## Deploying the Operator

```
# deploy the latest version 
make deploy
# alternatively a specific version: 
# make docker-build docker-push deploy IMAGE_TAG_BASE='jankul02/zookeeperset' VERSION='0.0.421" 
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
  namespace: kafka
spec:
  zookeeper:
    app: zk
    autoscale:
      range:
        min: 3
        max: 3
      target: 3
    maxUnavailable: 1
    updateOnChange: {}
    image: jankul02/kubernetes-zookeeper:0.0.67
    spec:
      env:
        - name: WELCOME_MESSAGE
          value: "The generic Hello World from zookeeper"      
        - name: ZOOKEEPER_TOOLS_LOG4J_LOGLEVEL
          value: DEBUG
        - name: ZOOKEEPER_LOG4J_ROOT_LOGLEVEL
          value: DEBUG
        - name: ZOOKEEPER_CLIENT_PORT
          value: "2181"
        - name: ZOOKEEPER_TICK_TIME
          value: "2000"
        - name: ZOOKEEPER_SECURE_CLIENT_PORT
          value: "2182"
        - name: ZOOKEEPER_SSL_QUORUM
          value: "true"
        - name: ZOOKEEPER_SSL_QUORUM_KEYSTORE_LOCATION
          value: /mnt/keystore.jks
        - name: ZOOKEEPER_SSL_QUORUM_TRUSTSTORE_LOCATION
          value: /mnt/truststore.jks
        - name: ZOOKEEPER_SSL_QUORUM_HOSTNAME_VERIFICATION
          value: "false"
        - name: ZOOKEEPER_SSL_HOSTNAME_VERIFICATION
          value: "false"
        - name: ZOOKEEPER_SERVER_CNXN_FACTORY
          value: org.apache.zookeeper.server.NettyServerCnxnFactory
        - name: ZOOKEEPER_SSL_KEYSTORE_LOCATION
          value: /mnt/keystore.jks
        - name: ZOOKEEPER_SSL_KEYSTORE_TYPE
          value: JKS
        - name: ZOOKEEPER_SSL_TRUSTSTORE_LOCATION
          value: /mnt/truststore.jks
        - name: ZOOKEEPER_SSL_TRUSTSTORE_TYPE
          value: JKS
        - name: ZOOKEEPER_CLIENT_PORT_UNIFICATION
          value:  "true"
        - name: ZOOKEEPER_SSL_CLIENT_AUTH
          value:  none
        - name: ZOOKEEPER_AUTH_PROVIDER_X509
          value: org.apache.zookeeper.server.auth.X509AuthenticationProvider
        - name: ZOOKEEPER_AUTH_PROVIDER_SASL
          value: org.apache.zookeeper.server.auth.SASLAuthenticationProvider
    instances:
    - id: "0"
      nodeSelector:
        zkinstanceid: "0"
        updateOnChange: {}
      spec:
        env: 
          - name: ZOOKEEPER_SSL_QUORUM_TRUSTSTORE_PASSWORD
            valueFrom:
              secretKeyRef:
                name: keystorepasswd-zk-0
                key: keystorepassword
          - name: ZOOKEEPER_SSL_QUORUM_KEYSTORE_PASSWORD
            valueFrom:
              secretKeyRef:
                name: keystorepasswd-zk-0
                key: keystorepassword
          - name: ZOOKEEPER_SSL_KEYSTORE_PASSWORD
            valueFrom:
              secretKeyRef:
                name: keystorepasswd-zk-0
                key: keystorepassword
          - name: ZOOKEEPER_SSL_TRUSTSTORE_PASSWORD
            valueFrom:
              secretKeyRef:
                name: keystorepasswd-zk-0
                key: keystorepassword
        volumeMounts:
          # name must match the volume name below
          - name: keystore
            mountPath: /mnt
          # The secret data is exposed to Containers in the Pod through a Volume.
        volumes:
          - name: keystore
            secret:
              secretName: keystore-zk-0
    - id: "1"
      nodeSelector:
        zkinstanceid: "1"
        updateOnChange:
        - kind: Secret
          name: keystore-zk-1
...

    - id: "2"
      nodeSelector:
        zkinstanceid: "2"  
        updateOnChange:
        - kind: Secret
          name: keystore-zk-2
...           
  kafka:
    app: kafkaapp
    image: jankul02/kubernetes-kafka:0.0.69
    autoscale:
      minReplicas: 3
      maxReplicas: 3
      initial: 3
      targetCPUUtilizationPercentage: 1
      resources:
        requests: 
          cpu: "200m"
        limits:
          cpu: "300m"
      behavior:
        scaleDown: 
          policies: 
          - type: Pods 
            value: 1
            periodSeconds: 60 

          stabilizationWindowSeconds: 100
        scaleUp: 
          policies:
          - type: Pods
            value: 1
            periodSeconds: 70
          stabilizationWindowSeconds: 100
    affinity: {}

    maxUnavailable: 1    
    updateOnChange: {}
    spec:
      env: 
        - name: KAFKA_LISTENERS
          value: "INTERNET://:9092,INTRANET://:9093"
        - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
          value: "INTERNET:PLAINTEXT,INTRANET:PLAINTEXT" 
        - name: KAFKA_SSL_KEYSTORE_LOCATION
          value: "/mnt/keystore.jks"
        - name: KAFKA_SSL_TRUSTSTORE_LOCATION
          value: "/mnt/truststore.jks"
        - name: KAFKA_INTER_BROKER_LISTENER_NAME
          value: INTRANET
        - name: KAFKA_LOG4J_ROOT_LOGLEVEL
          value: DEBUG
        - name: KAFKA_TOOLS_LOG4J_LOGLEVEL
          value: DEBUG
    instances:
    - id: "0"
      nodeSelector:
        kafkainstanceid: "0"
      updateOnChange:
      - kind: Secret
        name: keystore-kafka-0
      spec:
        env: 
          - name: KAFKA_SSL_KEYSTORE_PASSWORD
            valueFrom:
              secretKeyRef:
                name: keystorepasswd-kafka-1
                key: keystorepassword
          - name: KAFKA_SSL_KEY_PASSWORD
            valueFrom:
              secretKeyRef:
                name: keystorepasswd-kafka-1
                key: keystorepassword     
          - name: KAFKA_SSL_TRUSTSTORE_PASSWORD
            valueFrom:
              secretKeyRef:
                name: keystorepasswd-kafka-1
                key: truststorepassword                    
          - name: KAFKA_SSL_KEYSTORE_LOCATION
            value: "/mnt/keystore.jks"
          - name: KAFKA_SSL_TRUSTSTORE_LOCATION
            value: "/mnt/truststore.jks"
        volumeMounts:
          - name: keystore
            mountPath: /mnt
        volumes:
          - name: keystore
            secret:
              secretName: keystore-kafka-0
    - id: "1"
      nodeSelector:
        kafkainstanceid: "1"  
      updateOnChange:
      - kind: Secret
        name: keystore-kafka-1
  ...   
    - id: "2"
      nodeSelector:
        kafkainstanceid: "2"  
      updateOnChange:
      - kind: Secret
        name: certificate-kafka-2
    ....
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
for i in 0 1 2; do kubectl exec kafka-$i -- hostname -f; done
for i in 0 1 2; do kubectl exec zk-$i -- hostname -f; done

```

### functional case: write to zk-0 and get it back from zk-1

*if you configured TLS auth then you would  need the -zk-tls-config-file file *
```

kubectl exec zk-0 -it -- zookeeper-shell zk-0:2181 create /mykey "Hello World"
kubectl exec zk-1 -it -- zookeeper-shell zk-1:2181 get /mykey 

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




# uncordon
kubectl uncordon node-1231 

```

## Deployment and configuration

### Deployment

Change the image name in the zookeeperset resource file and then apply it and observe

```
kubectl apply -f config/samples/dataproxy_v1alpha1_zookeeperset.yaml

kubectl get pods -w # to observe the rollout 
```

### Instance Configuration
 
1. change Configuration of the zk-2 instance in the zookeeperset resource file

```
kubectl apply -f config/samples/dataproxy_v1alpha1_zookeeperset.yaml

kubectl get pods -w # to observe the restart zk-2 only 

# view the beginning of the log 
kubectl logs zk-2

```

# Helpful oneliners

### Build, push, deploy the operator 
```
make docker-build docker-push deploy VERSION=0.0.390
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
  zookeeper:
    spec:
        env:
          - name: WELCOME_MESSAGE
            value: "The generic Hello World from zookeeper"      
          - name: ZOOKEEPER_TOOLS_LOG4J_LOGLEVEL
            value: DEBUG
          - name: ZOOKEEPER_LOG4J_ROOT_LOGLEVEL
            value: DEBUG
        volumes:
        ....
        volumeMounts:
        ....
...
  kafka:
      spec:
        env: 
          - name: WELCOME_MESSAGE
            value: "generic WELCOME"
          - name: KAFKA_LISTENERS
            value: "INTERNET://:9092,INTRANET://:9093"
        volumes:
        ....
        volumeMounts:
        ....
``` 

By conventions the instance specific configuration should be named ${ORD}-1 and specified under spec.data like

```yaml
  kafka:
...
  instances:
    - id: "0"
      nodeSelector: {}
      updateOnChange: {}
      spec:
        env: 
          - name: WELCOME_MESSAGE
            value: "HW from kafka 0"
          - name: KAFKA_SSL_KEYSTORE_PASSWORD
        volumes:
        ....
        volumeMounts:
        ....

    - id: "1"
      nodeSelector: {}
      updateOnChange: {}
      spec:
        env: 
          - name: WELCOME_MESSAGE
            value: "HW from kafka 1"
          - name: KAFKA_SSL_KEYSTORE_PASSWORD
            valueFrom:
              secretKeyRef:
                name: keystorepasswd-kafka-1
                key: keystorepassword
          - name: KAFKA_SSL_KEY_PASSWORD
            valueFrom:
              secretKeyRef:
                name: keystorepasswd-kafka-1
                key: keystorepassword     
        volumes:
        ....
        volumeMounts:
        ....
 
```
The configuration will be pached into the instances (pods) at their start up.
The patches can be seen in podpresets:
```
kubectl get pod zk-0 -o json
```

### deploying test client

1. Deploy a zookeeper client pod with configuration:
```
    apiVersion: v1
    kind: Pod
    metadata:
      name: zookeeper-client
      namespace: default
    spec:
      containers:
      - name: zookeeper-client
        image: confluentinc/cp-zookeeper:6.1.0
        command:
          - sh
          - -c
          - "exec tail -f /dev/null"
```
2. Log into the Pod

```
  kubectl exec -it zookeeper-client -- /bin/bash
```
3. Use zookeeper-shell to connect in the zookeeper-client Pod:
```
  zookeeper-shell zk-hs:2181
```
4. Explore with zookeeper commands, for example:

```
  # Gives the list of active brokers
  ls /brokers/ids

  # Gives the list of topics
  ls /brokers/topics

  # Gives more detailed information of the broker id '0'
  get /brokers/ids/0## ------------------------------------------------------
```

## Kafka
## ------------------------------------------------------
To connect from a client pod:

1. Deploy a kafka client pod with configuration:
```
    apiVersion: v1
    kind: Pod
    metadata:
      name: kafka-client
      namespace: default
    spec:
      containers:
      - name: kafka-client
        image: confluentinc/cp-enterprise-kafka:6.1.0
        command:
          - sh
          - -c
          - "exec tail -f /dev/null"
```
2. Log into the Pod
```
  kubectl exec -it kafka-client -- /bin/bash
```
3. Explore with kafka commands:
```
  # Create the topic
  kafka-topics --zookeeper cp-helm-charts-1632213534-cp-zookeeper-headless:2181 --topic cp-helm-charts-1632213534-topic --create --partitions 1 --replication-factor 1 --if-not-exists

  # Create a message
  MESSAGE="`date -u`"

  # Produce a test message to the topic
  echo "$MESSAGE" | kafka-console-producer --broker-list cp-helm-charts-1632213534-cp-kafka-headless:9092 --topic cp-helm-charts-1632213534-topic

  # Consume a test message from the topic
  kafka-console-consumer --bootstrap-server cp-helm-charts-1632213534-cp-kafka-headless:9092 --topic cp-helm-charts-1632213534-topic --from-beginning --timeout-ms 2000 --max-messages 1 | grep "$MESSAGE"
```


Cert Manager setup

```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml

```



Weavescope setup

```
kubectl apply -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"
kubectl port-forward -n weave "$(kubectl get -n weave pod --selector=weave-scope-component=app -o jsonpath='{.items..metadata.name}')" 4040

```

setting 'kafka' as the default namespace
```
kubectl config set-context --current --namespace=kafka
```

Kubernetes-Zookeeper
```
cd docker
make build push 
```

Kubernetes-Kafka
```
cd docker
make build push 
```

PodPresets
```
make undeploy docker-build docker-push  deploy

```

keys
```
cd ~/projects/verifyk8/generatingstores/kafka
 akafka-generate-ssl-automatic.sh
kubectl apply -f keystores.base64.secret.yaml
cd ~/projects/verifyk8/generatingstores/kafka
./zookeeper.sh 
kubectl apply -f keystores.base64.secret.yaml

```

zookeeperset
```
make undeploy
make docker-build docker-push deploy
kubectl apply -f config/samples/dataproxy_v1alpha1_msts_zookeeperset.yaml

```

```
kubectl logs -n zookeeperset-system  zookeeperset-controller-manager-979878fb8-pzl65 manager
```
