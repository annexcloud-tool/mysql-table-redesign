# Streaming MySQL Data to Amazon S3 using Debezium and Kafka Connect

This guide outlines the steps to set up a data streaming pipeline from MySQL to Amazon S3 using Debezium for change data capture and Kafka Connect with the kafka-connect-s3 sink connector for streaming data storage.

### Prerequisites
- **MySQL Database:** Ensure your MySQL database is configured with binary logging enabled. Modify your MySQL configuration (my.cnf or my.ini) to include:

```mysql
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-format=row

```

- **Apache Kafka and Kafka Connect:** Set up Apache Kafka and Kafka Connect. Debezium operates as a Kafka Connect connector, so you need these components to run Debezium.
- **AWS Account:** You need an AWS account with access to create and manage an S3 bucket.


## Steps
1. Configure Debezium MySQL Connector
> Create a JSON configuration file for the Debezium MySQL connector (mysql-connector.json):

```json
{
  "name": "mysql-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "localhost",
    "database.port": "3306",
    "database.user": "mysqluser",
    "database.password": "mysqlpassword",
    "database.server.id": "1",
    "database.server.name": "my-app-connector",
    "database.whitelist": "mydatabase",
    "table.whitelist": "mydatabase.mytable",
    "snapshot.mode": "initial",
    "snapshot.locking.mode": "none",
    "include.schema.changes": "true",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter.schemas.enable": "false"
  }
}

```

Adjust `database.hostname`, `database.user`, `database.password`, `database.whitelist`, `table.whitelist` according to your MySQL setup and the specific tables you want to capture changes from.

2. Deploy Debezium MySQL Connector

Deploy the Debezium MySQL connector using the Kafka Connect REST API or a UI. For example, using curl:

```bash
curl -X POST -H "Content-Type: application/json" --data @mysql-connector.json http://localhost:8083/connectors
```


3. Configure Kafka Connect S3 Sink Connector

Create a JSON configuration file for the `kafka-connect-s3 sink` connector (`s3-sink-config.json`):

```json
{
  "name": "s3-sink-connector",
  "config": {
    "connector.class": "io.confluent.connect.s3.S3SinkConnector",
    "tasks.max": "1",
    "topics": "your-topic-name",  // Replace with your Debezium topic name
    "s3.region": "your-s3-region",
    "s3.bucket.name": "your-s3-bucket",
    "s3.part.size": "5242880",  // 5 MB part size
    "flush.size": "3",  // Number of records to write to S3 before committing the files
    "storage.class": "io.confluent.connect.s3.storage.S3Storage",
    "format.class": "io.confluent.connect.s3.format.json.JsonFormat",
    "schema.compatibility": "NONE",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "partitioner.class": "io.confluent.connect.storage.partitioner.DefaultPartitioner",
    "locale": "en_US"
  }
}
```

Adjust `topics`, `s3.region`, `s3.bucket.name`, `flush.size`, and other parameters based on your specific configuration needs and AWS environment.

4. Deploy Kafka Connect S3 Sink Connector
Deploy the `kafka-connect-s3` sink connector using the Kafka Connect REST API or a UI:

```bash
curl -X POST -H "Content-Type: application/json" --data @s3-sink-config.json http://localhost:8083/connectors
```

5. Verify Data Flow
 - Monitor Kafka Connect and S3 logs to ensure both connectors start successfully.
 - Check your S3 bucket to verify that data files are being written as expected.

## Additional Considerations

- **Security**: Ensure AWS credentials (access key and secret key) are securely configured.
- **Performance**: Monitor performance of the connectors and adjust configurations as needed.
- **Data Formats**: Ensure data formats match between Debezium/Kafka topics and S3 storage.

## References
- [Debeziun mysql connector configuration](https://debezium.io/documentation/reference/stable/connectors/mysql.html)
- [Real time CDC replications between MySQL and PostgreSQL using Debezium](https://timothyzhang.medium.com/real-time-cdc-replications-between-mysql-and-postgresql-using-debezium-connectors-24aa33d58f1e)
- [Github: CDC-MySQL-Debezium-PostgreSQL](https://github.com/AmanLonare/CDC-MySQL-Debezium-PostgreSQL)
- [Streaming-data-from-mysql-into-apache-kafka](https://sandeepkattepogu.medium.com/streaming-data-from-mysql-into-apache-kafka-db9b5468667e)
- [Streaming-data-from-mysql-into-bigquery](https://mohamed-dhaoui.medium.com/data-streaming-journey-moving-from-postgresql-to-bigquery-with-kafka-connect-and-debezium-2679fdbbffd0)
- [Streaming-data-from-mysql-to-aws-s3]()
- [Streaming from mysql to s3 tool (www.decodable.co)](https://www.decodable.co/data-movement/mysql-to-s3)
- [Streaming-data-from-mysql-into-kinesis](https://debezium.io/blog/2018/08/30/streaming-mysql-data-changes-into-kinesis/)
- [Streaming-data-from-mysql-into-aws-msk](https://blog.searce.com/realtime-cdc-from-mysql-using-aws-msk-with-debezium-28da5a4ca873)
- [Streaming-data-from-postgresql-to-s3](https://medium.com/@joaopaulonobregaalvim/streaming-data-from-postgresql-to-s3-using-debezium-kafka-and-python-16c6cdd6dc1e)
- [CDC-pipeline-using-debezium-kafka-aws-s3](https://medium.com/@dinunirmani9d/changed-data-capture-pipeline-using-debezium-kafka-aws-s3-athena-1a53d91801c7)