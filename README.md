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

# Confluent Platform
## TODO: explicar que es cada cosa, poner capturas

# Exportar de Confluent Kafka datos a ElasticSearch usando KSQLDB y kafka-connect
Aunque todo lo que se va a explicar a continuación se puede ver y realizar a través de la interfaz gráfica de la plataforma de Confluent, se ha optado por realizarlo mediante línea de comandos desde el interior del contenedor **ksqldb-server**.
- Ejecutar shell del contenedor **ksqldb-server**:
```
sudo docker exec -it ksqldb-server /bin/bash
```
-  Iniciar el cliente ksql, que permite escribir y ejecutar consultas SQL-like para procesar eventos en tiempo real. 
```
ksql
```
- Ver topics existentes en el entorno de Kafka:
```
show topics;
```
- Crear STREAM que se exportará a ElasticSearch

``` 
CREATE STREAM PRUEBA (id VARCHAR, location STRUCT<lat DOUBLE,long DOUBLE>) WITH (KAFKA_TOPIC='localizacion', VALUE_FORMAT='JSON', PARTITIONS=10);
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
**Nota:**
Para ponder usar ```Elasticsearch Connector``` será muy importante tenerlo instalado y configurado en el entorno. Esto se realiza de forma automática en el docker-compose, en la creación del contenedor ```connect```:
```sh
confluent-hub install --no-prompt confluentinc/kafka-connect-elasticsearch:11.0.1
```
Repositorio de conectores de Confluent: [Confluent Hub](https://www.confluent.io/hub/?utm_medium=sem&utm_source=google&utm_campaign=ch.sem_br.brand_tp.prs_tgt.confluent-brand_mt.mbm_rgn.emea_lng.eng_dv.all_con.confluent-hub&utm_term=%2Bconfluent%20%2Bhub&creative=&device=c&placement=&gad_source=1&gclid=CjwKCAiA75itBhA6EiwAkho9ezjJsYhIJF5xElmRRmuf6wwkbUqg4mvRZK-Atr4SWdM2L7GI9-T9jRoCpWIQAvD_BwE)
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
