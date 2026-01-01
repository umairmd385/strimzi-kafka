# 📖 Kafka Strimzi & KRaft: Comprehensive Guide

Welcome! This document provides a deep dive into how this Kafka cluster is architected, how to set it up, and how the "magic" of scaling works behind the scenes.

---

## 🏗️ Architecture Overview

This setup uses **Strimzi**, an operator that turns Kafka into a "Cloud Native" citizen. Instead of managing servers, we manage Custom Resources (YAML).

### Key Components:

* **KRaft Mode:** We use the latest Kafka protocol that removes the need for Zookeeper. Metadata is managed by the Kafka nodes themselves.
* **Node Pools:** We separate nodes into **Controllers** (the brains) and **Brokers** (the muscle/storage).
* **Strimzi Operator:** A pod that sits in your cluster, constantly watching your YAML and making sure the AWS infrastructure matches your desired state.

---

## 🚀 Quick Start Setup

### 1. Prepare the Environment (AWS EKS)

Kafka requires stable storage. Run these to set up the environment and the operator:

```bash
# 1. Set EBS as default storage class
kubectl patch sc ebs-sc -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# 2. Install Strimzi Operator
kubectl create namespace kafka
kubectl apply -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka

```

### 2. Deploy the Cluster

Apply the manifests in this order:

```bash
# Apply monitoring config first
kubectl apply -f 01-metrics-config.yaml

# Apply the NodePools and Cluster
kubectl apply -f 02-kafka-cluster.yaml

# Create your first topic
kubectl apply -f 03-kafka-topic.yaml

```

---

## 🤯 The "Confusion" Part: How Scaling Works

One of the most common questions is: *"If I scale from 3 brokers to 5, how does my app know about the 2 new ones without updating the connection string?"*

### The Metadata Discovery Protocol

Your application only needs **one** stable entry point (the Bootstrap Server).

1. **Initial Connection:** Your app connects to the `bootstrap.server` URL.
2. **Metadata Fetch:** The app asks, *"Who is in this cluster?"*
3. **Cluster Map:** Kafka returns a list of **all** 5 brokers, their IP addresses, and which partitions they own.
4. **Direct Communication:** Your app now talks directly to all 5 brokers, even if they weren't in your initial config.

**Result:** You can scale your cluster from 3 to 100 brokers, and you **never** have to restart your application or update its configuration.

---

## 📈 Scaling and Rebalancing

When you add new brokers, they are empty. To move data to them, we use **Cruise Control**.

### To Scale Up:

Modify your `KafkaNodePool` replica count:

```bash
kubectl scale kafkanodepool dev-kafka-nodepool-broker --replicas=5 -n kafka

```

### How Data Moves:

Because we have `autoRebalance` enabled in the `Kafka` resource, the Strimzi Operator automatically triggers a `KafkaRebalance`.

* **Step 1:** Cruise Control calculates a "Goal Violation" (some brokers are empty, some are full).
* **Step 2:** It generates a "Proposal" to move data.
* **Step 3:** The Operator executes the move.
* **Step 4:** Your application's metadata is updated, and it begins producing/consuming from the new brokers.

---

## 📊 Monitoring

Every Kafka pod has a **JMX Exporter** running. You can view raw metrics or hook them into Prometheus.

* **Metrics Endpoint:** `http://<pod-ip>:9404/metrics`
* **Key Metrics to Watch:**
* `kafka_server_brokermetadatametrics_fencedbroker_count`: Should be 0.
* `kafka_server_raftmetrics_current_state`: Shows who the leader is.
* `kafka_server_raftmetrics_commit_latency_avg`: Measures metadata performance.



---

## 🛡️ Best Practices for New Users

* **Don't manually delete PVCs:** Strimzi manages them. If you delete a pod, the PVC remains so data is preserved.
* **Resource Requests/Limits:** Kafka is memory-intensive. Always ensure your EKS nodes have enough RAM for the JVM (Heap) and the OS Page Cache.
* **Tolerations:** If you have taints on your nodes, ensure you add `tolerations` to your `Kafka` spec so the pods can actually schedule.
