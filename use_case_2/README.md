# Use Case 2

All services for this demo scenario are pre-configured and supposed to be run using the provided docker compose environment.

_NOTE: Make sure you are in the `use_case_2/
` directory of this repository before running any of the following commands._

### 0. Start Docker Compose Environment

Running `docker compose up` starts the following services:

* Zookeeper
* Kafka Broker
* Kafka Connect
* Minio

### 1. Inspect original JSON records of input file

```
docker run --tty --rm \
    --network uc2-network \
    -v ${PWD}/:/home \
    debezium/tooling:1.2 \
    cat /home/data/connect/sample_data.txt | jq .
```

One sample JSON record of this file looks as follows:

```json5
{
  "guid": "1248145f-c80b-414c-805b-e3d5ea8c75e8",
  "personal": {
    "firstname": "Marks",
    "lastname": "Hunter",
    "age": 20,
    "gender": "male",
    "height": 162,
    "weight": 58,
    "eyeColor": "brown"
  },
  "knownResidences": [
    "356 Rock Street, Delwood, Washington, 5632",
    "156 Hart Place, Cartwright, Rhode Island, 5608",
    "911 Frank Court, Santel, Florida, 1797",
    "410 Walker Court, Orason, Maryland, 856"
  ],
  "isActive": false,
  "profilePic": "https://picsum.photos/128",
  "registered": "2020-12-09T03:48:49 -01:00"
}
```


### 2. Create FileStream Source Connector

The Kafka Connect FileStream source connector is configured together with the `CipherField` SMT to read and produce Kafka records based on the JSON payloads in the file `data/connect/sample_data.txt`. The JSON object fields `personal` and `knownResidences` get encrypted due to the SMT settings.

Run the following command in your host terminal to start the debezium tools container and enter an interactive bash session.

```
docker run -it --rm \
    --network uc2-network \
    -v ${PWD}/:/home \
    debezium/tooling:1.2 \
    bash
```

_NOTE: All of the following commands are supposed to be executed in the container's bash._

[kcctl](https://github.com/kcctl/kcctl) - a CLI for Apache Kafka Connect - is used to perform any Kafka Connect related operations. First the connect cluster address is set and used as the CLI tool's context. Then the MySQL source connector is created.

```
kcctl config set-context default --cluster=http://connect:8083
```

Depending on which SMT `field_mode` should be used during encryption there are different source connector configurations available to choose from.

**For encryption in `OBJECT` mode run:**

```
kcctl apply -f /home/register_filestream_source_enc_objects.json
```

**For encryption in `ELEMENT` mode run:**

```
kcctl apply -f /home/register_filestream_source_enc_elements.json
```

### 3. Inspecting Partially Encrypted Records

The next step is to verify the resulting records in the Kafka corresponding Kafka topics.

**To inspect records encrypted with `OBJECT` mode run:**

`
kafkacat -b kafka:9092 -C -t personal-data-enc-objects -o beginning -q | jq .
`

This shows JSON records with their `personal` and `knownResidences` fields being encrypted in `OBJECT` mode. Here is one example:

```json5
{"guid":"837abb22-3e56-426b-8748-90d2ce4b1e5c","personal":"jwHnSLlRAlv3Ffl5+iF1kDOA4caLOWdSzs+OJhvHFdQtZFt+Vqe+SXllpRDkaN71STz3L51RaQoaKoPIRdT/p8mcZdixKzTlC7QHQbBP9maIZShZnSCgkPMWCgOwNXIrWVfjBc6HuWW89JOoLPymxC9rXSvCDy3pDgQw5Ie5/Tc/fSQN2FMzmv7LEhhiLDwlDDCxa7E=","knownResidences":"iQJXvXNgsmuBQHqYqIkcD4qcYdNYLEFdlTswcXNtpWdCFPLLU2wmSoDLGuBGs1z38WlL777+SMhbvDpEPdJQHUuACQUbY5wvHwMOoAHwWIHy7MVZsuzAQq+mNl3Ed5X0IjCIlcoU2W+TWmWPdVKr0S8XdBmerv8RQObaEwm6Aa2W1ZWQ4NGzMmsP1MTNYtZuyTQQ3FwhZkO4KYb2DaLzIh9tQF3I21VTE1BnbtqgC4FVpoqsPZlh1UNZ32quuzfs9MK5zxiKSz4IiF48Loj3BltbM8d92zaH0V4G/O75UW7rP1558/ApDLem5zE7tcVbmUvw7F3jc/b+Q6IZuNbR2P2sv9cWAaNSfGoMMLFrsQ==","isActive":true,"profilePic":"https://picsum.photos/128","registered":"2018-08-12T01:58:44 -02:00"}
```

Afterwards, hit `CTRL + C` to stop the kafkacat consumer process.

**To inspect records encrypted with `ELEMENT` mode run:**

`
kafkacat -b kafka:9092 -C -t personal-data-enc-elements -o beginning -q | jq .
`

This shows JSON records with their `personal` and `knownResidences` fields being encrypted in `ELEMENT` mode. Here is one example:

```json5
{"guid":"837abb22-3e56-426b-8748-90d2ce4b1e5c","personal":{"firstname":"JtrbAn6T6PU/wteHkXERsfOEqRox9GpawRkPSofmusutTLC+ZDUMMLFrsQ==","lastname":"J3giBjRif2ozFvgy9tLag8LbLtJyB/Pzvb+I5A3h2A/Dbo7TY4LZDDCxa7E=","age":"Iy6JS4+fpZdXmiHZ9C/4wuRrELIp7nZHVjNnysXRtYz4oC0MMLFrsQ==","gender":"KFBI3kqXrniqsuaH01BvY33SaDRWCsh0WsP7AhHKJMz8PmcvU3txCQwwsWux","height":"JFfZbIwwd3cPTtLaWwMJwJOX/M5cxHPoKLXXvczuXx3MI5AIDDCxa7E=","weight":"IwtaJ8YGvDtapmXMmUZ06CIdYSQYt81P7ym6ecZBrCjoGxYMMLFrsQ==","eyeColor":"JxpC99oxiX8aHr4eng0yTdagQU13jCfmXCkgzcg2Klki+mzH0ApQDDCxa7E="},"knownResidences":["VLLgGES7/BgsfFf6Hlj4ZbAx33Ovfrn7KguKOu0yB4cOaynM21UBF47MXSWwB2FyO2kdP5YLrXn73gxro8vMQODyfIFDP+Ajb5foqANFserI5deZDDCxa7E=","TFRt5f0hFwMQYTcYNEAAsrWPNh6Dz/vTtXz2Gk6QlUJiQFisBBypL2Fw6j0CFR9xERWbrIE3f93ZQ5yfx3MLZQUIfH1a+5zum4U/6QwwsWux","UmIXS+hrPXyLJF24T5T/AYJ6UZoEbOeOriVW7CUdaGb9t8puWT21vf7LeHVfoMaSA3UsMpab+9y/J1LHzg4qOd5qQ3zZuVsoUoo26Hk53Ym60AwwsWux","Zrt4XP4j5nf+dHIDFlb8Hr5rnl6mWPi1Phi12Mqwh/BsDmBk7/JJ1N5l0OUS+ipr3sbJaMa4R1YVsXOXC6IntTumvDwBqjWj4jkv9baXqHxWS1gA1Q0TpW3MjRAkiNT8PQ3ttpePDDCxa7E="],"isActive":true,"profilePic":"https://picsum.photos/128","registered":"2018-08-12T01:58:44 -02:00"}
```

Afterwards, hit `CTRL + C` to stop the kafkacat consumer process.

### 4. Create Camel Minio Sink Connector

The Apache Camel Minio sink connector is configured together with the `CipherField` SMT to store the records as files into a Minio bucket named `kafka-connect-kryptonite`. The values for both fields `personal` and `knownResidences` get decrypted due to the SMT settings.

Again kcctl is used to create connector instances. Depending on which SMT `field_mode` should be used during decryption there are different sink connector configurations available to choose from.

**For decryption in `OBJECT` mode run:**

```
kcctl apply -f /home/register_minio_sink_dec_objects.json
```

**For decryption in `ELEMENT` mode run:**

```
kcctl apply -f /home/register_minio_sink_dec_elements.json
```

In the container's bash type `exit` to get back to your host terminal.

### 5. Inspecting Decrypted Documents

To verify the resulting files in the Minio bucket `kafka-connect-kryptonite` run the following container and enter an interactive bash session:

```
docker run -it --rm \
    --network uc2-network \
    --entrypoint bash \
    quay.io/minio/mc:RELEASE.2021-11-05T10-05-06Z
```

In the container's bash the Minio client `mc` is used to interact with buckets and their files.

The following command sets the client endpoint to the minio service running in docker compose:

```
mc alias set myminio http://minio:9000 admin minio12345
```

The next step is to list all of the bucket entries in a specific bucket

```
mc ls myminio/kafka-connect-kryptonite-objects

mc ls myminio/kafka-connect-kryptonite-elements
```

which shows all contained files similar to the listing below. Note that the actual file names will be different though: 

```
[2021-11-26 10:14:54 UTC]   503B 20211126-101452733-CCE479B40963B68-0000000000000000.json
[2021-11-26 10:14:54 UTC]   370B 20211126-101454699-CCE479B40963B68-0000000000000001.json
[2021-11-26 10:14:54 UTC]   476B 20211126-101454761-CCE479B40963B68-0000000000000002.json
[2021-11-26 10:14:54 UTC]   478B 20211126-101454802-CCE479B40963B68-0000000000000003.json
[2021-11-26 10:14:54 UTC]   348B 20211126-101454850-CCE479B40963B68-0000000000000004.json
[2021-11-26 10:14:54 UTC]   478B 20211126-101454920-CCE479B40963B68-0000000000000005.json
[2021-11-26 10:14:55 UTC]   390B 20211126-101454991-CCE479B40963B68-0000000000000006.json
[2021-11-26 10:14:55 UTC]   466B 20211126-101455048-CCE479B40963B68-0000000000000007.json
[2021-11-26 10:14:55 UTC]   389B 20211126-101455094-CCE479B40963B68-0000000000000008.json
[2021-11-26 10:14:55 UTC]   478B 20211126-101455144-CCE479B40963B68-0000000000000009.json
...
```

In order to see the contents of such files run 

```
mc cat myminio/kafka-connect-kryptonite-objects/20211126-101452733-CCE479B40963B68-0000000000000000.json

mc cat myminio/kafka-connect-kryptonite-elements/20211126-203132842-21946983760726A-0000000000000000.json
```

while making sure to adapt the filenames accordingly.

This shows a JSON representation of one record in which with both fields, `personal` and `knownResidences` have been successfully decrypted: 

```json5
{"profilePic":"https://picsum.photos/128","guid":"837abb22-3e56-426b-8748-90d2ce4b1e5c","registered":"2018-08-12T01:58:44 -02:00","personal":{"firstname":"Judy","lastname":"Hayes","age":38,"gender":"female","height":167,"weight":50,"eyeColor":"brown"},"isActive":true,"knownResidences":["529 Glenmore Avenue, Waumandee, Connecticut, 8220","489 Lake Street, Glasgow, Tennessee, 1469","927 Rutledge Street, Fresno, Pennsylvania, 5610","728 Debevoise Street, Gerton, Federated States Of Micronesia, 7007"]}
```

### 6. Stop Docker Compose Environment

Running `docker compose down` stops all services.
