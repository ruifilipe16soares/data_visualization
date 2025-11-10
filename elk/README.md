# ELK Stack Setup (Elasticsearch, Logstash, Kibana)

This setup uses **Docker Compose** to deploy a secure and production-ready **ELK stack** with SSL certificates, authentication, and backup support.

---

## Starting the Stack

1. **Set environment variables** in a `.env` file (same directory as the `docker-compose.yml`):

   ```bash
   STACK_VERSION=8.12.2
   CLUSTER_NAME=elk-cluster
   LICENSE=basic
   MEM_LIMIT=2g
   ES_PORT=9200
   KIBANA_PORT=5601
   ELASTIC_PASSWORD=YourElasticPassword
   KIBANA_PASSWORD=YourKibanaPassword

YourPassword=The password you want to choose.

Start the containers:

`docker-compose up -d`

The setup process will:

Generate CA and SSL certificates under /usr/share/elasticsearch/config/certs

Configure Elasticsearch security and Kibana credentials

Wait until all services are healthy

Once running:

Elasticsearch: https://localhost:9200

Kibana: https://localhost:5601

2. **Anonymous User in Kibana**

Kibana was configured with an anonymous access provider, defined in `kibana.yml`:

```yaml
xpack.security.authc.providers:
  basic.basic1:
    order: 0
  anonymous.anonymous1:
    order: 1
    credentials:
      username: "anonymous_service_account"
      password: "anonymous_service_account_password"
```

This allows public (read-only) access to dashboards without login.
You can still use the elastic or kibana_system users for full administrative access.

If you want to disable anonymous access later, remove or comment the anonymous.anonymous1 block in `kibana.yml`.
