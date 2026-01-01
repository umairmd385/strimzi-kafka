# Kafka on Kubernetes with Strimzi (KRaft Mode) 🚀

This repository contains the production-ready manifests for deploying a **Zookeeper-less** Apache Kafka cluster on Kubernetes using the **Strimzi Operator**.

## 📑 Why Strimzi + KRaft?

Running Kafka on Docker often leads to networking "listener" headaches and manual scaling pains. This setup solves those issues by using:

* **KRaft Mode:** Eliminates Zookeeper dependency by using the Raft consensus protocol for metadata.
* **KafkaNodePools:** Allows independent scaling and hardware configuration for **Brokers** and **Controllers**.
* **JBOD Storage:** Enables high-throughput data processing by spreading logs across multiple persistent disks.
* **Cruise Control:** Automates partition rebalancing and cluster self-healing.

---

## 🛠️ Step-by-Step Setup Guide

### 1. Set Up Default Storage (EKS/EBS)

Kafka requires persistent storage. If you are on AWS EKS, run this command to ensure the EBS StorageClass is marked as the default for your Persistent Volume Claims (PVCs).

```bash
kubectl patch sc ebs-sc \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

```

### 2. Install the Strimzi Cluster Operator

Install the latest Strimzi Operator into the `kafka` namespace. This operator will manage the lifecycle of your Kafka cluster.

```bash
# Create the namespace first
kubectl create namespace kafka

# Install the Operator
kubectl apply -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka

```

### 3. Deploy the Metrics Config

Before starting Kafka, apply the `ConfigMap` containing the JMX Prometheus Exporter rules. This ensures your cluster is observable from day one.

```bash
kubectl apply -f kafka-metrics-config.yaml

```

### 4. Provision Node Pools & Kafka Cluster

Apply the manifests to create your separate Controller and Broker pools.

```bash
# Deploys Controller and Broker NodePools + Kafka Cluster
kubectl apply -f kafka-cluster.yaml

```

### 5. Verify the Deployment

Monitor the pods until they are all in a `Running` state. In KRaft mode, you will see pods for `controller` and `broker` pools separately.

```bash
kubectl get pods -n kafka -w

```

### 6. Create Your First Topic

Once the cluster is ready, create a topic using the Strimzi `KafkaTopic` resource:

```bash
kubectl apply -f kafka-topic.yaml

```

---

## 📊 Monitoring

A `Service` is included in this repo to expose JMX metrics on port `9404`. If you have Prometheus installed, it will automatically scrape these metrics using the provided annotations.

| Metric Port | Path | Protocol |
| --- | --- | --- |
| 9404 | /metrics | TCP |

