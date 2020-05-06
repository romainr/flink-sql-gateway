
# Flink SQL Editor

The goal is to demo how to execute Flink SQL queries. We use the new Flink SQL gateway project and point to a Flink cluster with live data in a docker container. Hue is used as the SQL Editor for querying Flink tables.

## Setup

Flink demo cluster is based on https://github.com/ververica/sql-training which has easy [setup instructions](https://github.com/ververica/sql-training/wiki/Setting-up-the-Training-Environment):

    git clone https://github.com/ververica/sql-training.git
    cd sql-training
    docker-compose up -d

Then http://localhost:8081/#/overview should be up.

[image ]


Here we start a SQL client container and install the gateway inside (to avoid installing a local Flink as the gateway needs a FLINK_HOME) but this could be done locally or in another containers.

    docker-compose exec sql-client bash

We Grab a [release](https://github.com/ververica/flink-sql-gateway/releases) of the gateway:

    cd /opt
    wget https://github.com/ververica/flink-sql-gateway/releases/download/v0.1-snapshot/flink-sql-gateway-0.1-SNAPSHOT-bin.zip
    unzip flink-sql-gateway-0.1-SNAPSHOT-bin.zip
    cd flink-sql-gateway-0.1-SNAPSHOT

    echo $FLINK_HOME

Then we copy the Flink SQL config to the gateway so that we get the demo tables by default:

    docker cp docs/demo/sql-gateway-defaults.yaml flink-sql-training_sql-client_1:/opt/flink-sql-gateway-0.1-SNAPSHOT/conf/

And we are ready to boot it:

    cd bin
    ./sql-gateway.sh --library /opt/sql-client/lib

Putting the server in the background with `CTRL-Z` and then:

    bg

And now we can issue a few commands to validate the setup:

    curl localhost:8083/v1/info
    > {"product_name":"Apache Flink","version":"1.10.0"}

    curl -X POST localhost:8083/v1/sessions -d '{"planner":"blink","execution_type":"streaming"}'
    > {"session_id":"7eea0827c249e5a8fcbe129422f049e8"}


## Query Editor

As detailed in the [connector](https://docs.gethue.com/administrator/configuration/connectors/) section of Hue, we add a Flink interpreter:

    [notebook]
    [[interpreters]]

    [[[flink]]]
      name=Flink
      interface=flink
      options='{"api_url": "http://172.18.0.7:8993"}'

If setting up the gateway in the client container and we want to access it via your local host, we need to update its bind IP with the IP of the sql client container.

The IP of the API is the one of the running container. We inspect the `flink-sql-training_sql-client_1` to retrieve its IP:

    docker ps
    > CONTAINER ID        IMAGE                                                COMMAND                  CREATED              STATUS              PORTS                                                NAMES
    > 638574b31cd6        fhueske/flink-sql-training:1-FLINK-1.10-scala_2.11   "/docker-entrypoint.…"   About a minute ago   Up About a minute   6123/tcp, 8081/tcp                                   flink-sql-training_sql-client_1
    > 59d1627c412a        wurstmeister/kafka:2.12-2.2.1                        "start-kafka.sh"         About a minute ago   Up About a minute   0.0.0.0:9092->9092/tcp                               flink-sql-training_kafka_1
    > 6711c0707f1e        flink:1.10.0-scala_2.11                              "/docker-entrypoint.…"   About a minute ago   Up About a minute   6121-6123/tcp, 8081/tcp                              flink-sql-training_taskmanager_1
    > 6a8149af6c1e        flink:1.10.0-scala_2.11                              "/docker-entrypoint.…"   About a minute ago   Up About a minute   6123/tcp, 0.0.0.0:8081->8081/tcp                     flink-sql-training_jobmanager_1
    > 3de8275dff26        wurstmeister/zookeeper:3.4.6                         "/bin/sh -c '/usr/sb…"   About a minute ago   Up About a minute   22/tcp, 2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp   flink-sql-training_zookeeper_1
    > a28cee7627a0        mysql:8.0.19                                         "docker-entrypoint.s…"   About a minute ago   Up About a minute   3306/tcp, 33060/tcp                                  flink-sql-training_mysql_1

    docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' 638574b31cd6
    > 172.18.0.7

## Next

There are lot of [future iterations](https://github.com/cloudera/hue/blob/master/docs/designs/apache_flink.md) on this first version to make it production ready but the base is getting there.

It should also be possible to deploy the SQL gateway not in the SQL client container by having:

* local Flink [binary package](https://www.apache.org/dyn/closer.lua/flink/flink-1.10.0/flink-1.10.0-bin-scala_2.11.tgz) with FLINK_HOME configured
* Updating the `jobmanager.rpc.address` to the real jobmanger address in $FLINK_HOME/conf/flink-conf.yaml
* Changing the two address properties: sql-gateway-defaults.yaml

    server:
      # The address that the gateway binds itself.
      bind-address: 172.18.0.7
      # The address that should be used by clients to connect to the gateway.
      address: 172.18.0.7



## Adding tables

Now we are configuring the gateway to load the demo tables and borrow from https://github.com/ververica/sql-training/blob/master/build-image/conf/sql-client-conf.yaml

    apt-get update
    apt-get install vim

    vim ../conf/sql-gateway-defaults.yaml

And update the *tables:* and *functions:* sections like in [sql-gateway-defaults.yaml](sql-gateway-defaults.yaml):

    tables:
      - name: Rides
        type: source
        update-mode: append
        schema:
        - name: rideId
          type: LONG
        - name: taxiId
          type: LONG
        - name: isStart
          type: BOOLEAN
        - name: lon
          type: FLOAT
        - name: lat
          type: FLOAT
        - name: rideTime
          type: TIMESTAMP
          rowtime:
            timestamps:
              type: "from-field"
              from: "eventTime"
            watermarks:
              type: "periodic-bounded"
              delay: "60000"
        - name: psgCnt
          type: INT
        connector:
          property-version: 1
          type: kafka
          version: universal
          topic: Rides
          startup-mode: earliest-offset
          properties:
          - key: zookeeper.connect
            value: zookeeper:2181
          - key: bootstrap.servers
            value: kafka:9092
          - key: group.id
            value: testGroup
        format:
          property-version: 1
          type: json
          schema: "ROW(rideId LONG, isStart BOOLEAN, eventTime TIMESTAMP, lon FLOAT, lat FLOAT, psgCnt INT, taxiId LONG)"
      - name: Fares
        type: source
        update-mode: append
        schema:
        - name: rideId
          type: LONG
        - name: payTime
          type: TIMESTAMP
          rowtime:
            timestamps:
              type: "from-field"
              from: "eventTime"
            watermarks:
              type: "periodic-bounded"
              delay: "60000"
        - name: payMethod
          type: STRING
        - name: tip
          type: FLOAT
        - name: toll
          type: FLOAT
        - name: fare
          type: FLOAT
        connector:
          property-version: 1
          type: kafka
          version: universal
          topic: Fares
          startup-mode: earliest-offset
          properties:
          - key: zookeeper.connect
            value: zookeeper:2181
          - key: bootstrap.servers
            value: kafka:9092
          - key: group.id
            value: testGroup
        format:
          property-version: 1
          type: json
          schema: "ROW(rideId LONG, eventTime TIMESTAMP, payMethod STRING, tip FLOAT, toll FLOAT, fare FLOAT)"
      - name: DriverChanges
        type: source
        update-mode: append
        schema:
        - name: taxiId
          type: LONG
        - name: driverId
          type: LONG
        - name: usageStartTime
          type: TIMESTAMP
          rowtime:
            timestamps:
              type: "from-field"
              from: "eventTime"
            watermarks:
              type: "periodic-bounded"
              delay: "60000"
        connector:
          property-version: 1
          type: kafka
          version: universal
          topic: DriverChanges
          startup-mode: earliest-offset
          properties:
          - key: zookeeper.connect
            value: zookeeper:2181
          - key: bootstrap.servers
            value: kafka:9092
          - key: group.id
            value: testGroup
        format:
          property-version: 1
          type: json
          schema: "ROW(eventTime TIMESTAMP, taxiId LONG, driverId LONG)"
      - name: Drivers
        type: temporal-table
        history-table: DriverChanges
        primary-key: taxiId
        time-attribute: usageStartTime

    functions:
    - name: timeDiff
      from: class
      class: com.ververica.sql_training.udfs.TimeDiff
    - name: isInNYC
      from: class
      class: com.ververica.sql_training.udfs.IsInNYC
    - name: toAreaId
      from: class
      class: com.ververica.sql_training.udfs.ToAreaId
    - name: toCoords
      from: class
      class: com.ververica.sql_training.udfs.ToCoords
