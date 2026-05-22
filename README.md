# Replicación de sistemas heterogéneos

## Introducción

En la siguiente práctica usaremos dos máquinas con **Ubuntu Server 24.04** para trabajar con dos de bases de datos diferentes como son **MySQL** y **PostgreSQL**. Primero migraremos la base de datos `employees` desde MySQL hacia PostgreSQL usando **pgloader**. Después configuraremos **SymmetricDS** para replicar cambios entre ambos servidores.

La práctica la dividiremos en tres puntos principales, la preparación y migración inicial, replicación unidireccional (**principal-secundario**) y replicación bidireccional (**principal-principal**).

---

## Entorno que utilizaremos

Para realizar la práctica usaremos dos máquinas virtuales en Vbox con Ubuntu 24.04 Server en ambas.

Máquina con MySQL: ServidorUbuntuMysql

Nombre: Dhayan, Nombre Server: mysql, Nombre usuario: dhayan, Contraseña: D****7, IP: 192.168.1.25

Máquina con PostgreSQL: ServidorUbuntuPostgres

Nombre: Dhayan, Nombre server: postgres, Nombre usuario: dhayan, Contraseña: D****7, IP: 192.168.1.35

La máquina `ServidorUbuntuMysql` será la que tendrá originalmente la base de datos `employees`. La máquina `ServidorUbuntuPostgres` será donde migraremos los datos y donde después configuraremos la replicación.

---

# Primera parte: preparación y migración con pgloader

---

# Servidor MYSQL

## Instalación de las herramientas básicas

En primer lugar, para llevar a cabo esta práctica, tendremos que tener instaladas ciertas herramientas de descarga y descompresión para poder bajarnos la base de datos employees que queremos cargar, en mi caso, al iniciar la práctica en una máquina nueva, usaré el siguiente comando para descargar dichas herramientas:

```bash
dhayan@mysql:~$ sudo apt install -y wget unzip
```

---

## Descarga de la BBD Employees y descompresión

Seguidamente, pasaremos a la descarga del zip de la tabla employees desde el sitio oficial, para ello, haremos uso del siguiente comando:

```bash
dhayan@mysql:~$ wget -O test_db-master.zip https://github.com/datacharmer/test_db/archive/refs/heads/master.zip
```

Una vez descargado, lo descomprimimos:

```bash
dhayan@mysql:~$ unzip -o test_db-master.zip
```

Comprobamos que se ha descargado correctamente:

```bash
dhayan@mysql:~$ cd test_db-master
dhayan@mysql:~/test_db-master$ ls
Changelog                      load_dept_manager.dump  README.md
employees_partitioned_5.1.sql  load_employees.dump     sakila
employees_partitioned.sql      load_salaries1.dump     show_elapsed.sql
employees.sql                  load_salaries2.dump     sql_test.sh
images                         load_salaries3.dump     test_employees_md5.sql
load_departments.dump          load_titles.dump        test_employees_sha.sql
load_dept_emp.dump             objects.sql             test_versions.sh
```

Una vez comprobado, podemos acceder a Mysql, crear la bbdd y cargar los datos, que lo haremos de manera sencilla con la siguiente orden:

```bash
dhayan@mysql:~/test_db-master$ sudo mysql < employees.sql
```

Y nos debería devolver algo como esto, que indica que está creando la estructura de la bbdd, y la carga de las distintas tablas que contiene:

```text
INFO
CREATING DATABASE STRUCTURE
INFO
storage engine: InnoDB
INFO
LOADING departments
INFO
LOADING employees
INFO
LOADING dept_emp
INFO
LOADING dept_manager
INFO
LOADING titles
INFO
LOADING salaries
data_load_time_diff
00:00:54
```

Para terminar de comprobarlo, entramos a mysql, seguidamente en la bbdd y haremos un par de consultas:

```bash
dhayan@mysql:~$ sudo mysql -u root
```

```sql
mysql> use employees;

mysql> show tables;
+----------------------+
| Tables_in_employees  |
+----------------------+
| current_dept_emp     |
| departments          |
| dept_emp             |
| dept_emp_latest_date |
| dept_manager         |
| employees            |
| salaries             |
| titles               |
+----------------------+
8 rows in set (0,00 sec)

mysql> select * from employees limit 5;
+--------+------------+------------+-----------+--------+------------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  |
+--------+------------+------------+-----------+--------+------------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |
|  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |
|  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |
+--------+------------+------------+-----------+--------+------------+
5 rows in set (0,00 sec)
```

---

# Servidor PostgreSQL

## Preparación del servidor PostgreSQL

Ahora pasaremos a trabajar en `ServidorUbuntuPostgres`. Aquí, crearemos una base de datos vacía llamada `employees`, que será donde haremos la migración.

Entramos en PostgreSQL:

```bash
dhayan@postgres:~$ sudo -u postgres psql
```

Creamos la base de datos:

```sql
postgres=# CREATE DATABASE employees;
CREATE DATABASE
```

Creamos un usuario para trabajar con esta base de datos:

```sql
postgres=# CREATE USER dosorio WITH PASSWORD 'D***7';
CREATE ROLE
```

Le damos permisos sobre la base de datos:

```sql
postgres=# GRANT ALL PRIVILEGES ON DATABASE employees TO dosorio;
GRANT
```

También dejamos al usuario como propietario de la base de datos, porque pgloader necesitará hacer cambios durante la migración:

```sql
postgres=# ALTER DATABASE employees OWNER TO dosorio;
ALTER DATABASE
```

Salimos:

```sql
\q
```

Comprobamos que podemos conectarnos con el usuario creado:

```bash
dhayan@postgres:~$ psql -U dosorio -d employees -h 127.0.0.1
Contraseña para usuario dosorio: 
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Conexión SSL (protocolo: TLSv1.3, cifrado: TLS_AES_256_GCM_SHA384, compresión: desactivado, ALPN: postgresql)
Digite «help» para obtener ayuda.

employees=> 
```

Y efectivamente, entra sin error.

---

## Creación del usuario de migración en MySQL

Para la migración no usaremos `root`. En su lugar, crearemos un usuario específico en MySQL para que `pgloader` pueda leer la base de datos `employees`.

Entramos a MySQL:

```bash
sudo mysql
```

Y una vez dentro, procedemos a crear el usuario con todos los permisos sobre la base de datos:

```sql
mysql> CREATE USER 'Aescobar'@'%' IDENTIFIED BY 'D***7';
Query OK, 0 rows affected (0,02 sec)
mysql> GRANT ALL PRIVILEGES ON employees.* TO 'Aescobar'@'%';
Query OK, 0 rows affected (0,00 sec)
mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0,00 sec)
```

Y salimos:

```sql
mysql> EXIT;
Bye
```

## Compilar e instalar pgloader

Seguidamente, instalaremos pgloader en la misma máquina mysql.
Para ello, en lugar de usar la versión antigua del repositorio, instalaremos primero las dependencias necesarias para poder compilarlo:

```bash
dhayan@mysql:~$ sudo apt update -y
dhayan@mysql:~$ sudo apt install -y git make build-essential sbcl curl \
libsqlite3-dev libcurl4-openssl-dev libpcre3-dev libssl-dev libzip-dev
```

Una vez instaladas, procedemos a descargar y compilar el archivo usando els siguiente comando:

```bash
dhayan@mysql:~$ git clone https://github.com/dimitri/pgloader.git
Cloning into 'pgloader'...
remote: Enumerating objects: 12050, done.
remote: Counting objects: 100% (215/215), done.
remote: Compressing objects: 100% (132/132), done.
remote: Total 12050 (delta 135), reused 83 (delta 83), pack-reused 11835 (from 4)
Receiving objects: 100% (12050/12050), 20.44 MiB | 150.00 KiB/s, done.
Resolving deltas: 100% (8417/8417), done.
```
Seguidamente entramos en la carpeta de pgloader:

```bash
dhayan@mysql:~$ cd pgloader/
```

Compilamos:

```bash
dhayan@mysql:~/pgloader$ make
```

Comprobamos la versión que se nos ha instalado para verificar que es la que necesitamos con:

```bash
dhayan@mysql:~/pgloader$ ./build/bin/pgloader --version
pgloader version "3.6.d9ca38e"
compiled with SBCL 2.2.9.debian
```

La versión nos debe de salir 3.6.7 o superior, en este caso, muestra varios caracteres después del "3.6" pero significan que es la versión más reciente.

---

##  Permitir acceso remoto a MySQL

Como la migración se hará entre dos máquinas, el siguiente paso es dar acceso remoto a la máquina mysql, para ello, entraremos en el siguiente archivo de configuración:

```bash
dhayan@mysql:~$ sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Buscaremos la siguiente línea donde apunta solo al localhost:

```conf
bind-address = 127.0.0.1
```

Y la cambiaremos por lo siguiente para permitir que escuche a cualquier sitio:

```conf
bind-address = 0.0.0.0
```

Guardamos, salimos y reiniciamos el servicio:

```bash
dhayan@mysql:~$ sudo systemctl restart mysql
```

Comprobamos que funcione correctamente viendo su estado:

```bash
dhayan@mysql:~$ sudo systemctl status mysql
● mysql.service - MySQL Community Server
     Loaded: loaded (/usr/lib/systemd/system/mysql.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-02-07 20:53:32 UTC; 9s ago
```

---

## Permitir acceso remoto a PostgreSQL

Al igual que en la máquina mysql, en la máquina postgres, debemos habilitar PostgreSQL para conexiones remotas, para ello, entraremos en el archivo de configuración de postgres con:

```bash
dhayan@postgres:~$ sudo nano /etc/postgresql/18/main/postgresql.conf
```

Y una vez dentro, buscamos listen_addresses y los dejamos así, escuchando a todo:

```conf
listen_addresses = '*'
```

Seguidamente, abriremos el archivo de configuración siguiente:

```bash
dhayan@postgres:~$ sudo nano /etc/postgresql/18/main/pg_hba.conf
```

Una vez dentro añadiremos lo siguiente al final del archivo:

```conf
host    employees       dosorio         192.168.1.25/32         scram-sha-256
```

Esto permite que nuestro servidor mysql se conecte el usuario dosorio a la bbdd employees con contraseña.

Por último, guardamos, cerramos, reiniciamos y comprobamos el funcionamiento del servicio:

```bash
dhayan@postgres:~$ sudo systemctl restart postgresql
dhayan@postgres:~$ sudo systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled; preset: enabled)
     Active: active (exited) since Sat 2026-02-07 21:17:08 UTC; 30s ago
```

---

## Probar conexión desde MySQL hacia PostgreSQL

Antes de continuar con la migración, haremos una pequeña prueba para ver si tenemos conexión entre ambas máquinas de forma muy simple. Para ello, simplemente instalaremos un cliente de postgres en la máquina mysql e intentaremos que nos devuelva algún dato. Si es así, significa que hay conexión y podemos continuar con la conexión, para ello usaremos las siguientes órdenes:

```bash
dhayan@mysql:~$ sudo apt install -y postgresql-client
```

```bash
dhayan@mysql:~$ psql -h 192.168.1.35 -U dosorio -d employees -c "SELECT 1;"
```

```text
Password for user dosorio: 
 ?column? 
----------
        1
(1 row)
```

Al devolver 1, confirma que las dos máquinas se comunican correctamente.

---

# Migración MySQL a PostgreSQL con pgloader

## Primer intento de migración

Al principio intenté hacer la migración directamente, es decir, indicando en el mismo comando la base de datos origen en MySQL y la base de datos destino en PostgreSQL.

La idea era que `pgloader` se conectara a la base de datos `employees` de MySQL, leyera su estructura y sus datos, y los copiara directamente dentro de la base de datos `employees` en PostgreSQL.

```bash
dhayan@mysql:~$ ~/pgloader/build/bin/pgloader mysql://Aescobar:Dhayan07@192.168.1.25/employees \
postgresql://dosorio:Dhayan07@192.168.1.35/employees
2026-02-07T21:28:35.011999Z LOG pgloader version "3.6.d9ca38e"
2026-02-07T21:28:35.011999Z LOG Data errors in '/tmp/pgloader/'
2026-02-07T21:28:35.081997Z LOG Migrating from #<MYSQL-CONNECTION mysql://Aescobar@192.168.1.25:3306/employees {1005E1B003}>
2026-02-07T21:28:35.081997Z LOG Migrating into #<PGSQL-CONNECTION pgsql://dosorio@192.168.1.35:5432/employees {1006107313}>
2026-02-07T21:28:35.142995Z ERROR mysql: Failed to connect to mysql at "192.168.1.25" (port 3306) as user "Aescobar": MySQL Error [1045]: "Access denied for user 'Aescobar'@'192.168.1.25' (using password: YES)"
```
Como se puede apreciar, pgloader sí que llega a iniciar el proceso e intenta conectar con MySQL y PostgreSQL, pero falla al autenticarse en MySQL con el usuario Aescobar.
Esto nos está indicando que MySQL no está aceptando correctamente la conexión de este usuario. Por lo que para solucionarlo, ajustaremos el usuario de MySQL para que use mysql_native_password, que  es el método clásico y el que funciona mejor con pgloader. Para ello, editaremos el archivo de configuración:

```bash
dhayan@mysql:~$ sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Una vez dentro, en el bloque [mysqld] añadimos:

```conf
default-authentication-plugin=mysql_native_password
```

Guardamos y reiniciamos:

```bash
dhayan@mysql:~$ sudo systemctl restart mysql
```

Comprobamos que el servidor usa el método correcto:

```bash
dhayan@mysql:~$ sudo mysql -e "SHOW VARIABLES LIKE 'default_authentication_plugin';"
```

```text
+-------------------------------+-----------------------+
| Variable_name                 | Value                 |
+-------------------------------+-----------------------+
| default_authentication_plugin | mysql_native_password |
+-------------------------------+-----------------------+
```

Después, ajustamos el usuario de migración para asegurarnos de que usa ese sistema:

```bash
dhayan@mysql:~$ sudo mysql -e "
ALTER USER 'Aescobar'@'%'
IDENTIFIED WITH mysql_native_password BY 'D****';
FLUSH PRIVILEGES;"
```

Con esto, en teoría, evitamos errores de autenticación y permitimos que pgloader se conecte correctamente.

---

## Migración usando fichero `.load`

Como el primer intento daba problemas, haremos la migración usando un fichero `.load`.  
Esta forma de hacerlo es bastante más sencilla porque dejamos la configuración de la migración escrita en un archivo y luego solo ejecutamos `pgloader` sobre ese archivo.

Creamos el fichero:

```bash
dhayan@mysql:~$ nano ~/employees.load
```

Dentro añadimos lo siguiente:

```conf
LOAD DATABASE
     FROM mysql://Aescobar:D***7@192.168.1.25/employees
     INTO postgresql://dosorio:D***7@192.168.1.35/employees

SET MySQL PARAMETERS
     net_read_timeout  = '120',
     net_write_timeout = '120',
     character_set_client      = 'latin1',
     character_set_connection  = 'latin1',
     character_set_results     = 'latin1',
     collation_connection      = 'latin1_swedish_ci'

WITH include drop, create tables, create indexes, reset sequences;
```

Con este fichero le indicamos a pgloader que la base de datos origen es la de MySQL y que la base de datos destino es la de PostgreSQL.

Guardamos el fichero y ejecutamos:

```bash
dhayan@mysql:~$ pgloader ~/employees.load
```

Una vez ejecutado nos sale este error:

```
2026-02-07T22:38:54.018999Z LOG pgloader version "3.6.7~devel"
2026-02-07T22:39:12.904658Z ERROR PostgreSQL Database error 42501: debe ser dueño de la base de datos employees
QUERY: ALTER DATABASE "employees" SET search_path TO public, employees;
```

Este error indica que el usuario dosorio puede usar la base de datos, pero no es el propietario, por lo cual, pgloader no puede hacer algunos cambios necesarios en PostgreSQL.

Para solucionarlo, en la máquina PostgreSQL ejecutamos:

```bash
dhayan@postgres:~$ sudo -u postgres psql -c "ALTER DATABASE employees OWNER TO dosorio;"
```

Seguidamente, volvemos a probar la migración:

```bash
dhayan@mysql:~$ pgloader ~/employees.load
Total import time          ✓    3919015   134.9 MB         22.206s
```

Con esto, la migración de MySQL a PostgreSQL queda realizada correctamente.

## Verificación en PostgreSQL

Ahora comprobaremos que los datos realmente esten en PostgreSQL. En primer lugar, editamos el archivo pg_hba.conf para poder entrar con el usuario dosorio sin usar -h 127.0.0.1:

```bash
dhayan@postgres:~$ sudo nano /etc/postgresql/18/main/pg_hba.conf
```

Cambiamos esta línea:

```conf
local   all             all                                     peer
```

Por algo parecido a esto:

```conf
local   all             all                                     scram-sha-256
```

Reiniciamos PostgreSQL:

```bash
dhayan@postgres:~$ sudo systemctl restart postgresql
```

Ahora entramos en la base de datos:

```bash
dhayan@postgres:~$ psql -U dosorio -d employees
```

Dentro de PostgreSQL hacemos estas comprobaciones:

```sql
employees=> \dn
employees=> \dt employees.*
employees=> SELECT COUNT(*) FROM employees.employees;
count  
--------
300024
(1 fila)
```

Con esto, damos por completada con éxito la migración de los datos de una base a otra.
