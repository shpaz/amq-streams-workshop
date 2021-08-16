## Monitor your Kafka cluster using prometheus & Grafana

We all know the importance of Metrics in Production, its ability to help the Operation teams catch onto a problem before it occurs, alert them, and as a result improve out production uptime.  

Metrics in Red Hat AMQ can help us understand what is the state of Kafka cluster.
In this exercise, we will use AMQ Streams operator to deploy Kafka on Openshift, and expose its metrics to Prometheus and Grafana.

in Order to Expose metrics using our custom CR. we will need to add the fields - `spec.kafka.metrics` , `spec.zookeeper.metrics` and `spec.kafkaExporter` :

## Table of Contents

- [Objective](#objective)
- [Diagram](#diagram)
- [Guide](#guide)
- [Takeaways](#takeaways)

# Objective

Getting to know better with our AMQ monitoring possibilities: 

- Deploy `Promehtues` and `Grafana` to monitor our cluster 
- Understand how we can view metrics to understand better our cluster's state and performance   

# Diagram

![](https://478h5m1yrfsa3bbe262u7muv-wpengine.netdna-ssl.com/wp-content/uploads/2018/08/kubernetes_prom_diagram2.png)

# Guide

## Step 1 

Make sure you navigate to `Add+ --> YAML` and copy the following YAML into the code snipped presented in the browser:

```bash
apiVersion: kafka.strimzi.io/v1alpha1  
kind: Kafka  
metadata:  
  name: my-cluster  
  labels:  
    app: my-cluster  
spec:  
  kafka:  
    replicas: 3  
    listeners:  
      plain: {}  
      tls: {}  
    config:  
      offsets.topic.replication.factor: 3  
      transaction.state.log.replication.factor: 3  
      transaction.state.log.min.isr: 2  
    storage:  
      type: persistent-claim  
      size: 5Gi  
      deleteClaim: true  
    metrics:  
      lowercaseOutputName: true  
      rules:  
      - pattern : kafka.server<type=ReplicaManager, name=(.+)><>(Value|OneMinuteRate)  
        name: "kafka_server_replicamanager_$1"  
      - pattern : kafka.controller<type=KafkaController, name=(.+)><>Value  
        name: "kafka_controller_kafkacontroller_$1"  
      - pattern : kafka.server<type=BrokerTopicMetrics, name=(.+)><>OneMinuteRate  
        name: "kafka_server_brokertopicmetrics_$1"  
      - pattern : kafka.network<type=RequestMetrics, name=RequestsPerSec, request=(.+)><>OneMinuteRate  
        name: "kafka_network_requestmetrics_requestspersec_$1"  
      - pattern : kafka.network<type=SocketServer, name=NetworkProcessorAvgIdlePercent><>Value  
        name: "kafka_network_socketserver_networkprocessoravgidlepercent"  
      - pattern : kafka.server<type=ReplicaFetcherManager, name=MaxLag, clientId=(.+)><>Value  
        name: "kafka_server_replicafetchermanager_maxlag_$1"  
      - pattern : kafka.server<type=KafkaRequestHandlerPool, name=RequestHandlerAvgIdlePercent><>OneMinuteRate  
        name: "kafka_kafkarequesthandlerpool_requesthandleravgidlepercent"  
      - pattern : kafka.controller<type=ControllerStats, name=(.+)><>OneMinuteRate  
        name: "kafka_controller_controllerstats_$1"  
      - pattern : kafka.server<type=SessionExpireListener, name=(.+)><>OneMinuteRate  
        name: "kafka_server_sessionexpirelistener_$1"  
  zookeeper:  
    replicas: 3  
    readinessProbe:  
      initialDelaySeconds: 15  
      timeoutSeconds: 5  
    livenessProbe:  
      initialDelaySeconds: 15  
      timeoutSeconds: 5  
    storage:  
      type: persistent-claim  
      size: 5Gi  
      deleteClaim: true  
    metrics:  
      lowercaseOutputName: true  
      rules:  
      - pattern: "org.apache.ZooKeeperService<name0=ReplicatedServer_id(\\d+)><>(\\w+)"  
        name: "zookeeper_$2"  
      - pattern: "org.apache.ZooKeeperService<name0=ReplicatedServer_id(\\d+), name1=replica.(\\d+)><>(\\w+)"  
        name: "zookeeper_$3"  
        labels:  
          replicaId: "$2"  
      - pattern: "org.apache.ZooKeeperService<name0=ReplicatedServer_id(\\d+), name1=replica.(\\d+), name2=(\\w+)><>(\\w+)"  
        name: "zookeeper_$4"  
        labels:  
          replicaId: "$2"  
          memberType: "$3"  
      - pattern: "org.apache.ZooKeeperService<name0=ReplicatedServer_id(\\d+), name1=replica.(\\d+), name2=(\\w+), name3=(\\w+)><>(\\w+)"  
        name: "zookeeper_$4_$5"  
        labels:  
          replicaId: "$2"  
          memberType: "$3"  
      - pattern: "org.apache.ZooKeeperService<name0=StandaloneServer_port(\\d+)><>(\\w+)"  
        name: "zookeeper_$2"  
      - pattern: "org.apache.ZooKeeperService<name0=StandaloneServer_port(\\d+), name1=(InMemoryDataTree)><>(\\w+)"  
        name: "zookeeper_$2_$3"  
  entityOperator:  
    topicOperator: {}  
    userOperator: {}  
  kafkaExporter:  
    topicRegex: ".*"  
    groupRegex: ".*" 
```

*Question: What Does the `deleteClaim` mean? How it is connected to our previous exercise?*

Please note, you can also export the basic metrics for zookeeper and kafka using `metrics: {}` , but we’ve added some more metrics presented in [strimzi github project](https://github.com/strimzi/strimzi-kafka-operator/blob/master/examples/metrics/kafka-metrics.yaml)

## Step 2 

Make sure your Kafka cluster was successfully installed and that you can see all of its compnents: 

![](../1-explore-amq-operator/pictures/my-kafka-cluster.png)


## Step 3 

Veirfy that your Kafka cluster installation had been successful by using the `Project -> Pods` in the inventory: 

![](../1-explore-amq-operator/pictures/get-pods.png)

-   We have our `amq-streams-cluster-oprator` which is the amq-streams operator
-   We have 3 Kafka pod and 3 Zookeeper pods as stated in `spec.kafka.replicas` and `spec.zookeeper.replicas` accordingly
-   we also have an entity operator, which comprises of the topic operator and the User operator
-   Lastly, we have our kafka exporter which exports our cluster metrics as stated in `spec.kafkaExporter` , the operator creates a service to reach this pod

## Step 4

Ensure that the Kafka nodes are indeed using an persistent volumes for storing the Kafka logDirs by using `Project -> PVCs` on the left tab: 

![](../1-explore-amq-operator/pictures/persistent-pvcs.png)

## Step 5 

Make sure that you have running instances of `Prometheus` and `Grafana` and that both pods are in running state. 

## Step 6 

Try and access you `Grafana` route (`user: admin`, `password: 123456`), do you have any data? why? 

## Step 7 

Let's create a Kafka topic using the `Add+ -> Operator Backed -> Kafka Topic -> Create` with the name `my-topic`: 

![](../1-explore-amq-operator/pictures/create-topic.png)


Make sure you leave the default values and hit the `Create` button. 

## Step 8 

Validate that the created Kafka topic was created successfuly by using `get kt` command: 

```bash 
$ oc get kt
                                                                                
NAME       PARTITIONS   REPLICATION FACTOR
my-topic   12           3
```

The Kafka topic was created with 12 parititions and replication factor of 3. 

## Step 9 

Let's create a Kafka user to interact with the created topic, move through the `KafkaUser` CR to verify that you understand how user management is handled in AMQ.

Copy this YAML, and paste it in `Add+ -> Operator Backed -> Kafka User` in order to create the `Kafka User` CR: 
 

![](../1-explore-amq-operator/pictures/create-user.png)

Before you hit the `Create` button, switch to the `YAML View` section to verify you understand all the ACLs that is being given to our created user.

## Step 10

Now let's create a Kafka Producer that will write messages to our `my-topic` topic, and a consumer that will consume those messages via `Add+ -> YAML` 

```bash 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-world-producer
  name: hello-world-producer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world-producer
  template:
    metadata:
      labels:
        app: hello-world-producer
    spec:
      containers:
      - name: hello-world-producer
        image: strimzici/hello-world-producer:support-training
        env:
          - name: CA_CRT
            valueFrom:
              secretKeyRef:
                name: my-cluster-cluster-ca-cert
                key: ca.crt
          - name: USER_CRT
            valueFrom:
              secretKeyRef:
                name: my-user
                key: user.crt
          - name: USER_KEY
            valueFrom:
              secretKeyRef:
                name: my-user
                key: user.key
          - name: BOOTSTRAP_SERVERS
            value: my-cluster-kafka-bootstrap:9093
          - name: TOPIC
            value: my-topic
          - name: DELAY_MS
            value: "5000"
          - name: LOG_LEVEL
            value: "INFO"
          - name: MESSAGE_COUNT
            value: "5000"
```
Now let's create a Kafka consumer that will read messages from our `my-topic` topic via `Add+ -> YAML` :

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-world-consumer
  name: hello-world-consumer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world-consumer
  template:
    metadata:
      labels:
        app: hello-world-consumer
    spec:
      containers:
      - name: hello-world-consumer
        image: strimzici/hello-world-consumer:support-training
        env:
          - name: CA_CRT
            valueFrom:
              secretKeyRef:
                name: my-cluster-cluster-ca-cert
                key: ca.crt
          - name: USER_CRT
            valueFrom:
              secretKeyRef:
                name: my-user
                key: user.crt
          - name: USER_KEY
            valueFrom:
              secretKeyRef:
                name: my-user
                key: user.key
          - name: BOOTSTRAP_SERVERS
            value: my-cluster-kafka-bootstrap:9093
          - name: TOPIC
            value: my-topic
          - name: GROUP_ID
            value: my-group
          - name: LOG_LEVEL
            value: "INFO"
          - name: MESSAGE_COUNT
            value: "5000"
```
## Step 11 

Verify that you consumer and producer are working as expected, by printing their logs. 

## Step 12

Now Let's deploy `Kadrop` which is a simple UI for Kafka clusters. We'll see how we can browse our entire Kafka configuration, Monitor our cluster and even view the meesages landing in our created Topic. 

In order to do so, We'll have to create three components, The first one is the deployment itself for the `Kadrop` pod:

```bash
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: kafdrop
spec:
  selector:
    app: kafdrop
  replicas: 1
  template:
    metadata:
      labels:
        app: kafdrop
    spec:
      containers:
        - name: kafdrop
          image: obsidiandynamics/kafdrop:latest
          ports:
            - containerPort: 9000
          env:
          - name: KAFKA_BROKERCONNECT
            value: "my-cluster-kafka-bootstrap:9092"
          - name: JVM_OPTS
            value: "-Xms32M -Xmx64M"
          - name: SERVER_SERVLET_CONTEXTPATH
            value: "/"
```

Now that we have the `Pod` running, We'll deploy a `Service` that will help up interact within the cluster itself: 

```bash
apiVersion: v1
kind: Service
metadata:
  name: kafdrop
spec:
  selector:
    app: kafdrop
  ports:
    - protocol: TCP
      port: 9000
      targetPort: 9000
```

Now, We'll create a `Route` so we could access the `Kadrop` UI outside of the Openshift cluster:

```bash
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: kafrop
spec:
  subdomain: ''
  to:
    kind: Service
    name: kafdrop
    weight: 100
  port:
    targetPort: 9000
  wildcardPolicy: None
```

## Step 13 

Make sure you can Access the `Kadrop` UI by pressing the arrow on the right when looking at the `Topology` view: 

![](../1-explore-amq-operator/pictures/kadrop.png)


## Step 14

Play with the number of your consumers and producers to see how the data changes in your `Kadrop` dashboards.

# Complete

Congratulations! You have completed the second exercise :)

---
[Click Here to return to the AMQ streams Workshop](../README.md)
