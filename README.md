# [Astra Planeti](https://en.wikipedia.org/wiki/Astra_Planeta)

### don't forget to check out the more interesting [strimzi](https://github.com/LeonardoBonacci/astra-planeti/tree/strimzi) branch!

### Tech stack:
* Strimzi (TODO)
* Mysql
* Kafka Connect
* Debezium

## Run me!

### Mysql 
`docker run -d -it --rm --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:1.5`


### Kafka 
> Stripped version of Confluent's docker-compose 6.2.0.

`docker-compose up -d`

### Kafka Connect with Debezium
> Official Confluent binaries 6.2.0.

`./bin/confluent-hub install debezium/debezium-connector-mysql:1.5.0`

`[root@IT304246 confluent-6.2.0]# ./bin/connect-distributed etc/kafka/connect-distributed.properties`

`curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @mysql-source.json`

`curl -X DELETE -H "Content-type: application/json" http://localhost:8083/connectors/inventory-connector | jq`

`curl localhost:8083/connectors/inventory-connector | jq`

### Mysql client
`winpty docker run -it --rm --name mysqlterm --link mysql --rm mysql:5.7 sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -
p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'`

`mysql> use inventory;`

`mysql> SELECT * FROM customers;`

`mysql> UPDATE customers SET first_name='Anne Marie' WHERE id=1004;`

### Enfim
`./kafka-console-consumer 
     --bootstrap-server localhost:9092 
     --topic dbserver1.inventory.customers 
     --property print.key=true`

![Enfim](https://github.com/LeonardoBonacci/astra-planeti/blob/main/images/kafka-console-consumer.png)

## Resources
* https://debezium.io/documentation/reference/tutorial.html
