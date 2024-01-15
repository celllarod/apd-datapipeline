# Exportar datos a ElasticSearch usando KSQLDB
- Desde ksql:
```
> sudo docker exec ksqldb-server /bin/bash
> ksql
```
- Ver topics
```
show topics;
```
- Crear STREAM que se exportará a ElasticSearch

``` [source,sql]
CREATE STREAM PRUEBA (COL1 INT) WITH (KAFKA_TOPIC='temperatura', PARTITIONS=1, VALUE_FORMAT='JSON')
```
- Ver streams: ```show streams;```

- Crear connector para enviar stream a ElasticSearch (Para ID automatico: key.ignore='true', sino: false)
```
CREATE SINK CONNECTOR SINK_ELASTIC_PRUEBA WITH (
  'connector.class'         = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
  'connection.url'          = 'http://elasticsearch:9200',
  'key.converter'           = 'org.apache.kafka.connect.storage.StringConverter',
  'value.converter'         = 'org.apache.kafka.connect.json.JsonConverter',
  'value.converter.schemas.enable' = 'false',
  'type.name'               = '_doc',
  'topics'                  = 'temperatura',
  'key.ignore'              = 'true',
  'schema.ignore'           = 'true'
);
```
- Ver estado (debe ser 1/1):
 ```
 show connectors;
 ```
``` 
DESCRIBE CONNECTOR SINK_ELASTIC_PRUEBA;
```


 Si error, para ver qué pasa:
 ```
 docker compose logs -f connect
 ```

Para borrar:
```
DROP CONNECTOR SINK_ELASTIC_PRUEBA;
```

- Ver Datos en ElasticSearch (no me va)
```
docker compose exec elasticsearch curl -s http://localhost:9200/PRUEBA/_search -H 'content-type: application/json' -d '{ "size": 42  }' | jq -c '.hits.hits[]'
```

# Representar datos en Kibana, ElasticSearch
- Crear index
- Kivana>Management>Dev Tools:
```
GET temperatura/_search
```

# Referencias
- Exportar datos ElasticSearch:

https://www.youtube.com/watch?v=Cq-2eGxOCc8

https://github.com/confluentinc/demo-scene/blob/master/kafka-to-elasticsearch/README.adoc

https://docs.confluent.io/kafka-connectors/elasticsearch/current/overview.html

# Otros
### Guest Additions
```
sudo apt install gcc make perl
cd /media/<user>/VBox_GA...
sudo ./VBoxLinuxAdditions.run
```
