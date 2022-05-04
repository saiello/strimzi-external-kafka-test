
# Overview

Testing Strimzi operators with Kafka outside kubernetes


Prerequisites:

- minikube ( tested with v1.25.2 )
- kafka ( tested with RH AMQ Streams 1.7 )

## Start minikube

```
minikube start
```

## Start Kafka on localhost

Use the ip associated to host.minikube.internal as advertised listener.

```
export HOST_MINIKUBE_INTERNAL=$(minikube ssh grep host.minikube.internal /etc/hosts | awk '{print $1}')

${KAFKA_HOME}/bin/zookeeper-server-start.sh config/zookeeper.properties
${KAFKA_HOME}/bin/kafka-server-start.sh config/server.properties \
  -override listeners=PLAINTEXT://:9092,AUTHENTICATED://:9093,MINIKUBE://:9094 \
  -override advertised.listeners=PLAINTEXT://localhost:9092,AUTHENTICATED://${HOST_MINIKUBE_INTERNAL}:9093,MINIKUBE://${HOST_MINIKUBE_INTERNAL}:9094
```


## Create CA secrets

```

openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=MyClusterCA" -days 10000 -out ca.crt

kubectl create secret generic my-cluster-clients-ca --from-file=ca.key=ca.key
kubectl create secret generic my-cluster-clients-ca-cert --from-file=ca.crt=ca.crt
```

## Deploy Strimzi


```
kubectl create -f topic-operator
kubectl create -f user-operator
```

## Create some topic and users

```
kubectl create -f topics
```

Verify resources have been created and are ready.

```
kubectl get kafkatopic my-topic

NAME       CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
my-topic   my-cluster   1            1                    True
```

```
kubectl create -f users
```

```
kubectl get kafkauser

NAME    CLUSTER      AUTHENTICATION   AUTHORIZATION   READY
user1   my-cluster   tls              simple          True
user2   my-cluster   tls-external     simple          True
user3   my-cluster   scram-sha-512    simple          True
```


Verify Topics and ACLs have been created on Kafka

```
./bin/kafka-acls.sh --bootstrap-server localhost:9092 --list

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=my-topic, patternType=LITERAL)`:
        (principal=User:CN=user1, host=*, operation=READ, permissionType=ALLOW)
        (principal=User:CN=user2, host=*, operation=READ, permissionType=ALLOW)
        (principal=User:user3, host=*, operation=READ, permissionType=ALLOW)
        ...
```
