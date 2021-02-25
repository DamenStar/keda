### Keda 

Start by install [Keda](https://github.com/kedacore/keda) into the cluster and wait for it become ready:

```shell
kubectl apply -f deployment/keda-2.0.0-beta.yaml
kubectl rollout status deployment.apps/keda-operator -n keda
```

### Kafka

Next, if you don't have access to Kafka you can use these instructions to install Kafka into the cluster:

```shell
helm repo add confluentinc https://confluentinc.github.io/cp-helm-charts/
helm repo update
kubectl create ns kafka
helm install kafka confluentinc/cp-helm-charts -n kafka \
		--set cp-schema-registry.enabled=false \
		--set cp-kafka-rest.enabled=false \
		--set cp-kafka-connect.enabled=false \
		--set dataLogDirStorageClass=default \
		--set dataDirStorageClass=default \
		--set storageClass=default
kubectl rollout status deployment.apps/kafka-cp-control-center -n kafka
kubectl rollout status deployment.apps/kafka-cp-ksql-server -n kafka
kubectl rollout status statefulset.apps/kafka-cp-kafka -n kafka
kubectl rollout status statefulset.apps/kafka-cp-zookeeper -n kafka

When done, also deploy Kafka client and wait until it's ready:

```shell
kubectl apply -n kafka -f deployment/kafka-client.yaml
kubectl wait -n kafka --for=condition=ready pod kafka-client --timeout=120s
```

Next, create the `metric` topic which we will use:

> The number of `partitions` is connected to the maximum number of replicas Keda will create. 

```shell
kubectl -n kafka exec -it kafka-client -- kafka-topics \
		--zookeeper kafka-cp-zookeeper-headless:2181 \
		--topic metric \
		--create \
		--partitions 10 \
		--replication-factor 3 \
		--if-not-exists