# Spark Data Streaming con MongoDB

## ÍNDICE
1. [Introducción](#introducción)
2. [Objetivo](#objetivo)
3. [Spark Streaming](#spark-streaming)
4. [Entorno de Trabajo](#entorno-de-trabajo)
   - [Alojamiento de los Archivos CSV](#alojamiento-de-los-archivos-csv)
   - [Cluster Spark](#cluster-spark)
   - [MongoDB Atlas](#mongodb-atlas)
5. [Implementación](#implementación)
   - [Sesión de Spark](#sesión-de-spark)
   - [Definición del Esquema](#definición-del-esquema)
   - [Lectura de Datos en Streaming](#lectura-de-datos-en-streaming)
   - [Verificación del Streaming](#verificación-del-streaming)
   - [Salida de Datos en la Consola](#salida-de-datos-en-la-consola)
   - [Agregaciones en Spark](#agregaciones-en-spark)
   - [Consultas SQL en Datos de Streaming](#consultas-sql-en-datos-de-streaming)
   - [Escritura de Datos en MongoDB](#escritura-de-datos-en-mongodb)
6. [Conclusión](#conclusión)

## Introducción
En este trabajo discutiremos el streaming de datos con Apache Spark en Python proporcionando ejemplos de código y mostrando cómo persistir estos datos en MongoDB.

Apache Spark ofrece una API de streaming flexible que admite varias fuentes de datos. Este enfoque permite la tramitación casi en tiempo real de grandes volúmenes de datos superando las limitaciones de latencia asociadas con Hadoop en el procesamiento por lotes.

## Objetivo
En este ejercicio aprenderemos a:
- Transmitir un archivo CSV utilizando Spark SQL.
- Crear una vista temporal de la tabla para ejecutar consultas SQL.
- Utilizar modos de escritura en streaming tanto en la consola como en MongoDB.

## Spark Streaming
Spark Streaming proporciona un sistema de procesamiento por lotes altamente escalable eficiente y tolerante a fallos. Utiliza una arquitectura de "Discretized Streams" (DStreams) para el procesamiento de datos en tiempo real discretizando los datos en pequeños lotes que se procesan secuencialmente.

### Características de Spark Streaming
- Balanceo de carga dinámica
- Tolerancia a fallos
- Soporte para análisis avanzados y MLlib
- Alto rendimiento

## Entorno de Trabajo

### Alojamiento de los Archivos CSV
Utilizamos una carpeta alojada en el disco duro del equipo físico: `/C/MISDATOS/CSV` para alojar los archivos CSV que se sincronizan con la colección "people" de la base de datos "iabd" en MongoDB Atlas.

### Cluster Spark
Creamos un cluster de Apache Spark utilizando contenedores Docker. A continuación se muestra el archivo `docker-compose.yml` utilizado para configurar el cluster:

```yaml
version: '3.1'
networks:
  spark-net:
    driver: bridge

services:
  spark-master:
    image: imagen-spark-profesor-master:latest
    container_name: spark-master
    hostname: spark-master
    ports:
      - "8080:8080"  # Puerto de la UI del Master
      - "7077:7077"  # Puerto del Master
    volumes:
      - /c/misdatos:/home  # Directorio para los scripts
      - /c/misdatos/csv:/home/csv  # Directorio para los archivos *.csv
    networks:
      - spark-net

  spark-worker1:
    image: imagen-spark-profesor-worker:latest
    container_name: spark-worker1
    hostname: spark-worker1
    environment:
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_CORES=1
      - SPARK_WORKER_MEMORY=2G
      - SPARK_DRIVER_MEMORY=2G
      - SPARK_EXECUTOR_MEMORY=2G
    depends_on:
      - spark-master
    volumes:
      - /c/misdatos:/home
      - /c/misdatos/csv:/home/csv
    networks:
      - spark-net

  spark-worker2:
    image: imagen-spark-profesor-worker:latest
    container_name: spark-worker2
    hostname: spark-worker2
    environment:
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_CORES=1
      - SPARK_WORKER_MEMORY=2G
      - SPARK_DRIVER_MEMORY=2G
      - SPARK_EXECUTOR_MEMORY=2G
    depends_on:
      - spark-master
    volumes:
      - /c/misdatos:/home
      - /c/misdatos/csv:/home/csv
    networks:
      - spark-net
```

### MongoDB Atlas
Utilizamos una cuenta gratuita de MongoDB Atlas con tres nodos (un maestro y dos esclavos). En MongoDB Atlas tenemos una base de datos llamada "iabd" y dentro de esta una colección llamada "people".

## Implementación

### Sesión de Spark
Primero creamos una sesión de Spark para definir dónde está corriendo nuestro nodo master y cuántos núcleos va a usar.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.master("spark://spark-master:7077").appName("MongoDBAtlasConnection").getOrCreate()
```

### Definición del Esquema
Creamos un esquema para los datos que serán transmitidos usando Spark SQL API.

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

schemaIABD = StructType([
    StructField("edad", IntegerType(), True),
    StructField("nombre", StringType(), True),
    StructField("profesion", StringType(), True)
])
```

### Lectura de Datos en Streaming
Leemos los datos en streaming desde la carpeta donde se encuentran los archivos CSV.

```python
df = spark.readStream.schema(schemaIABD).option("maxFilesPerTrigger", 1).csv("/home/csv", header=True)
```

### Verificación del Streaming
Verificamos si los datos están siendo transmitidos.

```python
print(df.isStreaming)
```

### Salida de Datos en la Consola
Escribimos los datos en la consola para verificar que el streaming funcione correctamente.

```python
df.writeStream.format("console").outputMode("append").start().awaitTermination()
```

### Agregaciones en Spark
Realizamos una agregación sobre los datos transmitidos para contar el número de profesiones distintas.

```python
dfc = df.groupBy("profesion").count()
dfc.writeStream.outputMode("complete").format("console").start().awaitTermination()
```

### Consultas SQL en Datos de Streaming
Creamos una vista temporal para realizar consultas SQL sobre los datos de streaming.

```python
df.createOrReplaceTempView("tempdf")
dfclean = spark.sql("SELECT nombre, edad FROM tempdf WHERE profesion == 'Profesor'")
dfclean.writeStream.outputMode("append").format("console").start().awaitTermination()
```

### Escritura de Datos en MongoDB
Escribimos los datos de streaming en MongoDB Atlas estableciendo una conexión mientras creamos la sesión de Spark.

```python
spark = SparkSession.builder.master("spark://spark-master:7077").appName("MongoDBAtlasConnection").config("spark.mongodb.input.uri", "mongodb+srv://username:password@cluster0.mongodb.net/iabd.people").config("spark.mongodb.output.uri", "mongodb+srv://username:password@cluster0.mongodb.net/iabd.people").config("spark.jars.packages", "org.mongodb.spark:mongo-spark-connector_2.12:3.0.1").getOrCreate()

def write_row(df, batch_id):
    df.write.format("mongodb").mode("append").save()

df.writeStream.foreachBatch(write_row).start().awaitTermination()
```

## Conclusión
En este ejercicio hemos discutido la canalización de streaming de datos con Spark en Python y la gestión de la configuración al crear sesiones. Los puntos clave incluyen:
- Funciones básicas de Spark Streaming en Python.
- Transmisión de datos desde archivos locales, sockets de Internet y APIs.
- Persistencia de datos de streaming en MongoDB.

Para obtener más detalles sobre Spark Streaming se recomienda consultar la documentación oficial.
