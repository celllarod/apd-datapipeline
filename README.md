# Datapipeline: MQTT-Confluent Kafka y representadión de datos en Kivana

La gestión de los datos generados por dispositivos IoT es esencial para extraer información relevante y de interés. En este proyecto se va a implementar una canalización de datos IoT, utilizando tecnologías como Node-RED, MQTT, Confluent Kafka, HiveMQ, ElasticSearch y Kibana; con el objetivo de establecer una infraestructura que permita la recopilación, el procesamiento y la visualización de estos datos de interés.

Para ilustrar la implementación del pipeline, vamos a considerar un escenario en el cual obtenemos datos de la localización de aves migratorias a través de dispositivos IoT integrados en ellas. Visualizaremos la posición de estas aves en un mapa de Kibana.

![image](https://github.com/celllarod/apd-datapipeline/assets/59287155/1347cc64-3a7a-4df1-9f8d-5b6f7eff87de)
 
## Creación del entorno mediante Docker
La arquitectura completa ha sido configurada a través de un archivo Docker Compose. Esto significa que se puede replicar fácilmente en cualquier sistema que tenga Docker y Docker Compose instalados, simplemente ejecutando el siguiente comando:

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

## Componentes y configuración de la infraestructura
## Node-RED
Node-RED es una herramienta de desarrollo basada en flujo para programación visual desarrollada originalmente por IBM para conectar dispositivos de hardware, API y servicios en línea como parte de la Internet de las cosas. Proporciona un editor de flujo basado en navegador web, que se puede utilizar para crear funciones de JavaScript. Los elementos de las aplicaciones se pueden guardar o compartir para su reutilización. El tiempo de ejecución se basa en Node.js. Los flujos creados en Node-RED se almacenan mediante JSON.

Se ha elegido esta plataforma para simular los dispositivos fisicos que serían los encargados de tomar y transmitir las coordenadas de localización. Se han creado 4 bloques pertenecientes que simulan la localización de 4 aves diferentes y se ha unido a un bloque transmisor mediante mqtt, al que se le ha configurado los parametros necesarios (IP, puerto y QoS) para establecer la comunicación con el broker HiveMQ. Para facilitar el debug, se ha añadido un bloque receptor de mensajes mqtt conectado también al broóer para visualizar los mensajes. 

El flow de Node-RED ha sido guardado en un volumen de docker para facilitar el despliegue.

![image](https://github.com/celllarod/apd-datapipeline/assets/59287155/d3ce4593-2243-48b8-bb38-02a503013acb)



## HiveMQ
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

![image](https://github.com/celllarod/apd-datapipeline/assets/59287155/ebc8a8ae-91ad-485a-8f65-f15ffe0f9331)

![image](https://github.com/celllarod/apd-datapipeline/assets/59287155/6157cca7-7bfe-4fdd-9aee-5866f64837a4)

## Confluent Platform
<p align="justify">
Se trata de una plataforma de streaming basada en Apache Kafka cuyo objetivo es ayudar a los diferentes roles en la organización en los proyectos de intercambio de información en tiempo real. Para ello, desde un punto de vista de arquitectura tecnológica, mejora las capacidades nativas base de Apache Kafka en las siguientes perspectivas:
</p>

   - Capacidades de Streaming Enterprise
   - Conectividad y Gobierno del Dato
   - Procesamiento Avanzado
   - Monitorización y Gestión
   - Seguridad
   - Arquitecturas MultiSite y Apoyo en la evolución hacia el Cloud

### Confluent Server

El componente central de la arquitectura es el *Confluent Server* (contenedor **broker** en nuestro docker compose). Este componente, utilizando la tecnología de Apache Kafka, va a establecer el núcleo de coordinación de gestión de eventos en tiempo real.

Es capaz de:

   - Integrar fuentes productoras y consumidoras de eventos de diferente naturaleza
   - Gestionar las capacidades Real Time para transporte de altos volúmenes de datos (Megabytes por segundo)
   - Garantizar la resiliencia, escalabilidad y secuencialidad en la transferencia de eventos.
   - Colaborar con el resto de los componentes de arquitectura para la gestión de los metadatos, la distribución de carga dentro del clúster.

### Confluent Connect y Schema Registry
<p align="justify">
Confluent Platform simplifica la conectividad con productores y consumidores de información mediante un conjunto de más de 200 conectores pre-construidos soportados y verificados que acortan el tiempo de despliegue de la plataforma y garantizan su funcionamiento. Estos conectores son aplicaciones desarrolladas y mantenidas por Confluent que facilitan la lectura y escritura de sistemas terceros más habitualmente utilizados como Flume, S3, Splunk, HDFS, Oracle, ficheros, SAP Hana, Snowflake, MongoDB, IBM MQ, Mainframe… (más de 200).
</p>

La ingesta de eventos está integrada con otro de los componentes de Confluent Platform, *Confluent Schema Registry*, repositorio inteligente de esquemas compatible con Avro, JSON y Protobuf, que permite la validación de mensajes mejorando el desacople de productor/consumidor incorporando la capacidad de establecer “schemas” (estructuras pactadas) entre los mismos. Estos servicios se pueden encontrar en el docker compose bajo el nombre de **schema registry y connect**


### Confluent ksqlDB

Confluent Platform aporta capacidades avanzadas de transformación y el procesamiento de la información en tiempo real. Para ello, mejora la capacidad base de Apache Kafka basada en desarrollo de código de transformación, proponiendo el uso del componente **Confluent ksqlDB** como motor de transformación, filtrado y enriquecimiento durante el transporte de la información. Se usa mediante el lenguaje **SQL**.


### Confluent Control Center

Por último, **Confluent Control Center** como herramienta única de gestión y monitorización del sistema de Streaming en Tiempo Real. A través de él se dispone de una única aplicación web donde gestionar y monitorizar todos los aspectos relativos al despliegue de Confluent Platform, tanto en escenarios diseñados con clústeres únicos como en escenarios multicluster on-prem, SaaS o híbridos.

Utilizando todos los servicios mencionado anteriormente, en este trabajo se han seguido los siguientes pasos:

1) Crear el topic (localizacion) que recibirá los mensajes a traves de HiveMQ (automático).
2) Creación del stream de datos en ksql (*Se explica detalladamente en el apartado Exportar datos de Confluent Kafka a ElasticSearch usando KSQLDB y kafka-connect*).
3) Realizar la instalación dentro del contenedor connect la extensión para Elasticsearch (https://www.confluent.io/hub/confluentinc/kafka-connect-elasticsearch), esta instalización se hace de forma automática añadiendo los comandos necesarios en el fichero docker compose.
4) Entrar en el apartado connect de Confluent Control Center y realizar la configuración añadiendo la url de elasticsearch (se a realizado mediante la consola de ksql por comodidad).

**Nota**
Se puede encontrar mas documentación sobre la plataforma en [https://www.confluent.io/](https://docs.confluent.io/platform/7.5/overview.html)

# Parte práctica
Como práctica para completar la visualización de datos en Kibana, se propone seguir los siguientes pasos que se detallarán en los siguientes apartados:
1. Crear Índice en ElasticSearch al que se insertarán los datos desde Kafka como documentos.
2. Crear mapeo del campo "location" de los documentos del índice "localizacion" para que se transforme al formato **geo_point**.
3. Configurar un flujo KSQL para el topic "localizacion".
4. Crear conector de ElasticSearch en Congluent mediante la consola de ksql.
## ElasticSearch y Kibana
Para que los datos que se insertarán posteriormente en ElasticSearch desde Confluent Kafka puedan ser representados en un mapa de coordenadas de Kibana, debemos convertir las coordenadas del JSON de entrada en el formato **geo-point**.
``` json
{
"identificacion": "1",
"location":
    { "lat": 61.0666922, 
        "lon": -107.991707
    }
}
```
Para ello, se crea un index **localizacion** que mapeará el atributo **location** de los datos entrantes al formato deseado desde: [ElasticSearch&Kibana](http://localhost:5601) ```Management>Dev Tools```
``` sql
PUT localizacion
{
  "mappings": {
    "properties": {
      "location": {
        "type": "geo_point"
      }
    }
  }
}
```
A continuación, se debe crear un **index_pattern** para ```localizacion``` con el fin de poder representar el índice en Kibana posteriormente.
```Kibana>Index_pattens>Create Index pattern>``` -> Nombre: localizacion

## Exportar datos de Confluent Kafka a ElasticSearch usando KSQLDB y kafka-connect
Aunque todo lo que se va a explicar a continuación se puede ver y realizar a través de la interfaz de usuario de la plataforma de Confluent, se ha optado por realizarlo mediante línea de comandos desde el interior del contenedor **ksqldb-server**.
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

```sql
CREATE STREAM LOCALIZACION_STREAM (identificacion VARCHAR, location STRUCT<lat DOUBLE,lon DOUBLE>) WITH (KAFKA_TOPIC='localizacion', VALUE_FORMAT='JSON', PARTITIONS=10);
```
- Ver streams creados:
```
show streams;
```
- Crear connector para enviar stream a ElasticSearch (Para ID automatico: key.ignore='true', sino: false)
```sql
CREATE SINK CONNECTOR SINK_ELASTIC_LOCALIZACION WITH (
  'connector.class'         = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
  'connection.url'          = 'http://elasticsearch:9200',
  'key.converter'           = 'org.apache.kafka.connect.storage.StringConverter',
  'value.converter'         = 'org.apache.kafka.connect.json.JsonConverter',
  'value.converter.schemas.enable' = 'false',
  'type.name'               = '_doc',
  'topics'                  = 'localizacion',
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
```sql
DESCRIBE CONNECTOR SINK_ELASTIC_LOCALIZACION;
```
 Si error, para ver qué pasa:
 ```
 docker compose logs -f connect
 ```
Para borrar connector:
```
DROP CONNECTOR SINK_ELASTIC_LOCALIZACION;
```

## Representar datos en Kibana, ElasticSearch
- Comprobar que se están recibiendo los datos
```
GET temperatura/_search
```
- Crear mapa para representar coordenadas:
    -  Dashboard>Create Panel>Maps>
    - Add Layer>Documents>Index patterns -> localizacion

# Referencias
- Instalar plataforma de COnfluent mediante Docker

https://docs.confluent.io/platform/current/installation/docker/installation.html

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
