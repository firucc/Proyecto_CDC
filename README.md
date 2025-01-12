# Guía para configurar y ejecutar el sistema de captura de datos en tiempo real con Debezium y Kafka

Este documento describe los pasos necesarios para levantar los contenedores, registrar el conector de MySQL para Debezium, listar los topics de Kafka y crear un consumidor que lea los cambios en la base de datos.

## 1. Levantar contenedores Docker
El archivo `docker-compose.yml` incluye los siguientes servicios:

- **Zookeeper**: Coordinador de Kafka.
- **Kafka**: Plataforma de mensajería distribuida.
- **MySQL**: Base de datos relacional.
- **Debezium**: Conector que captura los cambios en MySQL y los publica en Kafka.

Para iniciar los contenedores:
```bash
docker-compose up -d
```
Esto iniciará los contenedores en segundo plano.

### Contenido del archivo `docker-compose.yml`
```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    platform: linux/amd64
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:latest
    platform: linux/amd64
    container_name: kafka
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper

  mysql:
    image: mysql:8.0
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - "./db_movies_netflix_transact.sql:/docker-entrypoint-initdb.d/db_movies_netflix_transact.sql"
    ports: 
      - "3307:3306"

  debezium:
    image: debezium/connect:2.4
    platform: linux/amd64
    container_name: debezium
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: "mysql-connector"
      CONFIG_STORAGE_TOPIC: debezium_config
      CONFIG_STORAGE_REPLICATION_FACTOR: 1
      OFFSET_STORAGE_TOPIC: debezium_offset
      OFFSET_STORAGE_REPLICATION_FACTOR: 1
      STATUS_STORAGE_TOPIC: debezium_status
      STATUS_STORAGE_REPLICATION_FACTOR: 1
      REST_ADVERTISED_HOST_NAME: "mysql-connector"
      CONNECT_KEY_CONVERTER_SCHEMAS_ENABLE: "false"
      CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE: "false"
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_REST_ADVERTISED_HOST_NAME: debezium
      CONNECT_REST_PORT: 8083
      CONNECT_LOG4J_ROOT_LOGLEVEL: INFO
    ports:
      - "8083:8083"
    depends_on: 
      - kafka
      - mysql
```

## 2. Registrar el conector de MySQL para Debezium

### Contenido del archivo `register-mysql.json`
```json
{
  "name": "mysql-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "root",
    "database.password": "root",
    "database.server.id": "184054",
    "database.server.name": "netflix",
    "database.include.list": "db_movies_netflix_transact",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "schema-changes.db_movies_netflix_transact",
    "include.schema.changes": "true",
    "snapshot.mode": "initial",
    "snapshot.locking.mode": "none",
    "topic.prefix": "netflix",
    "schema.history.internal.kafka.topic": "schema-history-topic",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092"
  }
}
```

### Usando PowerShell
```powershell
Invoke-WebRequest -Uri "http://localhost:8083/connectors" `
  -Method POST `
  -Headers @{"Accept"="application/json"; "Content-Type"="application/json"} `
  -Body (Get-Content -Raw -Path "register-mysql.json")
```

### Usando Bash
```bash
curl -i -X POST -H "Accept: application/json" -H "Content-Type: application/json" \
http://localhost:8083/connectors -d @register-mysql.json
```

## 3. Listar topics de Kafka
Para verificar los topics creados por Debezium:
```bash
docker exec -it kafka kafka-topics --bootstrap-server=kafka:9092 --list
```

## 4. Crear un consumidor de Kafka
Este comando inicia un consumidor que leerá los mensajes del topic correspondiente a los cambios en la base de datos:
```bash
docker exec -it kafka kafka-console-consumer --bootstrap-server kafka:9092 --topic netflix.db_movies_netflix_transact.movie --from-beginning
```

## 5. Iniciar contenedor MySQL
Para conectarse al contenedor MySQL y utilizar la base de datos:
```bash
docker exec -it mysql mysql -u root -p db_movies_netflix_transact
```

## 6. Insertar datos en la tabla Movie de MySQL
Ejecutar el siguiente comando dentro del contenedor MySQL:
```sql
INSERT INTO movie  VALUES ("80194187", "Jon Down II03", "2019-04-11", "English", "https://www.netflix.com/pe-en/title/80194187");
UPDATE movie SET originalLanguage = 'Spanish' WHERE movieTitle = 'Jon Down II03';
```

