ClickHouse JDBC bridge
=====

This is a JDBC bridge for ClickHouse. It stands for proxying SQL queries to external databases from ClickHouse.
Inside ClickHouse this functionality is used in following cases:

1. Table function jdbc('jdbc connection string', 'database', 'table')
2. Table engine JDBC('jdbc connection string', 'database', 'table')
3. (Not implemented yet) Dictionary source

## How it works?
### Overview
A bridge application starts, and looks for jar-files into given `--driver-path` directory.
Within each JAR file found, bridge will try to find implementations of `java.sql.Driver`, and make them 
available for usage. 
After driver is registered, and server started, you may send queries from ClickHouse:
```
select * from jdbc('jdbc:mysql://host.port/?user=user&password=passord', 'schema', 'table')
```
for convenience reasons, `jdbc:` part of connection string can be ommitted:
```
select * from jdbc('mysql://host.port/?user=user&password=passord', 'schema', 'table')
```

#### Aliasing
As we can see, exposing credentials in query is not very secure. 

To hide user/password, and/or to setup particular settings for JDBC driver, there is an "alias" or "named connection" system.
You may create a file, containing aliases for particular DSN for connections:

 ```properties
# An example file with connections
# Format:
# connection.$alias=[jdbc:]$DSN
connection.mysql-localhost=mysql://localhost:3306/?user=root&password=root&useSSL=false
 ```
E.g. each connection alias starts with `connection.` keyword, then goes alias value, then - connection string.
When applied, you may use the following from ClickHouse:
```roomsql
SELECT * FROM jdbc('alias://mysql-localhost', 'schema', 'table')
 ```
 
### Data types notes
Currently, bridge is able to map a limited subset of JDBC java.sql.Types into ClickHouse data types.
All the types supports nullability. Native database unsigned types are not supported yet. E.g. they would be transformed to signed ClickHouse data types.
Table of conversion:

| JDBC data type | ClickHouse data type |
|----------------|----------------------|
| TINYINT        | Int8 |
| SMALLINT       | Int16 |
| INTEGER        | UInt32 |
| BIGINT         | UInt64 |
| FLOAT          | Float32 |
| REAL           | Float32 |
| DOUBLE         | Float64 |
| TIMESTAMP      | DateTime |
| TIME           | DateTime |
| DATE           | Date |
| BIT            | UInt8 |
| BOOLEAN        | UInt8 |

## Building bridge
Prerequisites:
1. JDK 1.8+
2. Maven

Building script:
```
mvn clean package
```
During the build, tests would be launched. If you have `clickhouse-local` binary installed, you may specify it's full path, 
and then integration tests would be launched:
```
mvn clean package -Dclickhouse.local.bin=/usr/bin/clickhouse-local
```
Once build finished, you'll find `target/jdbc.bridge-1.0.jar`, ready to work.

## Running bridge
```
java -jar jdbc.bridge-1.0.jar --help

Usage: /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java -jar /home/krash/dev/java/clickhouse-jdbc-bridge/target/jdbc.bridge-1.0.jar [options]
  Options:
    --connection-file
      File, containing specifications for connections
    --driver-path
      Path to directory, containing JDBC drivers
    --err-log-path
      Where to redirect STDERR
    --help
      Show help message
    --http-port
      Port to listen on
      Default: 9019
    --listen-host
      Host to listen on
      Default: localhost
    --log-level
      Log level
      Default: DEBUG
    --log-path
      Where to write logs
```
Options meaning:

`--connection-file` a .properties file, containing so-called "aliases" for connections (see below)

`--driver-path` a path to a directory, containing jar-files with JDBC driver's implementation. Bridge will automatically 
find implementations of `java.sql.Driver` in each jar, and make it available for usage

`--err-log-path` path to file, where to redirect STDERR

`--http-port` a port to bind to

`--listen-host` a host to bind to

`--log-level` logging level (INFO, DEBUG, etc.)

`--log-path` where to write logs, produced by logging facility 


