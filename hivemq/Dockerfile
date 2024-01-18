# Utiliza la imagen base de HiveMQ
FROM hivemq/hivemq4:4.24.0

# Elimina el archivo DISABLED
RUN rm /opt/hivemq-4.24.0/extensions/hivemq-kafka-extension/DISABLED

# Copia el archivo kafka-configuration.xml al directorio de trabajo del contenedor
COPY kafka-configuration.xml /opt/hivemq-4.24.0/extensions/hivemq-kafka-extension

