---
Title: Streaming Data from MySQL into Kafka and consuming it using Python
Description: Building a CDC pipeline using Kafka
Thumbnail: "/assets/img/Pico.png"
Date: 22 February 2021
Category: Guides
Featured: "/assets/img/Pico.png"
Template: single
Purpose: pico_categories_page
---

## Introduction
TODO

## Prerequisites

To follow along, you will need:
- One Ubuntu 20.04 LTS System and a non-root user with sudo privileges.
- Zookeeper and Kafka installed. Follow the steps specified in this [article](https://otodiginet.com/software/how-to-install-apache-kafka-on-ubuntu-20-04-lts/) if you do not have them installed.
- MySQL server and a sample database.

## Step 1 - Ensure the Kafka server is running
To ensure that the server is running, check the logs for the *Kafka* service:
```sh
$ sudo systemctl status kafka
```
You will receive output like this:

[<img src="/assets/img/kafka-service.png" class="img-fluid"/>](/assets/img/kafka-service.png)

## Step 2 - Install Debezium MySQL CDC Connector
Let's download and extract the MySQL connector into one of the directories that is listed on the Connect worker's ***plugin.path*** configuration properties.

To start, download the connector from this [link](https://www.confluent.io/hub/debezium/debezium-connector-mysql) and extract it into `/usr/local/share/kafka/plugins` path. Using this path would allow us to use the path mentioned in the ***plugin.path*** configuration. 
Or, alternatively you could extract the zip file into a directory of your choice like `/home/user/Desktop/MySQLConnector`.
Then append this directory's path to plugin.path configuration.

If you've installed Kafka server using the link given in the Prerequisites section of this article, then:

```sh
$ cd /usr/local/kafka-server/config/
$ nano connect-distributed.properties
```
Otherwise, navigate to your Kafka installation directory and edit `connect-distributed.properties` file located inside config directory.
Uncomment plugin.path property
`#plugin.path=/usr/local/share/java,/usr/local/share/kafka/plugins,/opt/connectors`
Configuration will look like this:

[<img src="/assets/img/plugin_path_property.png" class="img-fluid"/>](/assets/img/plugin_path_property.png)

## Step 3 - Run Kafka Connect in a distributed mode
Now, we are all set and ready to run Kafka Connect in a distributed mode.
To run Kafka connect we need to execute its shell script located inside `bin` directory of our Kafka installation.
```sh
$ sudo ./bin/connect-distributed.sh config/connect-distributed.properties
```
Output should be like this:
[<img src="/assets/img/kafka_connect_output.png" class="img-fluid"/>](/assets/img/kafka_connect_output.png)

## Step 4 - Deploying the MySQL connector
No, we are ready to deploy the Debezium MySQL connector so that it can start monitoring the sample MySQL database(`employees`).

At this point, we are running Kafka services, a MySQL database server with a sample `employees` database (If you do not have a sample database, download it from this [link](https://github.com/datacharmer/test_db)).

### Writing the MySQL connector configuration

To monitor the sample MySQL database, we need to create a JSON configuration file.

```sh
$ cd ~
$ mkdir connectors
$ cd connectors
$ nano register-mysql-connector.json
```
Paste the below configuration in the terminal and change it accordingly.

```json
{
  "name": "employees-connector",  
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "localhost",
    "database.port": "3306",
    "database.user": "root",
    "database.password": "password",
    "database.server.id": "184054",
    "database.server.name": "localhost", 
    "database.include.list": "employees", 
    "database.history.kafka.bootstrap.servers": "localhost:9092",  
    "database.history.kafka.topic": "schema-changes.employees"
  }
}
```

### Registering a connector to monitor the `employees` database

Loading the connector configuration into Kafka connect using the REST API:
```sh
$ cat /path/to/register-mysql-connector.json | http POST http://localhost:8083/connectors/
```
`NOTE:` For the above command to work, we need to install httpie in our ubuntu server.

```sh
$ sudo apt update && sudo apt install httpie
```

Or, alternatively we can use `curl` command to submit our request to the Kafka connect REST API:

```sh
$ curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "employees-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "root", "database.password": "password", "database.server.id": "184054", "database.server.name": "localhost", "database.include.list": "employees", "database.history.kafka.bootstrap.servers": "localhost:9092", "database.history.kafka.topic": "dbhistory.employees" } }'
```

Now, we can check the registered connectors by hitting the API:
```sh
$ curl -H "Accept:application/json" localhost:8083/connectors/
```

## Step 5 - 