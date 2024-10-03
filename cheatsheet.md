
## Streams

Run the following:

```
kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic california_state_orders --from-beginning \
  --key-deserializer org.apache.kafka.common.serialization.StringDeserializer \
  --value-deserializer org.apache.kafka.common.serialization.IntegerDeserializer \
  --property print.key=true \
  --property key.separator=,
```

Run the following
```
kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic state_orders_count --from-beginning \
  --key-deserializer org.apache.kafka.common.serialization.StringDeserializer \
  --value-deserializer org.apache.kafka.common.serialization.LongDeserializer \
  --property print.key=true \
  --property key.separator=,
```

## CQRS 

### Launch Initial Containers

1. Open the _cqrs_ module folder
2. Right-click on the _docker-compose.yml_ file and select "Compose Up - Select Services", deselect all the checkmarks, and select `connect ksqldb-cli postgres control-center mongo mongo-express`
3. Login into `connect` container by using either `Attach Shell` on Gitpod or `docker exec -it connect /bin/bash`

### Create a JDBC Connector

1. Run the following in the container a JDBC Connect that reads from postgres - `confluent-hub install confluentinc/kafka-connect-jdbc:10.7.1`
2. Select `2. / (where this tool is installed)`
3. Answer `y` to `Do you want to install this into /usr/share/confluent-hub-components?`
4. Answer `y` to `I agree to the software license agreement (yN)`
5. Answer `y` to `Do you want to continue?`
6. Answer `y` to `Do you want to update all detected configs? (yN)`

### Create a MongoDB Connector

1. Run the following in the container `confluent-hub install mongodb/kafka-connect-mongodb:1.11.2`, or whatever the latest version is from https://confluent.io/hub[Confluent Hub]
2. Select `2. / (where this tool is installed)`
3. Answer `y` to `Do you want to install this into /usr/share/confluent-hub-components?`
4. Answer `y` to `I agree to the software license agreement (yN)`
5. Answer `y` to `Do you want to continue?`
6. Answer `y` to `Do you want to update all detected configs? (yN)`
7. Exit the container using `exit`
8. Restart the container in GitPod or using `docker restart connect`

### Run the Data Generator

1. Run the `CreateStocks` application by doing the following:

    a. Create a new terminal
    
    b. `cd cqrs`

    c. `mvn clean compile exec:java -Dexec.mainClass=com.evolutionnext.cqrs.CreateStocks` to generate data.

### View the Postgres Database

1. Login into your `postgres` container using `Attach Shell` or `docker exec -it postgres /bin/bash`
2. Run the following: `export PGPASSWORD='docker'`
3. Run the following: `psql -d docker -U docker`
4. In the Postgres shell run  `\dt` which will show all the tables
5. In the Postgres shell run `\d stock_trade`, which will show specific table schema
6. Run `SELECT * from stock_trade;` and ensure that the data exists
7. Exit the `postgres` container by [CTRL+D] and typing `exit` in the shell

### Create the Postgres Connector

1. Log into the Confluent Control Center
2. Select your cluster `controlcenter.cluster`
3. Select _Connect_ in the menu
4. Select the _connect_default_ cluster
5. Select the [Add Connector] button
6. Select the [JdbcSourceConnector] button
7. Add the following in the respective fields:

   a. *Key Converter Class* - `io.confluent.connect.avro.AvroConverter`

   b. *Value Converter Class* - `io.confluent.connect.avro.AvroConverter`

   c. *JDBC URL* - `jdbc:postgresql://postgres:5432/`
   
   d.*JDBC User* - `docker`

   e. *JDBC Password* - `docker`
   f. *Database Dialect* `PostgreSqlDatabaseDialect`
   g. *Table Loading Mode* `incrementing`
   h. *Topic Prefix* - `postgres_`
   i. *Additional Properties* -  `key.converter.schema.registry.url` set to  `http://schema-registry:8081`
   j. *Additional Properties* - `value.converter.schema.registry.url` set to `http://schema-registry:8081`
8. Click [Next]
9. Verify the JSON output, then select [Launch]
10. Go back to the home page of the Confluent Control Center
11. Go to the topics, and select _postgres_stock_trade_
12. Select the _Messages_ menu
13. View the data coming for data loading
14. You can stop the database loading by initiating [CTRL+C]

### Enrich the Data using KSQLDB


1. Go to KSQL-CLI Container by either attaching to the `ksqldb-cli` shell using `docker exec ksqldb-cli /bin/bash`
2. Run a ksql terminal that will attach to the KSQLDB Server using the following command

````
$ ksql http://ksqldb-server:8088
````

3. In the KSQLDB CLI, Create a Stream

````
CREATE STREAM stock_trades WITH (
KAFKA_TOPIC = 'postgres_stock_trade',
VALUE_FORMAT = 'AVRO'
);
````

4. Enter into the CLI the following:

````
SET 'auto.offset.reset'='earliest';
````

5. Show the live data coming from the source

````
select * from STOCK_TRADES emit changes;
````

6. Let's try something fancy, let's get a count of all the stocks and their count
````
select STOCK_SYMBOL, AS_VALUE(STOCK_SYMBOL) as symbol, count(*) as count from STOCK_TRADES group by stock_symbol EMIT CHANGES;
````

7. Create an aggregate topic from the above statement

````
create table stock_count with (PARTITIONS = 3, VALUE_FORMAT = 'JSON') as select STOCK_SYMBOL, AS_VALUE(STOCK_SYMBOL) as symbol, count(*) as count from STOCK_TRADES group by stock_symbol EMIT CHANGES;
````

8. Go to the topics, and select _STOCK_COUNT_
9. Select the _Messages_ menu
10. View the data coming for data loading

### Create a MongoDB Sink

1. Go back to the _Confluent Control Center_
2. Click on the [Connect] menu
3. Select the _connect_default_ cluster
4. Click on the [Upload connector config file] button
5. Select the file from the _cqrs_ module _src/main./resources/mongosink.json_
6 . Click [Next]
7. Verify the JSON output, then select [Launch]
8. Open the browser to the `mongo-express` container, port `10002` using the admin username `admin` and password `pass`
9. Locate the database _STOCK_COUNT_
10. Locate the collection _stock_counts_
11. Click [View]
