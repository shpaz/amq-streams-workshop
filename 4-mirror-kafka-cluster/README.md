  **Kafka's mirroring feature makes it possible to maintain a replica of an existing Kafka cluster**
  
  in this exercise, we are going to create 2 local kafka clusters on our openshift project - a source cluster and a target cluster.
 then, we will attach a producer to the source cluster, and a consumer to the target cluster,
 that way, we will be sure that if the consumer is able to read data from the Kafka cluster, the mirror maker is working correctly.
  
 architecture when using mirrormaker
  ![](https://cwiki.apache.org/confluence/download/attachments/27846330/mirror_maker.jpg?version=2&modificationDate=1336499633000&api=v2)

go to the openshift console and install the AMQ operator ***and the Jaeger operator on your namespace.


create the jaeger instance:
```bash
$ oc apply -f - <<EOF 
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: my-jaeger
spec:
  strategy: allInOne
  allInOne:
    image: jaegertracing/all-in-one:latest
    options:
      log-level: debug
  storage:
    type: memory
    options:
      memory:
        max-traces: 100000
  ingress:
    enabled: true
  agent:
    strategy: DaemonSet
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
EOF

```


create 2 kafka clusters, for convenience we'll call them:
source and target
```bash
oc create -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: source-cluster
spec:
  kafka:
    version: 2.4.0
    replicas: 3
    listeners:
      plain: {}
      tls: {}
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      log.message.format.version: "2.4"
    storage:
      type: ephemeral
  zookeeper:
    replicas: 3
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
---
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: target-cluster
spec:
  kafka:
    version: 2.4.0
    replicas: 3
    listeners:
      plain: {}
      tls: {}
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      log.message.format.version: "2.4"
    storage:
      type: ephemeral
  zookeeper:
    replicas: 3
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF
```

create 2 topics
both will be called my-topic, and each will run on another kafka cluster
```bash
oc create -f - <<EOF 
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaTopic
metadata:
  name: my-source-topic
  labels:
    strimzi.io/cluster: source-cluster
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: 7200000
    segment.bytes: 1073741824
---
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaTopic
metadata:
  name: my-target-topic
  labels:
    strimzi.io/cluster: target-cluster
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: 7200000
    segment.bytes: 1073741824
EOF
```

create 2 users (1 for each cluster)
the first is called `my-source-user` and the latter `my-target-user`
```bash
oc create -f - <<EOF 
 
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaUser
metadata:
  name: my-target-user
  labels:
    strimzi.io/cluster: target-cluster
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
      # Example consumer Acls for topic my-target-topic suing consumer group my-target-group
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
      # Example Producer Acls for topic my-target-topic
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
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaUser
metadata:
  name: my-source-user
  labels:
    strimzi.io/cluster: source-cluster
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
      # Example consumer Acls for topic my-source-topic suing consumer group my-group
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
EOF
```
create consumer in target cluster and producer in source cluster
```bash
oc create -f - <<EOF 
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
        image: strimzi/hello-world-producer:latest
        env:
          - name: CA_CRT
            valueFrom:
              secretKeyRef:
                name: source-cluster-cluster-ca-cert
                key: ca.crt
          - name: USER_CRT
            valueFrom:
              secretKeyRef:
                name: my-source-user
                key: user.crt
          - name: USER_KEY
            valueFrom:
              secretKeyRef:
                name: my-source-user
                key: user.key
          - name: BOOTSTRAP_SERVERS
            value: source-cluster-kafka-bootstrap:9093
          - name: TOPIC
            value: my-topic
          - name: DELAY_MS
            value: "100"
          - name: LOG_LEVEL
            value: "INFO"
          - name: MESSAGE_COUNT
            value: "5000"
          - name: JAEGER_SERVICE_NAME
            value: hello-world-producer
          - name: JAEGER_AGENT_HOST
            value: my-jaeger-agent
          - name: JAEGER_SAMPLER_TYPE
            value: const
          - name: JAEGER_SAMPLER_PARAM
            value: "1"
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
        image: strimzi/hello-world-consumer:latest
        env:
          - name: CA_CRT
            valueFrom:
              secretKeyRef:
                name: target-cluster-cluster-ca-cert 
                key: ca.crt
          - name: USER_CRT
            valueFrom:
              secretKeyRef:
                name: my-target-user
                key: user.crt
          - name: USER_KEY
            valueFrom:
              secretKeyRef:
                name: my-target-user
                key: user.key
          - name: BOOTSTRAP_SERVERS
            value: target-cluster-kafka-bootstrap:9093
          - name: TOPIC
            value: my-topic
          - name: GROUP_ID
            value: my-group
          - name: LOG_LEVEL
            value: "INFO"
          - name: MESSAGE_COUNT
            value: "1000"
          - name: JAEGER_SERVICE_NAME
            value: hello-world-consumer
          - name: JAEGER_AGENT_HOST
            value: my-jaeger-agent
          - name: JAEGER_SAMPLER_TYPE
            value: const
          - name: JAEGER_SAMPLER_PARAM
            value: "1"
EOF
```
note the Jaeger Env vars, which are used by Jaeger tracing.

check producer logs to see its producing

`oc logs $(oc get pod -l app=hello-world-producer -o=jsonpath='{.items[0].metadata.name}') -f`

apply mirror maker between both cluster

```bash
oc create -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaMirrorMaker
metadata:
  name: my-mirror-maker
spec:
  replicas: 1
  consumer:
    authentication:
      certificateAndKey:
        certificate: user.crt
        key: user.key
        secretName: my-source-user
      type: tls
    bootstrapServers: source-cluster-kafka-bootstrap:9093
    tls:
      trustedCertificates:
      - certificate: ca.crt
        secretName: "source-cluster-cluster-ca-cert"
    groupId: my-group
  producer:
    authentication:
      certificateAndKey:
        certificate: user.crt
        key: user.key
        secretName: my-target-user
      type: tls
    bootstrapServers: target-cluster-kafka-bootstrap:9093
    tls:
      trustedCertificates:
      - certificate: ca.crt
        secretName: "target-cluster-cluster-ca-cert"
  whitelist: ".*"
  tracing:
    type: jaeger
  template:
    mirrorMakerContainer:
      env:
        - name: JAEGER_SERVICE_NAME
          value: my-mirror-maker
        - name: JAEGER_AGENT_HOST
          value: my-jaeger-agent
        - name: JAEGER_SAMPLER_TYPE
          value: const
        - name: JAEGER_SAMPLER_PARAM
          value: "1"
EOF

```


check consumer logs to see he reads from the target cluster, which was written from the source cluster by mirror maker

`oc logs $(oc get pod -l app=hello-world-consumer -o=jsonpath='{.items[0].metadata.name}') -f`


we can see the traces in jaeger ui:

`oc get routes`

will show us the route to Jaeger ui.
which can give us a better grasp regarding the "path" the data is passing.


lets clean our enviornment: 

```bash
oc delete kafka --all
oc delete ku --all
oc delete kt --all
oc delete deployment hello-world-consumer hello-world-producer my-jaeger my-mirror-maker-mirror-maker 
```