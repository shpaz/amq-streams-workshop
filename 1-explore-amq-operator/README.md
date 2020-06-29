# Exercise 1 - Exploring the AMQ Operator

## Table of Contents

- [Objective](#objective)
- [Diagram](#diagram)
- [Guide](#guide)
- [Takeaways](#takeaways)

# Objective

Explore and understand the AMQ operator, In this exercise we will learn to: 
- Deploy the AMQ Operator using the OperatorHub
- Create our own Kafka cluster using `Kafka` CR 
- Getting to know the different installation components 

# Diagram

![Red Hat Ansible Automation Lab Diagram](../../../images/network_diagram.png)

Make sure you connect to the cluster before starting this exercise! 

# Guide

## Step 1

Create a projet named `amq-streams` for the purpose of this exercise: 

```bash 
$ oc new-project amq-streams
```

## Step 2

Navigate to the Openshift console, and get into the `OperatorHub` tab, where you will find various operators to install, and install the 
AMQ operator in the created `amq-streams` project.  

## Step 3 

Verify that your Operator was installed properly, by using the `get pods` command: 

```bash 
$ oc get pods
 
NAME                                                   READY   STATUS    RESTARTS   AGE
amq-streams-cluster-operator-v1.5.0-56d47bfcd5-fg7fg   1/1     Running   0          71s
```

After we have our operator running, we can now create our Kafka cluster using the `Kafka` CR. For now, we'll deploy an ephemeral Kafka cluster where both Zookeeper and Kafka use 
emptyDir as their data source. 

## Step 4 

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

## Step 5 

Veirfy that your Kafka cluster installation had been successful by using the `get pods` command: 

```bash 
$ oc get pods                                                    
 
NAME                                                   READY   STATUS    RESTARTS   AGE
amq-streams-cluster-operator-v1.5.0-56d47bfcd5-fg7fg   1/1     Running   0          9m20s
my-cluster-entity-operator-847ff5cb57-rgdxw            3/3     Running   0          60s
my-cluster-kafka-0                                     2/2     Running   0          2m10s
my-cluster-kafka-1                                     2/2     Running   0          84s
my-cluster-kafka-2                                     2/2     Running   0          2m10s
my-cluster-zookeeper-0                                 1/1     Running   0          3m9s
my-cluster-zookeeper-1                                 1/1     Running   0          3m51s
my-cluster-zookeeper-2                                 1/1     Running   0          3m51s
```

Here you can see that you have deployed threee things:

* Zookeeper pods - Used for holding the metadata for your Kafka cluster such as topics, users, etc.
* Kafka pods - Used for holding the logDirs, which is the actual data (sequential messages). 
* Entity Operator - Used for interacting with the Kafka cluster with things related to users, topics, etc.

## Step 6 

Ensure that the Kafka nodes are indeed using an emptyDir volumes for storing the Kafka logDirs: 

```bash 
$ oc get pods/my-cluster-kafka-0 --output jsonpath='{.spec.volumes[0]}'
 
map[emptyDir:map[] name:data]
```

## Step 7 

Having our Kafka cluster using emptyDirs means that on failure that Kafka cluster will have to replicate data on his own because the underlying volume is gone.
In this case we will count on Kafka's replication mechanism for replicating the data which can sometimes cause unwanted latency. 

To do so, we'll create a producer a Topic, a Producer and a Consumer that will send messages to one another. Producer --> Topic --> Consumer. 
We'll kill one of the Kafka nodes and see how it affects the offset being transfered between the producer and the consumer. 


## Step 8 

Let's create a Kafka topic using the `Topic` CR with the name `my-topic`: 

```
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

## Step 9 

Validate that the created Kafka topic was created successfuly by using `get kt` command: 

```bash 
$ oc get kt
                                                                                
NAME       PARTITIONS   REPLICATION FACTOR
my-topic   12           3
```

The Kafka topic was created with 12 parititions and replication factor of 3. 

## A moment of thinking  

* How will the paritions be divided across our Kafka nodes? How many parititions will every node get? 


## Step 10 

Let's create a Kafka user to interact with the created topic, move through the `KafkaUser` CR to verify that you understand how user management is handled in AMQ: 

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

## Step 11 

Verify that the user was indeed created using the `get ku` command: 

```bash 
$ oc get ku  
                                                                               
NAME      AUTHENTICATION   AUTHORIZATION
my-user   tls              simple
```

## Step 12

Now let's create a Kafka Producer that will write messages to our `my-topic` topic, and a consumer that will consume those messages: 

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

## Step 13 

Verify that both consumer and producer works as expected by browsing their logs, for example: 

```bash 
$ oc logs $(oc get pod -l app=hello-world-producer -o=jsonpath='{.items[0].metadata.name}') -f

2020-06-29 12:31:58 INFO  KafkaProducerExample:18 - Sending messages "Hello world - 1"
2020-06-29 12:31:59 INFO  KafkaProducerExample:18 - Sending messages "Hello world - 2"
2020-06-29 12:32:00 INFO  KafkaProducerExample:18 - Sending messages "Hello world - 3"
2020-06-29 12:32:01 INFO  KafkaProducerExample:18 - Sending messages "Hello world - 4"
```

```bash 
$ oc logs $(oc get pod -l app=hello-world-consumer -o=jsonpath='{.items[0].metadata.name}') -f

2020-06-29 12:31:57 INFO  KafkaConsumerExample:27 - 	value: Hello world - 0
2020-06-29 12:31:58 INFO  KafkaConsumerExample:24 - Received message:
2020-06-29 12:31:58 INFO  KafkaConsumerExample:25 - 	partition: 10
2020-06-29 12:31:58 INFO  KafkaConsumerExample:26 - 	offset: 0
2020-06-29 12:31:58 INFO  KafkaConsumerExample:27 - 	value: Hello world - 1
2020-06-29 12:31:59 INFO  KafkaConsumerExample:24 - Received message:
2020-06-29 12:31:59 INFO  KafkaConsumerExample:25 - 	partition: 2
2020-06-29 12:31:59 INFO  KafkaConsumerExample:26 - 	offset: 0
2020-06-29 12:31:59 INFO  KafkaConsumerExample:27 - 	value: Hello world - 2
2020-06-29 12:32:00 INFO  KafkaConsumerExample:24 - Received message:
2020-06-29 12:32:00 INFO  KafkaConsumerExample:25 - 	partition: 8
2020-06-29 12:32:00 INFO  KafkaConsumerExample:26 - 	offset: 0
2020-06-29 12:32:00 INFO  KafkaConsumerExample:27 - 	value: Hello world - 3
```

## Step 14 

Now after that we have our producer and consumer running as expected, open another terminal and continue tailing the `Consumer` logs in the first terminal. Now we will delete one of the Kafka nodes
to see if we will have downtime in doing so: 

From the second terminal: 

```bash 
$ oc delete pod/my-cluster-kafka-1
```

Go back to you first teminal and verify that you have lost connection the the Kafka node: 

```bash 
466363 [main] INFO org.apache.kafka.clients.FetchSessionHandler - [Consumer clientId=consumer-1, groupId=my-group] Error sending fetch request (sessionId=1145780833, epoch=920) to node 1: org.apache.kafka.common.errors.DisconnectException.
```

The failure happened because we have accessed a node that had no healthy replica of the data we've asked for. Replicating the data when using large scale can be very painful, In the next exercise we will see how we can use OCS RBD to save the persistency of our data so that each Kafka node that is being deleted will get the same PV.

## Step 15 

Delete the exercise's resources: 

```bash 
$ oc delete deployment hello-world-consumer
$ oc delete deployment hello-world-producer 
$ oc delete ku my-user
$ oc delete kt my-topic
$ oc delete kafka my-cluster
```
# Complete

Congratulations! You have completed the first exercise :)

---
[Click Here to return to the AMQ streams Workshop](../README.md)
