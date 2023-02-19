# kafka-quarkus-keda-scaler

Sample project that builds a simple `QuarkusConsumer`, based on [Quarkus](https://quarkus.io/).

## Architecture Overview

```bash
                                                         Kubernetes Cluster
┌──────────────────────────┐          ┌───────────────────────────────────────────────────────────┐
│                          │          │                                                           │
│  Apache Kafka cluster    │          │                                                           │
│                          │          │                                                           │
│                          │          │                                   Consumer Group          │
│   ┌──────────────────┐   │          │                                 ┌─────────────────┐       │
│   │ Topic            │   │          │                                 │                 │       │
│   │                  │   │          │  Pull Messages                  │ ┌─────────────┐ │       │
│   │ ┌──────────────┐ ◄───┼──────────┼─────────────────────────────────┤ │Consumer-1   │ │       │
│   │ │Partition-1   │ │   │          │                                 │ │             │ │       │
│   │ └──────────────┘ │   │          │                                 │ └─────────────┘ │       │
│   │                  │   │          │                                 │                 │       │
│   │      ..          │   │          │                                 │     ..          │       │
│   │                  │   │          │                                 │                 │       │
│   │      ..          │   │          │                                 │     ..          │       │
│   │                  │   │          │                                 │                 │       │
│   │      ..          │   │          │                                 │     ...         │       │
│   │ ┌──────────────┐ │   │          │                                 │                 │       │
│   │ │Partition-n   │ │   │          ├─────────────────┐               │  ┌────────────┐ │       │
│   │ └──────────────┘ │   │          │                 │               │  │Consumer-n  │ │       │
│   │                  │   │          │                 │               │  │            │ │       │
│   │                  │   │          │ KEDA*       ┌───┼─────────────► │  └────────────┘ │       │
│   │                  │ ◄─┼──────────┼──┐          │   │               │                 │       │
│   └──────────────────┘   │          │  │          │   │               └─────────────────┘       │
│                          │          │                 │                                         │
└──────────────────────────┘          └─────────────────┴─────────────────────────────────────────┘
   *KEDA scales the consumers of the consumer-gourp, based on records on the topic and its partitions.

```

## Apache Kafka topics

The consumers are getting messages from a topic like:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 10
  replicas: 3
  config:
    retention.bytes: 53687091200
    retention.ms: 36000000
```

We now use [KEDA](./keda-scalers.yaml) to dynamically scale our consumers, based on the consumer group for the topic.

The application is defined as a normal Kubernetes [deployment](./k8s.yaml).

## The load

For generating some load, a batch like below can be used:

* First create a kafka topic `my-topic` with > 1 partitions.

* Then use:

```
$ kubectl apply -f generate-load.yaml
```

## Scaled consumers

After the initial count of `one` replica from the deployment, once the above load kicks in, KEDA dynamically scales the app to `10` replicas, see:

```
k get pods           
NAME                                     READY   STATUS    RESTARTS   AGE
kafka-quarkus-consumer-fbbdb4c57-6d77c   1/1     Running   0          50s
kafka-quarkus-consumer-fbbdb4c57-bfjxj   1/1     Running   0          50s
kafka-quarkus-consumer-fbbdb4c57-bp6th   1/1     Running   0          20s
kafka-quarkus-consumer-fbbdb4c57-ccb6q   1/1     Running   0          20s
kafka-quarkus-consumer-fbbdb4c57-g7pr5   1/1     Running   0          35s
kafka-quarkus-consumer-fbbdb4c57-j6289   1/1     Running   0          35s
kafka-quarkus-consumer-fbbdb4c57-ll789   1/1     Running   2          10m
kafka-quarkus-consumer-fbbdb4c57-qrp2l   1/1     Running   0          50s
kafka-quarkus-consumer-fbbdb4c57-s6w7f   1/1     Running   0          35s
kafka-quarkus-consumer-fbbdb4c57-scljw   1/1     Running   0          35s
```

**NOTE:** This is a simple POC/demo, and nothing special for rebalancing was done!
