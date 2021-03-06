# Kuberentes horizontal autoscaling demo

This demo is using Kubernetes version 1.16 which required some changes. If you are using an older kubetentes version you need to change:
- `apps/v1` to `extensions/v1beta1` in kafka-consumer-application.yaml
-  You might also need to use an older kafka version too:
  - Change the strimzi version installed from `0.17.0` to `0.13.0`
  - Change the kafka version in the kafka.yaml from `2.4.0` to `2.3.0`

## Configuration

First we need to run and configure the metrics server:

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
```

After the metrics server is running we needs to edit its configuration:
```
kubectl edit deploy metrics-server -n kube-system
```

We should edit the file with the 2 kubelet options shown below (make sure you use spaces for the indentation):
```
containers:
  - args:
    - --cert-dir=/tmp
    - --secure-port=4443
    - --kubelet-insecure-tls
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
```

In case you dont have helm installed you should also install it: https://helm.sh/docs/intro/install/

## Running the demo

Create the namespace we'll be using:
```
kubectl create namespace consumer-test
```

Start strimzi:
```
curl -L https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.17.0/strimzi-cluster-operator-0.17.0.yaml \
  | sed "s/namespace: .*/namespace: consumer-test/" \
  | kubectl -n "consumer-test" apply -f -
```

Run kafka:
```
kubectl -n consumer-test create -f  kafka.yaml
```

After kafka is running we should create the topic that our application will use. So we get into the container:
```
kubectl -n consumer-test exec -it scraper-kafka-0 bash
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 10 --topic my-topic
```

Something to note here is that in the case of kafka, the numbers of partitions in the topic will limit the number of consumers we can have in the same consumer group, if we have 10 partitions we should make sure that we limit the autoscaler to 10 instances of the consumer.


Run the application thats going to autoscale automatically and the kafka exporter that exposes the kafka metrics:
```
kubectl -n consumer-test create -f kafka-application.yaml
kubectl -n consumer-test create -f kafka-exporter.yaml
```

Now we need to run prometheus which will scrape the kafka exporter hitting its `GET /metrics` endpoint, and the prometheus adapter which exposes the prometheus metrics to the kubernetes metrics server. Notice that we are configuring the adapter using the values.yaml, file which creates a new custom metric called kafka_consumergroup_lag and specifies the prometheus query needed to calculate it.
```
helm install prom-demo stable/prometheus
helm install prom-adapter stable/prometheus-adapter -f values.yaml
```

Finally we need to create the autoscaler to automatically scale our kafka-consumer-application based on the kafka lag:
```
kubectl -n consumer-test create -f consumer-hpa.yaml
```

We can check that everything is running correctly with:
```
kubectl -n consumer-test get all
```

And we can also open the kafka container again and start adding some messages into the kafka topic, causing the lag to increase and the kafka-consumer-application to scale automatically:
```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-topic
```

## Notes

Some good posts this was based one:

- https://levelup.gitconnected.com/building-kubernetes-apps-with-custom-scaling-a-gentle-introduction-a332d7ebc795
- https://medium.com/@ranrubin/horizontal-pod-autoscaling-hpa-triggered-by-kafka-event-f30fe99f3948
- https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

