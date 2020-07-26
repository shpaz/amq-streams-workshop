## Monitor your Kafka cluster using prometheus & Grafana

We all know the importance of Metrics in Production, its ability to help the Operation teams catch onto a problem before it occurs, alert them, and as a result improve out production uptime.  

Metrics in Red Hat AMQ can help us understand what is the state of Kafka cluster.
In this exercise, we will use AMQ Streams operator to deploy Kafka on Openshift, and expose its metrics to Prometheus and Grafana.

in Order to Expose metrics using our custom CR. we will need to add the fields - `spec.kafka.metrics` , `spec.zookeeper.metrics` and `spec.kafkaExporter` :

## Step 1 

```bash
$ oc create -f - <<EOF  
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
      type: ephemeral  
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
      size: 100Gi  
      deleteClaim: false  
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
EOF
```

Please note, you can also export the basic metrics for zookeeper and kafka using `metrics: {}` , but we’ve added some more metrics presented in [strimzi github project](https://github.com/strimzi/strimzi-kafka-operator/blob/master/examples/metrics/kafka-metrics.yaml)

## Step 2

Let's take a look on our runing pods in the project:

```bash
$ oc get pods  

amq-streams-cluster-operator-v1.5.0-986f4d669-s286z   1/1     Running   0          18m  
my-cluster-entity-operator-5fc6c4bf49-hw5mm           3/3     Running   0          78s  
my-cluster-kafka-0                                    2/2     Running   0          2m2s  
my-cluster-kafka-1                                    2/2     Running   0          104s  
my-cluster-kafka-2                                    2/2     Running   0          104s  
my-cluster-kafka-exporter-75c4c67757-v4zb2            1/1     Running   0          54s  
my-cluster-zookeeper-0                                1/1     Running   0          2m29s  
my-cluster-zookeeper-1                                1/1     Running   0          2m29s  
my-cluster-zookeeper-2                                1/1     Running   0          2m29s
```
-   We have our `amq-streams-cluster-oprator` which is the amq-streams operator
-   We have 3 Kafka pod and 3 Zookeeper pods as stated in `spec.kafka.replicas` and `spec.zookeeper.replicas` accordingly
-   we also have an entity operator, which comprises of the topic operator and the User operator
-   Lastly, we have our kafka exporter which exports our cluster metrics as stated in `spec.kafkaExporter` , the operator creates a service to reach this pod

## Step 3 

**Open the prometheus.yaml file, and edit the namespace according to your project in line 50**

you can also use sed on CHANGE_ME in order to achieve this (We need Prometheus to access the Kubernetes API, and to do so we should create the needed  Role).

Now, lets run `Prometheus` in our cluster:
```bash
$ oc apply -f prometheus.yaml
```

## Step 4 

Make sure the needed Prometheus resources were created, try to access your Prometheus route.

## Step 5 

Create you `Grafana` deployment:

```bash
oc apply -f grafana.yaml
```

## Step 6 

TRy and access you `Grafana` route, do you have any data? why? 

## Step 7

After we have our monitoring system up and running, we can now create some workload on our `Kafka` cluster, to see if we get the data to our `Grafana` dasboards:

```bash
oc create -f - <<EOF

apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaTopic
metadata:
  name: my-topic
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: 7200000
    segment.bytes: 1073741824
---
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaUser
metadata:
  name: my-user
  labels:
    strimzi.io/cluster: my-cluster
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
      # Example consumer Acls for topic my-topic suing consumer group my-group
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operation: Read
        host: "*"
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operation: Describe
        host: "*"
      - resource:
          type: group
          name: my-group
          patternType: literal
        operation: Read
        host: "*"
      # Example Producer Acls for topic my-topic
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operation: Write
        host: "*"
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operation: Create
        host: "*"
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operation: Describe
        host: "*"
---
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
            value: "100"
          - name: LOG_LEVEL
            value: "INFO"
          - name: MESSAGE_COUNT
            value: "5000"
---
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
            value: "1000"
EOF
```
## Step 8 

Verify that you consumer and producer are working as expected, by printing their logs. 

## Step 9 

Access your `Grafana` dashboards and see if you get the data regarding your `Kafka` cluster's state. 

## Step 10 

Play with the number of your consumers and producers to see how the data changes in your dashboards.

# Complete 
