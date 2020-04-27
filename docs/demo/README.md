
# Flink SQL Gateway Demo

The goal is to get an SQL gateway pointed to a Flink cluster with live data for easy demoing of the SQL capabilities.

## Setup

Flink demo cluster is based on https://github.com/ververica/sql-training which has good [setup instructions](https://github.com/ververica/sql-training/wiki/Setting-up-the-Training-Environment):

    git clone https://github.com/ververica/sql-training.git
    cd sql-training
    docker-compose up -d


Here we start a SQL client container to install the gateway (to avoid installing Flink again) but this could be done locally or in another of the containers.

    docker-compose exec sql-client bash

Grab a [release](https://github.com/ververica/flink-sql-gateway/releases) of the gateway:

    cd /opt
    wget https://github.com/ververica/flink-sql-gateway/releases/download/v0.1-snapshot/flink-sql-gateway-0.1-SNAPSHOT-bin.zip
    unzip flink-sql-gateway-0.1-SNAPSHOT-bin.zip
    cd flink-sql-gateway-0.1-SNAPSHOT

    echo $FLINK_HOME

    cd bin
    ./sql-gateway.sh

    CTRL-Z
    bg

    curl localhost:8083/v1/info
    > {"product_name":"Apache Flink","version":"1.10.0"}

    curl -X POST localhost:8083/v1/sessions -d '{"planner":"blink","execution_type":"streaming"}'
    > {"session_id":"7eea0827c249e5a8fcbe129422f049e8"}

## Adding tables

Now we are configuring the gateway to load the demo tables and borrow from https://github.com/ververica/sql-training/blob/master/build-image/conf/sql-client-conf.yaml

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