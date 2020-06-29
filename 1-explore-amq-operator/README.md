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




# Complete

You have completed lab exercise 1

---
[Click Here to return to the AMQ streams Workshop](../README.md)
