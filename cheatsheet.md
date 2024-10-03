
## Streams Demo

Run the following:

```
./bin/kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic california_state_orders --from-beginning \
  --key-deserializer org.apache.kafka.common.serialization.StringDeserializer \
  --value-deserializer org.apache.kafka.common.serialization.IntegerDeserializer \
  --property print.key=true \
  --property key.separator=,
```

Run the following
```
./bin/kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic state_orders_count --from-beginning \
  --key-deserializer org.apache.kafka.common.serialization.StringDeserializer \
  --value-deserializer org.apache.kafka.common.serialization.LongDeserializer \
  --property print.key=true \
  --property key.separator=,
```
