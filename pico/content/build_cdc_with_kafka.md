---
Title: Streaming Data from MySQL into Kafka and consuming it using Python
Description: Building a CDC pipeline using Kafka
Thumbnail: "/assets/img/Kafka_featured.png"
Date: 23 February 2021
Category: KAFKA
Featured: "/assets/img/Kafka_featured.png"
Template: single
Purpose: pico_categories_page
---

## Introduction
Recently, I was looking for ways to build a CDC (Change Data Capture) pipeline and Kafka's name popped out and why not?
Kafka is well researched and documentated streaming platform and confluent has done a pretty good job to design the architecture of source and sink connectors, just what I was looking for.

Because of its richness in connectors, then it's all just configuration that has to go right.
Although, you could easily find the docker compose file to setup this in minutes. However, our requirement was to build something simple without having to setup a Kubernetes cluster when deploying this in production.

Let's understand few terms first:
- `Connector:` The high level abstraction that coordinates data streaming by managing tasks.
- `Source:` Source connectors allows us to import data from any relational database.
- `Sink:` Sink connectors allows us to export data from any relational database.

In this article we will explore how to setup Kafka and stream MySQL events using Python.

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
```sh
#plugin.path=/usr/local/share/java,/usr/local/share/kafka/plugins,/opt/connectors
```
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
Now, we are ready to deploy the Debezium MySQL connector so that it can start monitoring the sample MySQL database(`employees`).

At this point, we are running Kafka services, a MySQL database server with a sample `employees` database (If you do not have a sample database, download it from this [link](https://github.com/datacharmer/test_db)).

### Checking the binlog setting
Before the deployment of our connector, we need to make sure binlog is set to `ON`. So that Kafka connect will create the topics and start producing them as soon as we deploy the connector.

There are a few methods by which you could verify the `binlog` setting from your MySQL command line:

- ```mysql 
SELECT @@log_bin;
```
```mysql
+-----------+
| @@log_bin |
+-----------+
|         1 |
+-----------+
1 row in set (0.02 sec)
```
- ```mysql
SHOW VARIABLES LIKE 'log_bin';
```
```mysql
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
1 row in set (0.32 sec)
```

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

## Step 5 - Consuming Kafka topic using Python
By mentioning the `employees` database in the connector configuration, Kafka has created topics corresponding to all the available tables in the `employees` database.

Topic representation: `<hostname>.<database>.<table_name>`

Example: `localhost.employees.employees`

For the purpose of this article, we will only consume one topic. Here's a sample Python code to consume `employees` table:

```python
from kafka import KafkaConsumer
import json

topic = 'localhost.employees.employees'
bootstrap_servers = 'localhost:9092'
consumer = KafkaConsumer(
    topic, bootstrap_servers=bootstrap_servers, auto_offset_reset='latest')
consumer.subscribe(topic)

for msg in consumer:
    print(msg)
```
Notice, we've set `auto_offset_reset` is equal to `latest`. We could set it to `earliest` to consume topic from the starting offset.

Open a new terminal, and use it to run the python script we've just created and let it run exponentially. This will consume as the new message comes into the topic and prints its output in the terminal.

### Updating the database and viewing the update event
We will now change one of the records in our employees table and see how the connector captures it.

1. Start a MySQL command line utility if you haven't started and run the the following statment:
```sql
use employees;
update employees set first_name="Ankit" where emp_no=499998;
```
Output should look like this:
[<img src="/assets/img/kafka_mysql_update.png" class="img-fluid"/>](/assets/img/kafka_mysql_update.png)

2. View the updated employees table:
[<img src="/assets/img/kafka_employees_update.png" class="img-fluid"/>](/assets/img/kafka_employees_update.png)

3. Switch to the terminal running Python script to see the latest captured event.
By changing a record in the `employees` table, MySQL connector generated a new event. You should see the generated event.
[<img src="/assets/img/Kafka_topic_output.png" class="img-fluid"/>](/assets/img/Kafka_topic_output.png)

### Writing the captured response to a JSON file (Bonus)
We've made few changes in our Python code to write it in a JSON file as the new event comes in. We are only writing the `payload` part to compute on the keys and values of the document.

```python
from kafka import KafkaConsumer
import json

topic = 'localhost.employees.employees'
bootstrap_servers = 'localhost:9092'
consumer = KafkaConsumer(
    topic, bootstrap_servers=bootstrap_servers, auto_offset_reset='latest')
consumer.subscribe(topic)

with open('data.json', 'w') as out_file:
    for msg in consumer:
        json_object = json.loads(msg.value.decode())
        print(json_object['payload'])
        json.dump(json_object['payload'], out_file)
```

## Conclusion
Here we've built out our CDC pipeline, streaming changes from MySQL into Kafka and from Kafka to a JSON file. Streaming to a JSON file isn't always so useful, but serves well for a simple example.
