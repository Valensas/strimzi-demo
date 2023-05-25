# Strimzi demo

This repository is a demo of running a Kafka cluster with Strimzi in Kubernetes, showcasing some of its features.

The file `00-full.yaml` contains the final version of the manifests (except for the final rebalance topic).

## Prerequisites
 - Access to a Kubernetes cluster
 - Dynamic volume provisioning
 - Access to LoadBalancer services
 - [Kubernetes Ingress Nginx Controller](https://kubernetes.github.io/ingress-nginx/) with [SSL passtrough enabled](https://kubernetes.github.io/ingress-nginx/user-guide/tls/#ssl-passthrough)
 - [Cert Manager](https://cert-manager.io) with an issuer configured

## Strimzi Installation

Install Strimzi with the following commands:

```bash
helm repo add strimzi https://strimzi.io/charts/
helm install strimzi-operator strimzi/strimzi-kafka-operator --set watchAnyNamespace=true
```

## Demo 1: Simple deployment

This demo demonstrates the minimal configurations required to set up a Kafka cluster running in Kubernetes.

```
# Create the Kafka cluster
$ kubectl apply -f 01-simple.yaml
kafka.kafka.strimzi.io/demo created
# Wait until the cluster is ready
$ kubectl get kafka
NAME   DESIRED KAFKA REPLICAS   DESIRED ZK REPLICAS   READY   WARNINGS
demo   3                        3                     True
# Wait until all the topics are ready
$ kubectl get kafkatopics
NAME                                                                                               CLUSTER   PARTITIONS   REPLICATION FACTOR   READY
consumer-offsets---84e7a678d08f4bd226872e5cdd4eb527fadc1c6a                                        demo      50           3                    True
strimzi-store-topic---effb8e3e057afce1ecf67c3f5d8e4e3ff177fc55                                     demo      1            3                    True
strimzi-topic-operator-kstreams-topic-store-changelog---b75e702040b99be8a9263134de3507fc0cc4017b   demo      1            3                    True
test-topic-1                                                                                       demo      1000         3                    True
```

### Test the cluster

Start a consumer from one terminal:

```bash
kubectl exec -it demo-kafka-0 -- /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server demo-kafka-bootstrap:9092 --topic test-topic-1
```

and a producer from another terminal:

```bash
kubectl exec -it demo-kafka-0 -- bash -c "while true; do uuidgen; sleep 1; done | /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server demo-kafka-bootstrap:9092 --topic test-topic-1"
```

## Demo 2: Monitoring and Load balancer

This demo shows how to enable Prometheus monitoring for your Kafka cluster and set up external access trough LoadBalancer services.

```
# Enable a Kafka Exporter and a LoadBalancer listener
$ kubectl apply -f 02-monitoring-lb.yaml
kafka.kafka.strimzi.io/demo configured
# Verify LoadBalancer services are created
$ kubectl get svc
NAME                               TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
demo-kafka-bootstrap               ClusterIP      10.43.197.147   <none>         9091/TCP,9092/TCP            17m
demo-kafka-brokers                 ClusterIP      None            <none>         9090/TCP,9091/TCP,9092/TCP   17m
demo-kafka-loadblancer-0           LoadBalancer   10.43.230.152   10.200.10.243   9093:31086/TCP               67s
demo-kafka-loadblancer-1           LoadBalancer   10.43.197.203   10.200.10.241   9093:30448/TCP               67s
demo-kafka-loadblancer-2           LoadBalancer   10.43.84.120    10.200.10.245   9093:30573/TCP               67s
demo-kafka-loadblancer-bootstrap   LoadBalancer   10.43.195.16    10.200.10.244   9093:31613/TCP               67s
demo-zookeeper-client              ClusterIP      10.43.42.61     <none>         2181/TCP                     18m
demo-zookeeper-nodes               ClusterIP      None            <none>         2181/TCP,2888/TCP,3888/TCP   18m
# Verify Kafka Exporter deployment is up and running
$ kubectl get deploy
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
demo-entity-operator   1/1     1            1           19m
demo-kafka-exporter    1/1     1            1           36s
# Optional: create a PodMonitor for Prometheus to scap the exporter
$ kubectl apply -f 02-pod-monitor.yaml
podmonitor.monitoring.coreos.com/demo-kafka-exporter created
# Grafana users can import the dashboards here: https://github.com/strimzi/strimzi-kafka-operator/tree/main/examples/metrics/grafana-dashboards
```

### Test the cluster from the LoadBalancer services

Start a consumer from one terminal:

```bash
LB_IP=$(kubectl get svc demo-kafka-loadblancer-bootstrap '-ojsonpath={.status.loadBalancer.ingress[0].ip}')
kafka-console-consumer --bootstrap-server "$LB_IP:9093" --topic test-topic-1
```

and a producer from another terminal:

```bash
LB_IP=$(kubectl get svc demo-kafka-loadblancer-bootstrap '-ojsonpath={.status.loadBalancer.ingress[0].ip}')
while true; do uuidgen; sleep 1; done | kafka-console-producer --bootstrap-server "$LB_IP:9093" --topic test-topic-1
```

### Test the Kafka Exporter

```
$ kubectl exec -it $(kubectl get pod -o name -lapp.kubernetes.io/name=kafka-exporter) -- curl localhost:9404/metrics
# HELP go_cgo_go_to_c_calls_calls_total Count of calls made from Go to C by the current process.
# TYPE go_cgo_go_to_c_calls_calls_total counter
go_cgo_go_to_c_calls_calls_total 0
# HELP go_gc_cycles_automatic_gc_cycles_total Count of completed GC cycles generated by the Go runtime.
# TYPE go_gc_cycles_automatic_gc_cycles_total counter
go_gc_cycles_automatic_gc_cycles_total 0
```

## Demo 3: Ingress

This demo showcases how to set up a Kafka cluster to use Ingresses to access the cluster from outside the cluster.

> You might need to change the hostnames and issuer settings for the files in this demo for it work properly.

```
# Create a certificate trough cert-manager
$ kubectl apply -f 03-certificate.yaml
certificate.cert-manager.io/v1/kafka-demo configured
# Create an ingress listener
$ kubectl apply -f 03-ingress.yaml
kafka.kafka.strimzi.io/demo configured
# Verify ingresses are created property
$ kubectl get ingress
NAME                           CLASS           HOSTS                                  ADDRESS        PORTS     AGE
demo-kafka-ingress-0           nginx           kafka-0.example.com           10.200.10.240   80, 443   2m13s
demo-kafka-ingress-1           nginx           kafka-1.example.com           10.200.10.240   80, 443   2m13s
demo-kafka-ingress-2           nginx           kafka-2.example.com           10.200.10.240   80, 443   2m13s
demo-kafka-ingress-bootstrap   nginx           kafka-bootstrap.example.com   10.200.10.240   80, 443   2m13s
```

### Test access trough ingress

> Don't forget to change the domain names! If your certificates are self-signed, you might need additional
configuration in kafka-client.props.

Start a consumer from one terminal:

```bash
kafka-console-consumer --bootstrap-server "kafka-bootstrap.example.com:443" --topic test-topic-1 --consumer.config kafka-client.props
```

and a producer from another:

```bash
while true; do uuidgen; sleep 1; done | kafka-console-producer --bootstrap-server "kafka-bootstrap.example.com:443" --topic test-topic-1 --producer.config kafka-client.props
```

# Demo 4: Replica rebalance

This demo shows how to use Cruise Control to perform a rebalance on the cluster.

```
# Scale up the Kafka cluster and enable Cruise Control
$ kubectl apply -f 04-scale-cc.yaml
kafka.kafka.strimzi.io/demo configured
```

See how broker #3 is not leader nor follower for any partition and there is a need to rebalance:

```bash
kubectl exec demo-kafka-0 -- bash -c "./bin/kafka-topics.sh --list --bootstrap-server demo-kafka-bootstrap:9092 | xargs -L1 ./bin/kafka-topics.sh --describe --bootstrap-server demo-kafka-bootstrap:9092 --topic"
```

If you set up Prometheus trough the PodMonitor, you can also use the following queries:

```
count(kafka_topic_partition_leader == 0)
count(kafka_topic_partition_leader == 1)
count(kafka_topic_partition_leader == 2)
count(kafka_topic_partition_leader == 3)
```

```
# Create a rebalance and wait for proposal to be ready
$ kubectl apply -f 04-rebalance.yaml
kafkarebalance.kafka.strimzi.io/demo-rebalance created
$ kubectl get kafkarebalance
NAME             CLUSTER   PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY
demo-rebalance   demo
$ kubectl get kafkarebalance
NAME             CLUSTER   PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY
demo-rebalance   demo                        True
# Approve and start the rebalance
$ kubectl annotate kafkarebalance demo-rebalance strimzi.io/rebalance=approve
kafkarebalance.kafka.strimzi.io/demo-rebalance annotated
# Rebalance will start
$ kubectl get kafkarebalance
NAME             CLUSTER   PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY
demo-rebalance   demo                                        True
# When finished
$ kubectl get kafkarebalance demo-rebalance
NAME             CLUSTER   PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY
demo-rebalance   demo                                                      True
```


Verify broker #3 is now assigned to some partitions:

```
kubectl exec demo-kafka-0 -- bash -c "./bin/kafka-topics.sh --list --bootstrap-server demo-kafka-bootstrap:9092 | xargs -L1 ./bin/kafka-topics.sh --describe --bootstrap-server demo-kafka-bootstrap:9092 --topic"
```
