# [Astra Planeti](https://en.wikipedia.org/wiki/Astra_Planeta)

### Tech stack:
* Strimzi 
* Mysql
* Kafka Connect
* Debezium

## Spin up Mysql 

*If your Mysql is already running somewhere safely skip this section.*

```
docker run -it --rm --name mysql -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser \
  -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:1.0

docker run -it --rm --name mysqlterm --link mysql --rm mysql:5.7 sh \
  -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" \
  -P"$MYSQL_PORT_3306_TCP_PORT" \
  -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'

mysql> use inventory;
mysql> show tables;
```

## Install Strimzi

*To make life even easier: we install Strimzi, the Kube Operator. You can skip this section too.*

```
kubectl create namespace kafka

curl -L https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.16.1/strimzi-cluster-operator-0.16.1.yaml \
  | sed 's/namespace: .*/namespace: kafka/' \
  | kubectl apply -f - -n kafka

kubectl -n kafka \
    apply -f https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/0.16.1/examples/kafka/kafka-persistent-single.yaml \
  && kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka
```

## Create a Docker image for your Kafka Connect cluster

*Finally something to do: create a docker image from your connector(s)*

```
curl https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/1.5.3.Final/debezium-connector-mysql-1.5.3.Final-plugin.tar.gz \
| tar xvz

curl https://repo1.maven.org/maven2/org/apache/kafka/connect-transforms/2.8.0/connect-transforms-2.8.0.jar -o connect-transforms.jar

export DOCKER_ORG=leonardobonacci
export IMAGE_NAME=connect-debezium2
docker build . -t ${DOCKER_ORG}/${IMAGE_NAME}
docker push ${DOCKER_ORG}/${IMAGE_NAME}
```

## MYSQL credentials
*Now go find your credentials!!*

```
cat <<EOF > debezium-mysql-credentials.properties
mysql_username: debezium
mysql_password: dbz
EOF
kubectl -n kafka create secret generic my-sql-credentials \
  --from-file=debezium-mysql-credentials.properties
rm debezium-mysql-credentials.properties
```

## Spin us your Kafka Connect cluster

*We're getting there... spin up your personal connect-cluster!*
*Make sure to set metadata.my-connector-cluster to something personal, and to configure bootStrapServers correctly.* 

```
cat <<EOF | kubectl -n kafka apply -f -
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
metadata:
  name: my-connect-cluster
  annotations:
  # use-connector-resources configures this KafkaConnect
  # to use KafkaConnector resources to avoid
  # needing to call the Connect REST API directly
    strimzi.io/use-connector-resources: "true"
spec:
  image: ${DOCKER_ORG}/${IMAGE_NAME}
  replicas: 1
  bootstrapServers: my-cluster-kafka-bootstrap:9093
  tls:
    trustedCertificates:
      - secretName: my-cluster-cluster-ca-cert
        certificate: ca.crt
  config:
    config.storage.replication.factor: 1
    offset.storage.replication.factor: 1
    status.storage.replication.factor: 1
    config.providers: file
    config.providers.file.class: org.apache.kafka.common.config.provider.FileConfigProvider
  externalConfiguration:
    volumes:
      - name: connector-config
        secret:
          secretName: my-sql-credentials

EOF
```

## Tandem: deploy your connector

*Notice spec.config.database.hostname!*
*Local hack is to find the containers ip address executing 'docker network inspect bridge'*
*In a real world scenario this is given.*
*Connector config docs: https://debezium.io/documentation/reference/connectors/mysql.html*

```
cat | kubectl -n kafka apply -f - << 'EOF'
apiVersion: "kafka.strimzi.io/v1alpha1"
kind: "KafkaConnector"
metadata:
  name: "inventory-connector"
  labels:
    strimzi.io/cluster: my-connect-cluster
spec:
  class: io.debezium.connector.mysql.MySqlConnector
  tasksMax: 1
  config:
    key.converter: "org.apache.kafka.connect.json.JsonConverter"
    key.converter.schemas.enable: "false"
    value.converter: "org.apache.kafka.connect.json.JsonConverter"
    value.converter.schemas.enable: "false"
    database.hostname: 172.17.0.2
    database.port: "3306"
    database.user: "${file:/opt/kafka/external-configuration/connector-config/debezium-mysql-credentials.properties:mysql_username}"
    database.password: "${file:/opt/kafka/external-configuration/connector-config/debezium-mysql-credentials.properties:mysql_password}"
    database.server.id: "184054"
    database.server.name: "dbserver1"
    database.whitelist: "inventory"
    database.history.kafka.bootstrap.servers: "my-cluster-kafka-bootstrap:9092"
    database.history.kafka.topic: "schema-changes.inventory"
    transforms: "unwrap,extractKey"
    transforms.unwrap.type: "io.debezium.transforms.ExtractNewRecordState"
    transforms.unwrap.drop.tombstones: "false"
    transforms.extractKey.type: "org.apache.kafka.connect.transforms.ExtractField$Key"
    transforms.extractKey.field: "id"
EOF
```

## Show me the money!

```
kubectl get kctr inventory-connector -o yaml -n kafka

kubectl -n kafka exec my-cluster-kafka-0 -c kafka -i -t -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --list

kubectl -n kafka exec my-cluster-kafka-0 -c kafka -i -t -- \
  bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic dbserver1.inventory.customers \
    --property print.key=true

mysql> UPDATE customers SET first_name='Anne Marie' WHERE id=1004;
```

![Enfim](https://github.com/LeonardoBonacci/astra-planeti/blob/strimzi/images/working-version-consumer.png)

## Clean your room!!!

```
curl -L https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.16.1/strimzi-cluster-operator-0.16.1.yaml \
  | sed 's/namespace: .*/namespace: kafka/' \
  | kubectl delete -f - -n kafka
```

## [Gratias tibi](https://strimzi.io/blog/2020/01/27/deploying-debezium-with-kafkaconnector-resource/)

