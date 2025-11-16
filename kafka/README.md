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

```
CRIAR CONECTORES:

curl -X POST -H "Content-Type: application/json" \
     --data @mysql-connector.json \
     http://localhost:8083/connectors

curl -X GET "http://localhost:8083/connectors/mysql-connector-chtv/status"

docker exec -it kafka sh -c "/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic mysql-chtv.chtv.pedido --from-beginning"


COPIAR O CERTIFICADO PARA A DIRETORIA root/ELK:
docker cp es01:/usr/share/elasticsearch/config/certs/ca/ca.crt ./ca.crt

Sai do Container e executa este comando na VM para copiar novamente o ficheiro com permissões abertas:
docker cp --archive /root/docker/docker/elk/ca.crt connect:/tmp/ca.crt

Então força a cópia com permissão global:
docker exec --user root connect keytool -import -trustcacerts -alias elastic-ca -file /tmp/ca.crt -keystore /etc/java/java-11-openjdk/java-11-openjdk-11.0.20.0.8-1.fc37.x86_64/lib/security/cacerts -storepass changeit -noprompt

DEPOIS REINICIEI O connect: 
docker restart connect




curl -X POST -H "Content-Type: application/json" \
     --data @elasticsearch-connector.json \
     http://localhost:8083/connectors


curl -X GET "http://localhost:8083/connectors/elasticsearch-connector-chtv/status"
