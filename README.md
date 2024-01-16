# Crear entorno mediante Docker
```
sudo docker compose up
```
**Notas**:
- Dado que el entorno consiste en un gran número de contenedores, la primera vez que se ejecute, será necesario descargar todas las imágenes, por lo que se espera que el proceso tome un tiempo considerable.
- Es recomendable eliminar contenedores que no estamos usando previamente para mantener limpio el entorno y evitar posibles conflictos. Para eliminar todos los contenedores detenidos, podemos usar: 
```
sudo docker container prune
```

Una vez levantados todos los cotenedores, podremos acceder a las interfaces gráficas de los mismos mediante las siguientes URLs:
- [Node-RED](http://localhost:1880)
- [HiveMQ](http://localhost:8001) (**usuario:** admin, **contraseña:** hivemq)
- [Confluent Platform](http://localhost:9021)
- [ElasticSearch&Kibana](http://localhost:5601)

# Node-RED

# HiveMQ
El bróker de HiveMQ será el que reciba los datos de los sensores mediante MQTT. El objetivo es que estos datos sean exportados a la plataforma de Confluent Kafka. Para ello, deberemos:
- Habilitar la extension que permite exportar datos a Kakfa: ```hivemq-kafka-extension``` eliminando el archivo DISABLED ubicado en ```/opt/hivemq-4.24.0/extensions/hivemq-kafka-extension```
- Crear un archivo de configuración denominado ```kafka-configuration.xml``` para definir el **cluster de kafka** y el mapeo del topic **/localizacion** a Kafka:
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<kafka-configuration xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                     xsi:noNamespaceSchemaLocation="config.xsd">
    <kafka-clusters>
        <kafka-cluster>
            <id>cluster01</id>
            <bootstrap-servers>broker:29092</bootstrap-servers>
        </kafka-cluster>
    </kafka-clusters>
    <mqtt-to-kafka-mappings>
        <mqtt-to-kafka-mapping>
            <id>mapping01</id>
            <cluster-id>cluster01</cluster-id>
            <mqtt-topic-filters>
                <mqtt-topic-filter>localizacion</mqtt-topic-filter>
            </mqtt-topic-filters>
            <kafka-topic>localizacion</kafka-topic>
        </mqtt-to-kafka-mapping>
    </mqtt-to-kafka-mappings>
</kafka-configuration>
```
**Nota**:
Lo explicado en este apartado se ha configurado para que se realice de forma automática a la hora de crear la imagen y el contenedor de hivemq. Si se deseara cambiar la configuración de la extensión de Kafka para, por ejemplo, mapear nuevos topics, únicamente habría que editar el archivo de configuración ubicado este repositorio en ```hivemq/kafka-configuration.xml```


# Exportar datos a ElasticSearch usando KSQLDB
- Desde ksql:
```
sudo docker exec -it ksqldb-server /bin/bash
ksql
```
- Ver topics
```
show topics;
```
- Crear STREAM que se exportará a ElasticSearch

``` 
CREATE STREAM PRUEBA (COL1 INT) WITH (KAFKA_TOPIC='temperatura', PARTITIONS=10, VALUE_FORMAT='JSON');
```
- Ver streams:
```
show streams;
```

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
