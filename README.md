# Módulo 1: Introducción a Hadoop y Hive (Standalone Setup)

## Objetivos
1. Demostrar cómo funciona Hadoop (HDFS y comandos básicos).
2. Demostrar cómo funciona Hive y cómo se conecta a Hadoop.
3. Mostrar cómo funciona todo con un único NameNode.

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
```yaml
version: '3.8'

services:
  namenode:
    image: apache/hadoop:3.4.1
    container_name: namenode
    hostname: namenode
    ports:
      - "9870:9870" # Web UI for NameNode
      - "9000:9000" # RPC for HDFS
    volumes:
      - namenode_data:/hadoop/dfs/name
      - ./config/core-site.xml:/etc/hadoop/core-site.xml
      - ./config/hdfs-site.xml:/etc/hadoop/hdfs-site.xml
    command: ["namenode"]

  datanode:
    image: apache/hadoop:3.4.1
    container_name: datanode
    hostname: datanode
    depends_on:
      - namenode
    ports:
      - "9864:9864" # Web UI for DataNode
    volumes:
      - datanode_data:/hadoop/dfs/data
      - ./config/core-site.xml:/etc/hadoop/core-site.xml
      - ./config/hdfs-site.xml:/etc/hadoop/hdfs-site.xml
    command: ["datanode"]

  hive:
    image: apache/hive:4.0.0
    container_name: hive-server
    hostname: hive-server
    depends_on:
      - namenode
    ports:
      - "10000:10000" # HiveServer2 Thrift port
    environment:
      - HADOOP_HOME=/opt/hadoop
      - HIVE_HOME=/opt/hive
    volumes:
      - ./config/hive-site.xml:/opt/hive/conf/hive-site.xml
    command: ["hiveserver2"]

volumes:
  namenode_data:
  datanode_data:
```
Guarda este archivo en la raíz como `docker-compose.yml`.

---

## Paso 4: Ejecutar el Entorno
1. Ejecuta el siguiente comando:
   ```bash
   docker-compose up -d
   ```
2. Valida:
   - NameNode UI: [http://localhost:9870](http://localhost:9870).
   - HiveServer2 en el puerto `10000`.

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

# Módulo 2: Configuración de un Clúster Hadoop Multi-Node

## Objetivos
1. Configurar un clúster con un NameNode y múltiples DataNodes.
2. Verificar la redundancia de datos.
3. Usar Hive en un entorno distribuido.

---

## Paso 1: Adaptar el Archivo `docker-compose.yml`
Añade más DataNodes:
```yaml
  datanode2:
    image: apache/hadoop:3.4.1
    container_name: datanode2
    hostname: datanode2
    depends_on:
      - namenode
    ports:
      - "9865:9864"
    volumes:
      - datanode_data2:/hadoop/dfs/data
      - ./config/core-site.xml:/etc/hadoop/core-site.xml
      - ./config/hdfs-site.xml:/etc/hadoop/hdfs-site.xml
    command: ["datanode"]

volumes:
  datanode_data2:
```

---

## Paso 2: Probar el Clúster
1. **Subir un archivo al clúster:**
   ```bash
   docker exec -it namenode hadoop fs -put example.txt /
   ```
2. **Verificar replicación:**
   ```bash
   docker exec -it namenode hadoop fs -ls / | grep example.txt
   ```

## Paso 3: Consultas Distribuidas con Hive
Realiza las mismas consultas del módulo 1, ahora sobre el clúster configurado.

---

## Conclusión
Has configurado y probado Hadoop y Hive en configuraciones standalone y multi-node. Esto te da una visión completa de cómo funcionan estas tecnologías.
