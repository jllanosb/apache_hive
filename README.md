# Instalacion de Apache Hive en Lunix

Apache Hive es un sistema de almacenamiento y gestión de datos distribuido construido sobre Apache Hadoop, diseñado para facilitar el análisis y consulta de grandes volúmenes de datos almacenados en sistemas como HDFS o Amazon S3. Utiliza un lenguaje de consultas similar a SQL llamado HiveQL, que permite a usuarios con conocimientos en SQL realizar análisis masivos sin necesidad de programar en Java. Hive traduce estas consultas en trabajos de procesamiento distribuido, como MapReduce o Apache Tez, ejecutados en clústeres Hadoop para procesar datos a escala petabyte.

## Componentes Principales de Apache Hive 

1. Metastore 
- Es el almacén central de metadatos de Hive.
- Almacena información sobre las tablas (esquema, ubicación, particiones, columnas, tipos de datos, etc.).
- Puede funcionar en modo incorporado (con Derby, solo para desarrollo) o en modo remoto (con bases de datos como MySQL, PostgreSQL, etc., para entornos de producción).
- Es accedido por el Hive Driver y otros componentes para obtener metadatos.

2. Driver (o Hive Driver) 
- Actúa como el motor principal que gestiona el ciclo de vida de una consulta HiveQL.
- Coordina los componentes necesarios para compilar, optimizar y ejecutar consultas.
- Incluye subcomponentes como:
    Compiler: Traduce HiveQL en un plan de ejecución (DAG).
    Optimizer: Optimiza el plan de ejecución (por ejemplo, eliminando redundancias, reordenando operaciones).
    Executor: Coordina la ejecución del plan optimizado en el motor de procesamiento subyacente (MapReduce, Tez o Spark).

3. Compiler 
- Recibe la consulta HiveQL y la convierte en un plan de ejecución lógico y luego en uno físico.
- Genera un DAG (Directed Acyclic Graph) de tareas que representan las operaciones a realizar (por ejemplo, escaneo de tablas, joins, agregaciones).

4. Optimizer 
- Mejora el rendimiento de las consultas aplicando técnicas como:
    Predicate pushdown: Mover filtros lo más cerca posible de la fuente de datos.
    Column pruning: Seleccionar solo las columnas necesarias.
    Join reordering, map-side joins, etc.

5. Executor 
- Ejecuta el plan físico generado por el optimizador.
- Interactúa con el framework de procesamiento subyacente (MapReduce, Tez o Spark) para lanzar tareas en el clúster.

6. CLI / Beeline / Thrift Server 
- CLI (Command Line Interface): Interfaz antigua para interactuar con Hive (en desuso en versiones recientes).
- Beeline: Cliente basado en JDBC más moderno y recomendado.
- HiveServer2 (HS2): Servicio que permite a múltiples clientes (como Beeline, herramientas BI, etc.) conectarse a Hive mediante Thrift. Soporta autenticación, concurrencia y sesiones.

7. SerDe (Serializer/Deserializer) 
- Componente clave que define cómo los datos se serializan al almacenarse y deserializan al leerse.
- Permite a Hive trabajar con formatos de datos personalizados (CSV, JSON, Avro, Parquet, ORC, etc.).
- Cada tabla puede tener su propio SerDe.

8. Input/Output Formats 
- Definen cómo se leen y escriben los datos en el sistema de archivos (por ejemplo, TextInputFormat, SequenceFileInputFormat, etc.).
- Trabajan junto con los SerDes para manejar el formato físico de los datos.

9. UDF (User-Defined Functions) 
- Permiten a los usuarios extender HiveQL con funciones personalizadas.
- Tipos comunes:
    UDF: Operan sobre una sola fila (ej: UPPER()).
    UDAF (Aggregate): Operan sobre múltiples filas (ej: SUM(), AVG()).
    UDTF (Table-Generating): Generan múltiples filas a partir de una entrada (ej: explode()).

# ✅ Paso 0. Requisitos previos 

Antes de instalar Hive, asegúrate de tener lo siguiente: 

1. Sistema operativo: Ubuntu 24.04 LTS
2. Java JDK 8 o 11 (Hive no es compatible con Java 17+ en versiones antiguas; para Hive 3.x se recomienda Java 8 o 11)
3. Apache Hadoop instalado y funcionando (versión 3.x recomendada)

Visitar el repositorio de [Instalacion Apache Hadoop](https://github.com/jllanosb/apache-hadoop)

4. Usuario sin privilegios de root con permisos sudo

# 🔧 Paso 1. Verificar requisitos
## Verificar Java
```bash
java -version
```
Si no tienes Java instalado:
```bash
sudo apt update
sudo apt install openjdk-11-jdk -y
```
Configura la variable JAVA_HOME si no está definida:
```bash
echo 'export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```
## Verificar Hadoop
Asegúrate de que Hadoop esté instalado y corriendo:
```bash
hadoop version
```
Y que los servicios estén activos:

```bash
start-dfs.sh
start-yarn.sh
```
```bash
jps
```
Deberías ver NameNode, DataNode, ResourceManager, NodeManager, etc.

# Paso 3. Descarga e instalación de Apache Hive junto a Apache Hadoop en  Ubuntu 24.04
## Autenticarse con el usuario creado para Apache Hadoop

```bash
su -u hadoop
```
Ingresar su contraseña

## Descargar Apache Hive
Ve al sitio oficial de [Apache Hive](https://hive.apache.org/general/downloads/) y copia el enlace de la versión estable más reciente compatible con tu Hadoop.

Moverse a la raiz
```bash
cd ~
```
Descargar Ve al sitio oficial de Apache Hive  y copia el enlace de la versión estable más reciente compatible con tu Hadoop.

```bash
wget https://archive.apache.org/dist/hive/hive-4.0.1/apache-hive-4.0.1-bin.tar.gz
```

Descomprime el archivo:

```bash
tar -xzvf apache-hive-4.0.1-bin.tar.gz
mv apache-hive-4.0.1-bin hive
```

# ⚙️ Paso 4. Configurar las variables de entorno
Edita tu archivo ~/.bashrc:

```bash
sudo nano ~/.bashrc
```
Agrega al final:

```bash
# Hive
export HIVE_HOME=~/hive
export PATH=$PATH:$HIVE_HOME/bin
export CLASSPATH=$CLASSPATH:/usr/lib/jvm/java-11-openjdk-amd64/lib/tools.jar
```
Guarda y recarga:
```bash
source ~/.bashrc
```

Verifica:

```bash
echo $HIVE_HOME
```

# 🗃️ Paso 5. Configurar Hive
## Crear directorio de trabajo en HDFS 

Hive necesita un directorio en HDFS para almacenar metadatos y datos: 
```bash
hdfs dfs -mkdir -p /user/hive/warehouse
hdfs dfs -mkdir -p /tmp
hdfs dfs -chmod g+w /user/hive/warehouse
hdfs dfs -chmod g+w /tmp
```

## Configurar hive-site.xml
Crea el archivo de configuración:

```bash
cd $HIVE_HOME/conf
cp hive-default.xml.template hive-site.xml
```
Edita hive-site.xml:

```bash
sudo nano hive-site.xml
```

Busca y modifica (o agrega) las siguientes propiedades dentro de < configuration > ... < /configuration > :
```bash
<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:derby:;databaseName=metastore_db;create=true</value>
  <description>Ubicación de la base de datos Derby para el metastore</description>
</property>

<property>
  <name>hive.metastore.warehouse.dir</name>
  <value>/user/hive/warehouse</value>
  <description>Ubicación del warehouse en HDFS</description>
</property>

<property>
  <name>hive.metastore.uris</name>
  <value/>
  <description>URI del metastore (vacío para modo embebido)</description>
</property>
```
Agrega al final de < configuration > ... < /configuration > :
```bash
<property>
  <name>system:java.io.tmpdir</name>
  <value>/tmp/hive-java</value>
</property>

<property>
  <name>system:user.name</name>
  <value>${user.name}</value>
</property>
```
## Crear directorio temporal local
Antes de usar Hive por primera vez, inicializa el metastore:

```bash
mkdir -p /tmp/hive-java
```
# 🐝 Paso 6: Inicializar Hive
Antes de usar Hive por primera vez, inicializa el metastore:
```bash
schematool -initSchema -dbType derby
```
# ▶️ Paso 7: Iniciar Hive CLI
Ejecuta Hive:
```bash
hive
```

Deberías ver algo como: 
hive>

Prueba un comando básico:

```bash
SHOW DATABASES;
```
###### Nota. Si ves default, ¡todo está funcionando!

# 🧪 Paso 8: (Opcional) Usar Beeline en lugar del CLI obsoleto

El CLI (hive) está obsoleto. Se recomienda usar Beeline con HiveServer2. 
Iniciar HiveServer2: 
```bash
hiveserver2
```
En otra terminal:

```bash
beeline -u jdbc:hive2://localhost:10000
```
Ingresa con tu usuario del sistema (sin contraseña en modo local).

### 🧹 Limpieza y consejos
Derby solo permite una sesión: si ves errores como Another instance of Derby..., elimina el lock:
```bash
rm -rf metastore_db/*.lck
```
Para producción, configura Hive con MySQL or PostgreSQL como metastore. 

Asegúrate de que Hadoop esté corriendo antes de iniciar Hive. 

# Configuracion Apache Hive en servidor VPS IP_PUBLICA

### ✅ Requisitos previos 

- Hadoop ya está instalado y corriendo en < IP_PUBLICA >.
- Hive ya está instalado en el mismo servidor (siguiendo buenas prácticas, con el mismo usuario que Hadoop, ej. hadoop).
- El firewall del servidor permite el puerto 10000 (o el que uses para HiveServer2).
- Tienes acceso SSH al servidor como el usuario que administra Hadoop/Hive.

## 🔧 Paso 1: Configurar hive-site.xml para modo servidor

Inicia sesión en tu servidor o en la terminal de windows:

```bash
ssh hadoop@< IP_PUBLICA >
```
Ve al directorio de configuración de Hive:

```bash
cd $HIVE_HOME/conf
```

Edita (o crea) el archivo hive-site.xml:
```bash
sudo nano hive-site.xml
```
Asegúrate de incluir al menos las siguientes propiedades dentro de < configuration > ... < /configuration >:
```bash
  <!-- Directorio del warehouse en HDFS -->
  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
    <description>Ubicación del warehouse en HDFS</description>
  </property>

  <!-- Metastore embebido con Derby (solo para pruebas) -->
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:derby:;databaseName=metastore_db;create=true</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>org.apache.derby.jdbc.EmbeddedDriver</value>
  </property>
```
Asegúrate de incluir al menos las siguientes propiedades dentro de < configuration > ... < /configuration >
```bash
  <!-- HiveServer2: escuchar en todas las interfaces -->
  <property>
    <name>hive.server2.thrift.bind.host</name>
    <value>0.0.0.0</value>
    <description>Permitir conexiones desde cualquier IP</description>
  </property>

  <property>
    <name>hive.server2.thrift.port</name>
    <value>10000</value>
  </property>

  <!-- Opcional: desactivar autenticación simple (solo para desarrollo) -->
  <property>
    <name>hive.server2.authentication</name>
    <value>NONE</value>
  </property>

  <!-- Directorio temporal local (evita errores de /tmp) -->
  <property>
    <name>hive.exec.scratchdir</name>
    <value>/tmp/hive</value>
  </property>
```
Crea el directorio temporal:

```bash
mkdir -p /tmp/hive
chmod 777 /tmp/hive  # Solo en desarrollo
```

## 🔁 Paso 2: Inicializar el esquema del metastore (si no lo has hecho)

```bash
schematool -initSchema -dbType derby
```
## ▶️ Paso 3: Iniciar HiveServer2

Ejecuta HiveServer2 en segundo plano (o en una sesión screen/tmux):

```bash
hiveserver2 &
```
O para ver logs en tiempo real:
```bash
nohup hiveserver2 > hiveserver2.log 2>&1 &
```

Verifica que esté escuchando en el puerto 10000:
```bash
netstat -tuln | grep 10000
# Deberías ver: tcp6 0 0 :::10000 :::* LISTEN
```

## 🔥 Paso 4: Abrir el puerto 10000 (Si y solo tienes activado el Firewall)

Si usas ufw (firewall de Ubuntu):

```bash
sudo ufw allow 10000/tcp
sudo ufw reload
```

Si usas iptables o un firewall de red (como en la nube), asegúrate de que el puerto 10000 esté abierto en el grupo de seguridad del servidor (por ejemplo, en AWS, GCP, etc.).

## 💻 Paso 5: Conectarte desde una máquina remota

Desde tu computadora local (no el servidor), puedes usar Beeline si tienes Hive instalado, o cualquier cliente JDBC. 
### Opción A: Usar Beeline (desde otra máquina con Hive) 
```bash
beeline -u "jdbc:hive2://<IP_PUBLICA>:10000"
```
Deberías ver:
```bash
Connecting to jdbc:hive2://<IP_PUBLICA>:10000
Connected to: Apache Hive (version ...)
Driver: Hive JDBC (version ...)
Transaction isolation: TRANSACTION_REPEATABLE_READ
0: jdbc:hive2://161.132.54.162:10000>
```
### Opción B: Usar DBeaver, SQuirreL SQL, etc.

- Driver: Hive JDBC (hive-jdbc-*.jar)
- URL: jdbc:hive2://< IP_PUBLICA >:10000
- Usuario/contraseña: dejar en blanco (si usas authentication=NONE)

# 🧪 Prueba rápida 

Desde tu cliente remoto: 
```bash
SHOW DATABASES;
CREATE DATABASE test_remote;
USE test_remote;
CREATE TABLE prueba (id INT, nombre STRING);
INSERT INTO prueba VALUES (1, 'Hola desde remoto');
SELECT * FROM prueba;
```
Si funciona, ¡todo está listo!

## 🛡️ Recomendaciones de seguridad (para producción)
1. No uses Derby en producción: cambia a MySQL o PostgreSQL como metastore.
2. Habilita autenticación:
Edita el archivo hive-site.xml:
```bash
sudo nano hive-site.xml
```
Agregar o modifica:
```bash
<property>
  <name>hive.server2.authentication</name>
  <value>PAM</value> <!-- o LDAP, Kerberos -->
</property>
```
3. Restringe el acceso por IP en el firewall.
4. Usa TLS si envías datos sensibles.

# Autor
 ® Jaime Llanos Bardales

## Conclusión 

Apache Hive simplifica el análisis de big data al permitir el uso de SQL en entornos Hadoop. Sus componentes trabajan en conjunto para traducir consultas declarativas en tareas distribuidas eficientes, mientras que su arquitectura modular permite flexibilidad en formatos de datos, motores de ejecución y extensibilidad mediante UDFs. 