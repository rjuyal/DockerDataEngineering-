# Docker-Bigdata
Some of the sections(services) in docker compose are commented. In order to use these services, please uncomment the lines.
   ## HADOOP
   ## HIVE
   ## SPARK (STANDALONE)
   ## SQOOP
   ## NIFI
   ## KAFKA
   ## ELASTIC-SEARCH

## docker-compose.yml
Creating a bigdata docker environment is one step ahead, run the docker compose file as docker-compose up -- build. This will brings up all necessary docker containers and you can start using it. The initial build will take considerably long time depends on your internet speed.

Log in to the cluster using Putty terminal. Putty configurations:
             
              Host: localhost
              Port: 1431
              Protocol: ssh
              Username: root
              Password: root
              
This docker compose contains 3 services

### mssql-db:

It is the database which acts as meta-store for Hive. Since the port of the database (1433) is exposed to the host machine port (1430),
you can connect via SQL Server Management Studio from your host machine.

```
              Host: localhost,1430
              Username: sa
              Password: password!1
```
also, you can use this database in any of the new services you want to attach in docker compose.
(In order to access the database from other services, use machine name: mssql-db , port:1433 and above username and password )
For example, see the service mssql-db-data-load. Open the file Docker-Bigdata/mssql-db-data-load/run-data-load.sh, you can see the machine
name used to connect to the database is mssql-db and port is 1433  (not localhost/1430)

Note: This service does not use a separate docker file as like other services below, since the image microsoft/mssql-server-linux:latest
contain all necessary steps to bring up the mssql database.

### mssql-db-data-load :

The Hive metastore requires a database in mssql-db before we start hive meta-store as part of below bigdata-cluster service.
This service is to create a simple DB named 'metastore'. See the docker file used for this service. It uses microsoft/mssql-server-linux:latest as the base image and copied everything from its context path. the command argument in docker compose
file tells the command to be executed in its beginning. 

``` 
  command: sh -c './wait-for-it.sh mssql-db:1433 --timeout=0 --strict -- sleep 10 && ./run-data-load.sh'
```
The script wait-for-it.sh, which we already copied as part of docker file, helps us to wait until the port 1433 is available from the mssql-db container. If the port is available, which means the database is brought up, and the script executes the command mentioned at the end.
``` 
  Sleep 10 && ./run-data-load.sh
```
This is to wait for another 10 seconds(for a safer side) and execute the script run-data-load.sh
This script contain the command 
```
  /opt/mssql-tools/bin/sqlcmd -S mssql-db,1433 -U sa -P 'password!1' -i create-hive-metastore-db.sql;
```
which plays the sql scripit create-hive-metastore-db.sql which contain the sql query 'CREATE DATABASE metastore'

Note:  This service uses the same image as of mssql-db, but does not start another database. This is achieved when we explicitly 
mentioned a command in the docker compose file which overrides the default command in this image to start the database.

### bigdata-cluster :

The big data service which consists of Hadoop, Hive, Spark, Sqoop. The ssh port of this service is exposed to 1431 of the host machine,
you can use the host's putty to connect to the machine once it is up.

Putty configurations:
 
```
              Host: localhost
              Port: 1431
              Protocol: ssh
              Username: root
              Password: root
```
Since this the heart of this docker project, a line by line explanation of docker file will be essential for a better understanding. 

The files in the folder Docker-Bigdata/bigdata-cluster/config/ contain all pre-configured configurations for different software that 
we install via docker file.


## Docker file : Docker-Bigdata/bigdata-cluster/Dockerfile

**Line 1**: Here we use a Ubuntu 16.O4 image as a base.

### Installing Java
**Line 3-9**: Installs java using apt-get installer that comes along with Ubuntu.  
Line 6 and 7 sets JAVA_HOME and PATH. since we used ENV command of docker, this variable sets only when we get into the docker container via "docker exec -it" command. So in order to get these environment variables available to ssh logins sessions (this is what Hadoop daemons uses internally to communicate), it is required to keep it in bash_profile. Line 8 and 9 do the same. The same approach is taken in many places in this docker file.

### Installing rsync,vim,sudo,OpenSSH-server,ssh
**Line 14**: Installs rsync. I am not sure whether this is a must for Hadoop installation. ( I will check and update )  
**Line 15**: Installs vim and sudo. Vim is the file editor. I am not sure whether sudo is a must for Hadoop installation. 

**Line 16-22**: Installs open ssh server and client, and also perform the steps for enabling a passwordless login. the EXPOSE 22 is required to let the container allow its port 20 to be accessible from other containers in the same network.

The port mapping in docker composer is different that EXPOSE. the port mapping let the docker containers port to be accessed via hosts port. 
Line 29: 30 in docker composer.
````
    ports:
    - "1431:22"
````
This tells the docker that any request to the host machine port 1431 should be forwarded to port 22 in docker container which is in a totally different network. this is the reason why we are able to connect to the big data cluster container via a putty client in the host machine using the port 1431. ( see the section bigdata-cluster ).

### Set root user a password
**Line 23**: sets a password 'root' for the user 'root'  
**Line 24**: This is a configuration change which will allow us to login to the bigdata-cluster container using the root user(root).
              Otherwise, we have to create a new user.

### Installs Hadoop
**Line 27** : Create a folder hadoop  
**Line 28** : Download Hadoop binary distribution file  
**Line 29** : Extract the binary dstribution file  
**Line 30-34** : Copies pre-configured files from host machine to container's specific folders.
                 Compare the conf files which come along with the binaries with the files I have modified to get an idea on the basic changes done to let Hadoop run.  
**Line 35-36-37-38**  Exports HADOOP_HOME and PATH. 
**Line 39**  Formating hadoop namenode
The startup command for hadoop is mentioned in start-ecosystem.sh file
### Installs hive, spark, sqoop
Many commands for installation of hive,sqoop,spark . etc are similar. Skipping those common commands from explaining.
Note: We are not installing the spark daemons as a running process. Spark's local mode( library mode) is enough for this development setup.
**Line 46**: Copies the mssql driver to the hive's library folder. We are using the mssql for hive meta-store and the meta-store initialization requires the specific database driver.  
**Line 47**: During hive meta-store initialization ( we will come to that later), the script hive-schema-1.2.0.mssql.sql is used.
This script contains some insert statement which will cause duplicate records in meta store DB during the second start of bigdata-cluster service. in order to avoid that I have added a 'delete where' before the insert. The modified file is copied to the directory and it will override the default script as part of the hive binary.

Example Line :  906 in hive-schema-1.2.0.mssql.sql
```
DELETE FROM NEXT_COMPACTION_QUEUE_ID WHERE NCQ_NEXT=1;
INSERT INTO NEXT_COMPACTION_QUEUE_ID VALUES(1);
````
The startup command for hive is mentioned in start-ecosystem.sh file
**Line 54**: Here we are installing Scala, The spark-cli requires Scala.  

**Line 71**: Copies the mssql driver to the Sqoop 's library folder. as Sqoop is the data export tool, Sqoop required the target database driver available as part of its libraries. 

**Line 78**: Installing Nifi, the startup command is mentioned in start-ecosystem.sh file
**Line 84**: Installing kafka, the startup command is mentioned in start-ecosystem.sh file

**Line 93**: Installing Elastic search. Elastic search is not allowed to run as root user for security reasons.
 A new user names 'elastic' is created and provided sudo access to it.
 **Line 96-97**:  Downloading tarball and extraction as elastic user.
 **Line 98-101**: Exporting environemnt variables
 
 **Line 104**: Copying ~/.bash_profile ( this file exists under root users home folder (~) ) to a common profile location( /etc/profile.d/bash_profile.sh)  as .sh file. All files under this folder is beings sourced by any user login to the system.
 This is to avail the env setting to the new user 'elastic'.
 
### Starting Ecosystem Hadoop,Hive .. etc

**Line 106-116** Copying the scripts which help us to start complete ecosystem, start each softwares individually and stop in similar fashions.

The command argument provided in the docker compose file, invoke the script start-ecosystem.sh as part of container initialization
Line 26 in docker compose :
```
    command: sh -c './start-ecosystem.sh'
``` 
 Below is the script content and it is self-explanatory
 
 ```
#!/bin/bash

# Start SSH server
service ssh start

# Start dfs  
$HADOOP_HOME/sbin/start-dfs.sh

# Start yarn  
$HADOOP_HOME/sbin/start-yarn.sh

#initialize metastore schema for hive and start
$HIVE_HOME/bin/schematool -dbType mssql -initSchema
hive --service metastore &
echo $! > hive-metsatore-pid.txt
hive --service hiveserver2 &
echo $! > hive-server2-pid.txt

# Start nifi
sudo service nifi start

# Start Kafka Zookeeper
$KAFKA_HOME/bin/zookeeper-server-start.sh $KAFKA_HOME/config/zookeeper.properties &

# Start Kafka server
$KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties &

# Start elastic search ( Elastic search can't be started as root user )
sudo -u elastic $ELASTIC_SEARCH_HOME/bin/elasticsearch &
echo $! > elsaticsearch-pid.txt

# To keep container running
tail -f /dev/null 

 ```

The last line in the script is a little tricky.  **tail -f /dev/null** , It will allow the container to keep running.

### Conclusion :

Creating a development environment for big data is super easy with this dockerization project. Now we can concentrate only on the installation of new components, rather than setting up the basic building blocks such as Database,SSL login, HDFS, etc. 
                 

