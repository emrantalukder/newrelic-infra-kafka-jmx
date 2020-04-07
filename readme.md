# Kafka Integration for New Relic (Kubernetes)

> Capture JMX metrics from Apache Kafka and publish to New Relic using New Relics' JMX plugin

## Getting Started

_Deploy with either the standalone yaml or helm chart_

### Install kube-state-metrics (required)
```sh
curl -L -o kube-state-metrics-1.7.2.zip \
https://github.com/kubernetes/kube-state-metrics/archive/v1.7.2.zip && \
unzip kube-state-metrics-1.7.2.zip && \
kubectl apply -f kube-state-metrics-1.7.2/kubernetes
```

### Deploy Standalone YAML

#### Modify `newrelic-infrastructure-k8s-latest.yaml` with CLUSTER_NAME and NRIA_LICENSE_KEY
```yaml
env:
  - name: NRIA_LICENSE_KEY
    value: YOUR_LICENSE_KEY
  - name: CLUSTER_NAME
    value: YOUR_CLUSTER_NAME
```

#### Create/Apply
```sh
kubectl create -f newrelic-infrastructure-k8s-latest.yaml
```

### Deploy with Helm

* `/newrelic-infrastructure-helm` is the helm chart available in `helm/charts` git repo
* `/newrelic-infrastructure-helm/values.yaml` has been updated to include JMX integration.
    * Uses discovery to find IP addresses of every each node for kafka, zookeeper, connect, ksql, streams, and schema registry

#### Replace $LICENSE_KEY and $CLUSTER_NAME and run the command below:
```sh
helm install newrelic-infra ./newrelic-infrastructure-helm \
    --namespace operator \
    --set licenseKey=$LICENSE_KEY \
    --set cluster=$CLUSTER_NAME
```

### Verify Daemonset
```
$ kubectl get daemonsets -n operator                
NAME                                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
newrelic-infra-newrelic-infrastructure   4         4         4       4            4           <none>          74m
```

## Appendix

### Install new relic logging agent

* clone repo:
```sh
git clone https://github.com/newrelic/kubernetes-logging.git
```

* install using helm:
```sh
helm install newrelic-logging \
--set licenseKey=$LICENSE_KEY \
./kubernetes-logging/helm/newrelic-logging
```

### New Replic Prometheus OpenMetrics integration (Kubernetes)

* Replace YOUR_LICENSE_KEY on line 70 of `nri-prometheus-latest.yaml` with your NR license key
* Update `cluster_name` on line 84 with your Kubernetes cluster name
* Update `urls` on line 106 with the URL of the running prometheus service (cp-operator used 7778)

#### Deploy 
```sh
# deploy
kubectl apply -f nri-prometheus-latest.yaml

# check
kubectl get deployments nri-prometheus
```

### Query metrics in New Relic with NRQL
```sql
FROM Metric SELECT count(*) WHERE clusterName = 'YOUR_CLUSTER_NAME' since 1 hour ago
```

### Create EKS Cluster

Create a Kubernetes cluster on AWS EKS with eksctl: 4 t3.xlarge nodes running version 1.15 in us-east-1

```sh
eksctl create cluster \
    --name=k8-emran \
    --version=1.15 \
    --nodes=4 \
    --node-type=t3.xlarge \
    --tags author=emran \
    --region=us-east-1 \
    --zones=us-east-1a,us-east-1b,us-east-1d
```

*note: zones are explicitly added to avoid error with us-east-1e*

### Deploy Confluent Platform with Confluent Operator

```sh

# create namespace
kubectl create namespace operator

# deploy operator
helm install operator ./confluent-operator -f ./providers/private.yaml --namespace operator --set operator.enabled=true

# deploy zookeeper
helm install zookeeper ./confluent-operator -f ./providers/private.yaml --namespace operator --set zookeeper.enabled=true

# deploy kafka
helm install kafka ./confluent-operator -f ./providers/private.yaml --namespace operator --set kafka.enabled=true

# deploy schema registry
helm install schemaregistry ./confluent-operator -f ./providers/private.yaml --namespace operator --set schemaregistry.enabled=true

# deploy kafka connect
helm install connectors ./confluent-operator -f ./providers/private.yaml --namespace operator --set connect.enabled=true

# deploy ksql
helm install ksql ./confluent-operator -f ./providers/private.yaml --namespace operator --set ksql.enabled=true

# deploy replicator
helm install replicator ./confluent-operator -f ./providers/private.yaml --namespace operator --set replicator.enabled=true

# deploy control center
helm install controlcenter ./confluent-operator -f ./providers/private.yaml --namespace operator --set controlcenter.enabled=true

```
