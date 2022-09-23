# Use Case 1

All services for this demo scenario are pre-configured and supposed to be run using the provided docker compose environment.

_NOTE: Make sure you are in the `use_case_1/
` directory of this repository before running any of the following commands._

### 0. Start Docker Compose Environment

Running `docker compose up` starts the following services:

* Zookeeper
* Kafka Broker
* Kafka Connect
* MySQL
* MongoDB

### 1. Inspect original rows in `addresses` table of MySQL

`docker compose exec mysql mysql -u mysqluser -pmysqlpw -e "use inventory; SELECT * FROM addresses;"`

This should result in displaying the data rows contained in the corresponding MySQL table `inventory.addresses`

```
+----+-------------+---------------------------+------------+--------------+-------+----------+
| id | customer_id | street                    | city       | state        | zip   | type     |
+----+-------------+---------------------------+------------+--------------+-------+----------+
| 10 |        1001 | 3183 Moore Avenue         | Euless     | Texas        | 76036 | SHIPPING |
| 11 |        1001 | 2389 Hidden Valley Road   | Harrisburg | Pennsylvania | 17116 | BILLING  |
| 12 |        1002 | 281 Riverside Drive       | Augusta    | Georgia      | 30901 | BILLING  |
| 13 |        1003 | 3787 Brownton Road        | Columbus   | Mississippi  | 39701 | SHIPPING |
| 14 |        1003 | 2458 Lost Creek Road      | Bethlehem  | Pennsylvania | 18018 | SHIPPING |
| 15 |        1003 | 4800 Simpson Square       | Hillsdale  | Oklahoma     | 73743 | BILLING  |
| 16 |        1004 | 1289 University Hill Road | Canehill   | Arkansas     | 72717 | LIVING   |
+----+-------------+---------------------------+------------+--------------+-------+----------+
```

### 2. Create Debezium Source Connector

Debezium's MySQL source connector is configured together with the `CipherField` SMT to perform log-based change data capture against the MySQL table `inventory.addressess`. The table's `street` column values get encrypted due to the SMT settings.

Run the following command in your host terminal to start the debezium tools container and enter an interactive bash session.

```
docker run -it --rm \
    --network uc1-network \
    -v ${PWD}/:/home \
    debezium/tooling:1.2 \
    bash
```

_NOTE: All of the following commands are supposed to be executed in the container's bash._

[kcctl](https://github.com/kcctl/kcctl) - a CLI for Apache Kafka Connect - is used to perform any Kafka Connect related operations. First the connect cluster address is set and used as the CLI tool's context. Then the MySQL source connector is created.

```
kcctl config set-context default --cluster=http://connect:8083

kcctl apply -f /home/register_mysql_source_connector_enc.json
```

### 3. Inspecting Partially Encrypted Records

To verify the resulting records in the Kafka topic `mysqlhost.inventory.addresses` run 

`
kafkacat -b kafka:9092 -C -t mysqlhost.inventory.addresses -o beginning -q
`

This should show JSON records for all addresses with their `street` field being encrypted. Here is one example:

```json5
{"id":13,"customer_id":1003,"street":"NIczySgmNLnO04d74ex/dT0dOKn9EklFzdlwiVed5v0IBjkiFiGBRHEw5Ec4lZHOA+akHAwwsWux","city":"Columbus","state":"Mississippi","zip":"39701","type":"SHIPPING"}
```

Afterwards, hit `CTRL + C` to stop the kafkacat consumer process.

### 4. Create MonogDB Sink Connector

The MongoDB sink connector is configured together with the `CipherField` SMT to store address records as documents into a MongoDB collection `mysqlhost.inventory.addresses`. The `street` field values of the documents get decrypted due to the SMT settings.

Again kcctl is used to create this sink connector instance.

```
kcctl apply -f /home/register_mongodb_sink_connector_dec.json
```

In the container's bash type `exit` to get back to your host terminal.

### 5. Inspecting Decrypted Documents

To verify the resulting documents in the MongoDB collection `mysqlhost.inventory.addresses` run 

```
docker run -it --rm \
        --network uc1-network \
        mongo:5.0.6 \
        mongo mongodb:27017/kryptonite --eval "db.getCollection('mysqlhost.inventory.addresses').find()"
```

This should show a JSON representation of all address documents with their `street` field being decrypted. Here is one example:

```json5
{ "_id" : NumberLong(13), "zip" : "39701", "state" : "Mississippi", "customer_id" : NumberLong(1003), "type" : "SHIPPING", "city" : "Columbus", "street" : "3787 Brownton Road" }
```

### 6. Stop Docker Compose Environment

Running `docker compose down` stops all services.
