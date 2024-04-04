
# Overview

Testing Strimzi operators with Kafka outside kubernetes


Prerequisites:

- minikube ( tested with v1.25.2 )
- kafka ( tested with RH AMQ Streams 1.7 )

## Related Links

- [Deploying the standalone Topic Operator](https://strimzi.io/docs/operators/latest/deploying.html#deploying-the-topic-operator-standalone-str)
- [Deploying the standalone User Operator](https://strimzi.io/docs/operators/latest/deploying.html#deploying-the-user-operator-standalone-str)

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





cfssl genkey -initca ca_root.json |  cfssljson -bare ca
cfssl genkey ca_intermediate.json | cfssljson -bare intermediate
cfssl sign -config config.json -profile CA -ca ca.pem -ca-key ca-key.pem intermediate.csr intermediate.json | cfssljson -bare intermediate

cfssl genkey strimzi.json | cfssljson -bare strimzi
cfssl sign -config config.json -profile clusterCA -ca intermediate.pem -ca-key intermediate-key.pem strimzi.csr strimzi.json | cfssljson -bare strimzi

	
cfssl genkey server.json | cfssljson -bare server
cfssl genkey admin.json | cfssljson -bare admin

cfssl sign -config config.json -ca intermediate.pem -ca-key intermediate-key.pem -profile server  server.csr | cfssljson -bare server
cfssl sign -config config.json -ca intermediate.pem -ca-key intermediate-key.pem -profile client  admin.csr | cfssljson -bare admin


# server bundle
cat server.pem >> server-bundle.crt
cat intermediate.pem >> server-bundle.crt
cat ca.pem >> server-bundle.crt


# client bundle 

cat admin.pem >> admin-bundle.crt
cat intermediate.pem >> admin-bundle.crt
cat ca.pem >> admin-bundle.crt


# Convert to pkcs12

openssl pkcs12 -export -in server-bundle.crt -inkey server-key.pem -out server-bundle.p12
openssl pkcs12 -export -in admin-bundle.crt -inkey admin-key.pem -out admin-bundle.p12

# Convert keys to PKCS8
openssl pkcs8 -topk8 -nocrypt -in ca-key.pem -out ca.key
openssl pkcs8 -topk8 -nocrypt -in intermediate-key.pem -out intermediate.key
openssl pkcs8 -topk8 -nocrypt -in clients-key.pem -out clients.key
openssl pkcs8 -topk8 -nocrypt -in strimzi-key.pem -out strimzi.key
