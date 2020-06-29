# Exercise 2 - Create a resilient Kafka cluster using OCS RBD and CruiseControl

## Table of Contents

- [Objective](#objective)
- [Diagram](#diagram)
- [Guide](#guide)
- [Takeaways](#takeaways)

# Objective

Getting to know better with patition allocation and logDirs persistency:
- Deploy a Kafka cluster using OCS RBD as block backend for Kafka logDirs
- Understand parition allocation across Kafka nodes 
- Use CruiseControl to get automatic rebalancing of paritions  

# Diagram

![Red Hat Ansible Automation Lab Diagram](../../../images/network_diagram.png)

Make sure you connect to the cluster before starting this exercise! 

# Guide

## Step 1

Create a Kafka cluster using the `Kafka` CR: 

```bash 
$ oc create -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: my-cluster
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
      type: persistent-claim
      size: 20Gi
      deleteClaim: true
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      deleteClaim: true
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF
```

Pay attention! now under *Storage* section you could see that the `Kafka` CR is asking for a 20GB PV for the Kafka nodes and 10GB PV for the Zookeeper nodes. that means that the data that is being saved by Kafka is no longer ephemeral and is persistent in OCS RBD which is the default storage class. 

## Step 2 

Verify that all the pods were successfuly created and that you have now PVCs allocated from the OCS RBD storage class: 

```bash 
$ oc get pods   
                                                                                                 
NAME                                                   READY   STATUS    RESTARTS   AGE
amq-streams-cluster-operator-v1.5.0-56d47bfcd5-nhfmg   1/1     Running   0          2m15s
my-cluster-entity-operator-847ff5cb57-4djk7            3/3     Running   0          35s
my-cluster-kafka-0                                     2/2     Running   1          80s
my-cluster-kafka-1                                     2/2     Running   1          80s
my-cluster-kafka-2                                     2/2     Running   0          80s
my-cluster-zookeeper-0                                 1/1     Running   0          2m4s
my-cluster-zookeeper-1                                 1/1     Running   0          2m4s
my-cluster-zookeeper-2                                 1/1     Running   0          2m4s
```

```bash 
$ oc get pvc  
                                                                                                  
NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
data-my-cluster-kafka-0       Bound    pvc-14f4fbe5-21fc-4f84-a6de-acbbde9edf3a   20Gi       RWO            ocs-storagecluster-ceph-rbd   117s
data-my-cluster-kafka-1       Bound    pvc-8dda668a-dbce-485d-9ac2-6f1d6c6dbfaf   20Gi       RWO            ocs-storagecluster-ceph-rbd   117s
data-my-cluster-kafka-2       Bound    pvc-40092645-dc14-463f-bb67-95ac06e9b2e0   20Gi       RWO            ocs-storagecluster-ceph-rbd   117s
data-my-cluster-zookeeper-0   Bound    pvc-46c66a88-71dd-45cb-90ac-13a5e9cc7f3e   10Gi       RWO            ocs-storagecluster-ceph-rbd   2m41s
data-my-cluster-zookeeper-1   Bound    pvc-30d1ab25-f353-4778-b41e-a10f69dd4c55   10Gi       RWO            ocs-storagecluster-ceph-rbd   2m41s
data-my-cluster-zookeeper-2   Bound    pvc-b407c1a6-f01b-4987-86a5-bae7f0383e1b   10Gi       RWO            ocs-storagecluster-ceph-rbd   2m41s
```

## Step 3 

Create a Kafka topic named `my-topic`:

```bash
$ oc create -f - <<EOF 
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
EOF
```

## Step 4 

Create a Kafka user named `my-user`:

```bash
$ oc create -f - <<EOF 
 
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
EOF
```

## Step 5 

Create a Producer and Consumer deployment for interacting with the created `my-topic` topic: 

```bash 
$ oc create -f - <<EOF 
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

## Step 6 

In you current terminal please tail the consumer logs by using: 

```bash 
$ oc logs $(oc get pod -l app=hello-world-consumer -o=jsonpath='{.items[0].metadata.name}') -f
```

Open a new terminal window, and delete one of the Kafka nodes by using: 

```bash 
$ oc delete pod/my-cluster-kafka-1
```

* Verify that there was no downtime while doing so
* Verify that the Kafka nodes still have the same PVs configured 

## Step 7 

Edit the Kafka CR with `oc edit` kafka my-cluster and add .spec.cruiseControl section set to {}. E.g.

```bash 
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
  zookeeper:
    # ...
  entityOperator:
    # ...
  cruiseControl: {}
```

## Step 8 

Take a look on the current parition distribution, pay attention to which parititions are being handled by which nodes: 

```bash 
kubectl exec -ti my-cluster-kafka-0 -c kafka -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe
```

## Step 9

Create a `KafkaRebalance` CR to start get a `KafkaProposal` on how the wizard thinks you should consider migrating your partitions:

```bash 
$ oc create -f - <<EOF

apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaRebalance
metadata:
  name: my-rebalance
  labels:
    strimzi.io/cluster: my-cluster
spec:
  goals:
    - RackAwareGoal
    - ReplicaCapacityGoal
    - DiskCapacityGoal
    - NetworkInboundCapacityGoal
    - NetworkOutboundCapacityGoal
    - CpuCapacityGoal
    - ReplicaDistributionGoal
    - DiskUsageDistributionGoal
    - NetworkInboundUsageDistributionGoal
    - NetworkOutboundUsageDistributionGoal
    - TopicReplicaDistributionGoal
    - LeaderReplicaDistributionGoal
    - LeaderBytesInDistributionGoal
EOF
```

## Step 10 

Now wait until `oc get kt` shows that the status is in `ProposalReady`: 

```bash 
$ oc describe kr
  
  Optimization Result:
    Data To Move MB:  0
    Excluded Brokers For Leadership:
    Excluded Brokers For Replica Move:
    Excluded Topics:
    Intra Broker Data To Move MB:         0
    Monitored Partitions Percentage:      100
    Num Intra Broker Replica Movements:   0
    Num Leader Movements:                 8
    Num Replica Movements:                11
    On Demand Balancedness Score After:   89.29167194382578
    On Demand Balancedness Score Before:  84.7221402891931
    Recent Windows:                       1
  Session Id:                             fe2b8f0b-bcb3-43c8-8c23-a9af6296dc2e
Events:                                   <none>
```

Notice that the balancing process has it's own score, so appying this proposal will improve us with 5 points to our balancing score, Let's approve the proposal:

```bash 
$ oc annotate kafkarebalance my-rebalance strimzi.io/rebalance=approve
```

Verify by using `oc describe kr` that the proposal status is in `Rebalancing` state, and then check the status of the topics again and compare the distribution:

```bash 
$ oc exec -ti my-cluster-kafka-0 -c kafka -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe
```
## Step 11

Delete the exercise's resources: 

```bash 
$ oc delete deployment hello-world-consumer
$ oc delete deployment hello-world-producer 
$ oc delete kr my-rebalance
$ oc delete ku my-user
$ oc delete kt my-topic
$ oc delete kafka my-cluster

*Note*: PVs will be deleted when the Kafka cluster is destroyed 
```
# Complete

Congratulations! You have completed the first exercise :)

---
[Click Here to return to the AMQ streams Workshop](../README.md)
