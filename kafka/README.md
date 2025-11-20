# Kafka & Debezium Docker Setup

This **Docker Compose** file deploys a lightweight **Kafka + Zookeeper + Debezium Connect** stack used for real-time data streaming and integration with the ELK environment.

---

## Services Overview

- **Zookeeper:** Coordinates and manages Kafka brokers (port `2181`)  
- **Kafka:** Message broker for streaming events (port `9092`)  
- **Connect (Debezium):** Handles connectors such as **MySQL → Elasticsearch** (port `8083`)

---

## Network Configuration

All containers run inside the same Docker network used by the **ELK stack**:

```yaml
networks:
  elasticnet:
    external: true
    name: elk_elasticnet
```

## Plugin Installation (Before Starting the Container)

Before running the stack with `docker-compose up -d`, make sure the **Elasticsearch Kafka Connector** plugin is installed inside the `kafka-connect-plugins` folder.

---

### How to Install the Plugin via Command Line

Run the following command in the same directory as your `docker-compose.yml`:

```bash
mkdir -p kafka-connect-plugins
cd kafka-connect-plugins
wget https://packages.confluent.io/maven/io/confluent/kafka-connect-elasticsearch/14.1.2/kafka-connect-elasticsearch-14.1.2.zip
unzip kafka-connect-elasticsearch-14.1.2.zip -d confluentinc-kafka-connect-elasticsearch-14.1.2
rm kafka-connect-elasticsearch-14.1.2.zip
```
The connector’s JAR files are too large to store directly in the repository, so only the plugin folder structure is included.
Before starting the containers, you need to download the plugin manually as shown above.

After installation, you can safely start the Kafka stack:

`docker-compose up -d`

IMP: Don't forget to write your credentials in `secrets` folder, on both files.


## Creating the MySQL Connector (Debezium → Kafka → Elasticsearch)

This section explains how to create and configure the **MySQL Debezium connector**, verify its status, and securely connect it to Elasticsearch using SSL certificates.

---

### Create the Connector

Make sure your `mysql-connector.json` file is correctly configured (contains connection details, topics, etc.), then run:

```bash
curl -X POST -H "Content-Type: application/json" \
     --data @mysql-connector.json \
     http://localhost:8083/connectors
```

You can verify that the connector was successfully created with:

```bash
curl -X GET "http://localhost:8083/connectors/mysql-connector/status"
```

To check if data is being streamed to Kafka, consume messages from the topic:

```bash
docker exec -it kafka sh -c "/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server kafka:9092 \
  --topic your.example.topic \
  --from-beginning"
```
---

### Import the Elasticsearch CA Certificate

To allow **Kafka Connect** to securely communicate with **Elasticsearch**, the **CA certificate** must be imported into the Connect container.

1. **Copy the certificate from Elasticsearch (es01) to the host:**

   ```bash
   docker cp es01:/usr/share/elasticsearch/config/certs/ca/ca.crt ./ca.crt
   ```

2. **Copy the certificate to the Connect container:**

   ```bash
   docker cp --archive /root/docker/docker/elk/ca.crt connect:/tmp/ca.crt
   ```

3. **Import the CA certificate into the Java truststore inside the Connect container:**

   ```bash
   docker exec --user root connect keytool -import -trustcacerts \
     -alias elastic-ca \
     -file /tmp/ca.crt \
     -keystore /etc/java/java-11-openjdk/java-11-openjdk-11.0.20.0.8-1.fc37.x86_64/lib/security/cacerts \
     -storepass changeit -noprompt
   ```

The command above forces the certificate import using root privileges to ensure all Java-based connectors can trust the Elasticsearch endpoint.

---

### Restart the Connect Container

After importing the certificate, restart the Connect service so it can reload the truststore:

```bash
docker restart connect
```
---

 Once restarted, the MySQL Debezium connector will stream data from your MySQL database into Kafka topics, which can then be synchronized with Elasticsearch:


## Creating the Elasticsearch Connector (Kafka → Elasticsearch)

This section explains how to create the **Elasticsearch Sink Connector**, which reads messages from Kafka topics and indexes them into Elasticsearch.

Once the JSON file is saved, create the connector with:

```bash
curl -X POST -H "Content-Type: application/json" \
     --data @elasticsearch-connector.json \
     http://localhost:8083/connectors
```

Verify that the connector is running correctly:

```bash
curl -X GET "http://localhost:8083/connectors/elasticsearch-connector-chtv/status"
```

Once active, the connector will automatically pull data from Kafka topics and index them into your Elasticsearch cluster. 
-> If you open Kibana, in the tab "Indices" you can check the indices being created.

