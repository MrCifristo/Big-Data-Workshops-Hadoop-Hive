# Guía Completa de Instalación y Configuración de Hadoop y Hive en Docker

## Introducción
Esta guía detallada tiene como objetivo ayudarte a configurar un entorno funcional de Hadoop y Hive utilizando Docker. Aprenderás a preparar tu entorno, configurar servicios clave como NameNode y DataNode, y realizar operaciones básicas en HDFS y Hive.

---

## Módulo 1: Introducción a Hadoop y Hive (Standalone Setup)

### Objetivos
1. Entender el funcionamiento básico de Hadoop (HDFS y comandos esenciales).
2. Conectar Hive con Hadoop para manejar datos estructurados.
3. Configurar y validar un entorno con un único NameNode.

---

## Paso 1: Preparar el Entorno
1. **Instalar Docker y Docker Compose:** Asegúrate de tener Docker instalado en tu sistema. Puedes descargarlo desde [Docker Desktop](https://www.docker.com/products/docker-desktop/).
2. **Crear la Estructura del Proyecto:**
   ```bash
   mkdir standalone-hadoop-hive
   cd standalone-hadoop-hive
   mkdir config
   ```

---

## Paso 2: Configurar Archivos para Hadoop y Hive

### Archivo `core-site.xml`
Define el sistema de archivos predeterminado de Hadoop.
```xml
<?xml version="1.0"?>
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://namenode:9000</value>
  </property>
</configuration>
```
Guarda este archivo como `config/core-site.xml`.

### Archivo `hdfs-site.xml`
Configura las propiedades de almacenamiento de Hadoop.
```xml
<?xml version="1.0"?>
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/hadoop/dfs/name</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/hadoop/dfs/data</value>
  </property>
</configuration>
```
Guarda este archivo como `config/hdfs-site.xml`.

### Archivo `hive-site.xml`
Configura las propiedades de Hive para la conexión con el metastore.
```xml
<?xml version="1.0"?>
<configuration>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:derby:;databaseName=metastore_db;create=true</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>org.apache.derby.jdbc.EmbeddedDriver</value>
  </property>
  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
  </property>
  <property>
    <name>hive.exec.scratchdir</name>
    <value>/tmp/hive</value>
  </property>
</configuration>
```
Guarda este archivo como `config/hive-site.xml`.

---

## Paso 3: Crear el Archivo `docker-compose.yml`
Configura los servicios de Docker para NameNode, DataNode y Hive.
```yaml
version: '3.8'

services:
  namenode:
    image: apache/hadoop:3.4.1
    container_name: namenode
    hostname: namenode
    platform: linux/amd64
    ports:
      - "9870:9870" # Web UI para NameNode
      - "9000:9000" # RPC para HDFS
      - "9001:9001" # Servicio RPC para NameNode
    volumes:
      - namenode_data:/hadoop/dfs/name
      - ./config/core-site.xml:/opt/hadoop/etc/hadoop/core-site.xml
      - ./config/hdfs-site.xml:/opt/hadoop/etc/hadoop/hdfs-site.xml
    command: ["/opt/hadoop/bin/hdfs", "namenode"]

  datanode:
    image: apache/hadoop:3.4.1
    platform: linux/amd64    
    container_name: datanode
    hostname: datanode
    depends_on:
      - namenode
    ports:
      - "9864:9864" # Web UI para DataNode
    volumes:
      - datanode_data:/hadoop/dfs/datanode
      - ./config/core-site.xml:/opt/hadoop/etc/hadoop/core-site.xml
      - ./config/hdfs-site.xml:/opt/hadoop/etc/hadoop/hdfs-site.xml
    command: ["/opt/hadoop/bin/hdfs", "datanode"]

volumes:
  namenode_data:
  datanode_data:
```

---

## Paso 4: Ejecutar el Entorno
1. Inicia los servicios definidos en `docker-compose.yml`:
   ```bash
   docker-compose up -d
   ```
2. Valida que los servicios estén activos:
   - Interfaz web del NameNode: [http://localhost:9870](http://localhost:9870).
   - Interfaz web del DataNode: [http://localhost:9864](http://localhost:9864).

---

## Paso 5: Subir los Datasets y Configurar Hive

### Subir Archivos al HDFS
1. **Crea un directorio para los datasets:**
   ```bash
   docker exec -it namenode hadoop fs -mkdir -p /datasets
   ```
2. **Sube los archivos de prueba al HDFS:**
   ```bash
   docker exec -it namenode hadoop fs -put word_count.txt /datasets/
   docker exec -it namenode hadoop fs -put sales.csv /datasets/
   docker exec -it namenode hadoop fs -put employees.csv /datasets/
   ```

### Crear Tablas en Hive
1. **Conéctate a HiveServer2:**
   ```bash
   docker exec -it hive-server beeline -u jdbc:hive2://localhost:10000
   ```
2. **Crea las tablas externas en Hive:**
   ```sql
   -- Tabla para Conteo de Palabras
   CREATE TABLE word_count (word STRING)
   ROW FORMAT DELIMITED
   FIELDS TERMINATED BY ' '
   STORED AS TEXTFILE
   LOCATION '/datasets/word_count.txt';

   -- Tabla para Ventas
   CREATE TABLE sales (id INT, product STRING, price FLOAT, quantity INT)
   ROW FORMAT DELIMITED
   FIELDS TERMINATED BY ','
   STORED AS TEXTFILE
   LOCATION '/datasets/sales.csv';

   -- Tabla para Empleados
   CREATE TABLE employees (id INT, name STRING, department STRING, salary FLOAT)
   ROW FORMAT DELIMITED
   FIELDS TERMINATED BY ','
   STORED AS TEXTFILE
   LOCATION '/datasets/employees.csv';
   ```

### Consultas de Prueba en Hive
- **Conteo de Palabras:**
  ```sql
  SELECT word, COUNT(*) AS word_count
  FROM word_count
  GROUP BY word;
  ```
- **Agrupaciones y Operaciones:**
  ```sql
  SELECT product, SUM(price * quantity) AS total_sales, COUNT(*) AS transactions
  FROM sales
  GROUP BY product;
  ```
- **Filtros:**
  ```sql
  SELECT name, department, salary
  FROM employees
  WHERE department = 'Engineering'
  AND salary > 60000;
  ```

---

## Módulo 2: Configuración de un Clúster Hadoop Multi-Node

### Objetivos
1. Configurar un clúster con un NameNode y múltiples DataNodes.
2. Verificar la redundancia de datos.
3. Ejecutar consultas distribuidas en Hive.

---

### Paso 1: Ampliar el Archivo `docker-compose.yml`

Añade más DataNodes para simular un entorno distribuido.

```yaml
  datanode2:
    image: apache/hadoop:3.4.1
    platform: linux/amd64
    container_name: datanode2
    hostname: datanode2
    depends_on:
      - namenode
    ports:
      - "9865:9864" # Web UI para DataNode 2
    volumes:
      - datanode_data2:/hadoop/dfs/data
      - ./config/core-site.xml:/opt/hadoop/etc/hadoop/core-site.xml
      - ./config/hdfs-site.xml:/opt/hadoop/etc/hadoop/hdfs-site.xml
    command: ["/opt/hadoop/bin/hdfs", "datanode"]

volumes:
  datanode_data2:
```

---

### Paso 2: Validar la Configuración del Clúster

1. **Sube un archivo al clúster:**
   ```bash
   docker exec -it namenode hadoop fs -put example.txt /
   ```
2. **Verifica la replicación del archivo:**
   ```bash
   docker exec -it namenode hadoop fs -ls /
   ```

---

### Paso 3: Consultas Distribuidas con Hive

Ejecuta las mismas consultas del módulo anterior para verificar el funcionamiento en el entorno multi-node. Asegúrate de que las tablas estén correctamente configuradas y que los datos sean accesibles en el clúster.

---

## Paso 6: Anexo: Comandos Comunes de Docker

### Validar el Estado de los Contenedores
- Lista los contenedores activos:
  ```bash
  docker ps
  ```
- Lista todos los contenedores (activos e inactivos):
  ```bash
  docker ps -a
  ```

### Revisar Logs
- Ver los logs de un contenedor específico:
  ```bash
  docker logs <nombre_contenedor>
  ```
- Ver logs en tiempo real:
  ```bash
  docker logs -f <nombre_contenedor>
  ```

### Detener y Eliminar Contenedores
- Detener todos los contenedores en ejecución:
  ```bash
  docker stop $(docker ps -q)
  ```
- Eliminar todos los contenedores:
  ```bash
  docker rm $(docker ps -aq)
  ```

### Gestionar Imágenes
- Lista todas las imágenes locales:
  ```bash
  docker images
  ```
- Eliminar imágenes no utilizadas:
  ```bash
  docker image prune
  ```

### Gestionar Volúmenes
- Lista todos los volúmenes existentes:
  ```bash
  docker volume ls
  ```
- Eliminar volúmenes no utilizados:
  ```bash
  docker volume prune
  ```

### Depuración de Redes
- Lista todas las redes creadas por Docker:
  ```bash
  docker network ls
  ```
- Inspecciona una red específica:
  ```bash
  docker network inspect <nombre_red>
  ```

---

Con esta guía detallada, estarás preparado para configurar y administrar un entorno de Hadoop y Hive en Docker, ya sea standalone o en un clúster multi-node. ¡Buena suerte y no dudes en consultar cualquier duda! 😊

