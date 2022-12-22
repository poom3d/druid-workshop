# Druid Workshop

## Commands

Produce to Kafka:

`kafka-console-producer.sh --broker-list kafka:9092 --topic test-data`

Consume from Kafka:

`kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic test-data --from-beginning`


## Workshop

1. Run cluster with

`docker-compose up`

2. We will ingest data from Kafka so let's produce some test data into kafka topic

`kafka-console-producer.sh --broker-list kafka:9092 --topic test-data`

With the following json:

```
{"product_id": 1, "price": 100, "user": {"user_id": 1, "user_name": "user1"}, "dt": 1671506358}
{"product_id": 2, "price": 200, "user": {"user_id": 2, "user_name": "user2"}, "dt": 1671506458}
{"product_id": 1, "price": 100, "user": {"user_id": 3, "user_name": "user3"}, "dt": 1671506558}

```

3. Open Druid Console via `http://localhost:8888`

4. Click on Load Data and select Kafka

5. Fill in following detail to connect to Kafka topic and then click Apply

![Kafka Config](/images/kafka-config.png "Kafka Config")

6. After some clicking through, you should see the data being ingested to Druid. After that, you can query via curl or console:

`curl -X 'POST' -H 'Content-Type:application/json' -d '{"query":"SELECT * FROM \"test-data\""}' http://localhost:8888/druid/v2/sql`

7. Let's add more data:

{"product_id": 1, "price": 100, "user": {"user_id": 1, "user_name": "user1"}, "dt": 1671516358}
{"product_id": 2, "price": 200, "user": {"user_id": 2, "user_name": "user2"}, "dt": 1671516458}
{"product_id": 1, "price": 100, "user": {"user_id": 3, "user_name": "user3"}, "dt": 1671516558}
{"product_id": 1, "price": 100, "user": {"user_id": 5, "user_name": "user1"}, "dt": 1671517358}
{"product_id": 2, "price": 200, "user": {"user_id": 6, "user_name": "user2"}, "dt": 1671517458}
{"product_id": 1, "price": 100, "user": {"user_id": 7, "user_name": "user3"}, "dt": 1671517558}
{"product_id": 1, "price": 100, "user": {"user_id": 8, "user_name": "user1"}, "dt": 1671518358}
{"product_id": 2, "price": 200, "user": {"user_id": 9, "user_name": "user2"}, "dt": 1671518458}
{"product_id": 1, "price": 100, "user": {"user_id": 1, "user_name": "user3"}, "dt": 1671518558}
{"product_id": 1, "price": 100, "user": {"user_id": 2, "user_name": "user1"}, "dt": 1671519358}
{"product_id": 2, "price": 200, "user": {"user_id": 3, "user_name": "user2"}, "dt": 1671519458}
{"product_id": 1, "price": 100, "user": {"user_id": 4, "user_name": "user3"}, "dt": 1671519558}
{"product_id": 1, "price": 100, "user": {"user_id": 5, "user_name": "user1"}, "dt": 1671520358}
{"product_id": 2, "price": 200, "user": {"user_id": 6, "user_name": "user2"}, "dt": 1671520458}
{"product_id": 1, "price": 100, "user": {"user_id": 7, "user_name": "user3"}, "dt": 1671520558}

8. Let's do some more fancy query:

Basically group sum of price by hour.

```
SELECT FLOOR(__time to HOUR) AS HourTime, SUM(price) AS Accumulated
FROM "test-data" WHERE "__time" BETWEEN TIMESTAMP '2022-12-19 00:00:00' AND TIMESTAMP '2022-12-21 00:00:00'
GROUP BY 1
```

9. Theta Sketch Tutorial

This tutorial works with the following data:

date: a timestamp. In this case it's just dates but as mentioned earlier, a finer granularity makes sense in real life.
uid: a user ID
show: name of a TV show
episode: episode identifier

Data:

```
date,uid,show,episode
2022-05-19,alice,Game of Thrones,S1E1
2022-05-19,alice,Game of Thrones,S1E2
2022-05-19,alice,Game of Thrones,S1E1
2022-05-19,bob,Wednesday,S1E1
2022-05-20,alice,Game of Thrones,S1E1
2022-05-20,carol,Wednesday,S1E2
2022-05-20,dan,Wednesday,S1E1
2022-05-21,alice,Game of Thrones,S1E1
2022-05-21,carol,Wednesday,S1E1
2022-05-21,erin,Game of Thrones,S1E1
2022-05-21,alice,Wednesday,S1E1
2022-05-22,bob,Game of Thrones,S1E1
2022-05-22,bob,Wednesday,S1E1
2022-05-22,carol,Wednesday,S1E2
2022-05-22,bob,Wednesday,S1E1
2022-05-22,erin,Game of Thrones,S1E1
2022-05-22,erin,Wednesday,S1E2
2022-05-23,erin,Game of Thrones,S1E1
2022-05-23,alice,Game of Thrones,S1E1
```

![Tutorial Theta 01](/images/tutorial-theta-01.png "Ingest data")


### Basic counting
Let's first see what the data looks like in Druid. Run the following SQL statement in the query editor:

`SELECT * FROM ts_tutorial`


The following query to compute the distinct counts of user IDs uses APPROX_COUNT_DISTINCT_DS_THETA and groups by the other dimensions:

```
SELECT __time,
       "show",
       "episode",
       APPROX_COUNT_DISTINCT_DS_THETA(theta_uid) AS users
FROM   ts_tutorial
GROUP  BY 1, 2, 3
```


As an example, query the total unique users that watched Wednesday:

```
SELECT THETA_SKETCH_ESTIMATE(
         DS_THETA(theta_uid) FILTER(WHERE "show" = 'Wednesday')
       ) AS users
FROM ts_tutorial
```

How many users watched both episodes of Wednesday? Use THETA_SKETCH_INTERSECT to compute the unique count of the intersection of two (or more) segments:

```
SELECT THETA_SKETCH_ESTIMATE(
         THETA_SKETCH_INTERSECT(
           DS_THETA(theta_uid) FILTER(WHERE "show" = 'Wednesday' AND "episode" = 'S1E1'),
           DS_THETA(theta_uid) FILTER(WHERE "show" = 'Wednesday' AND "episode" = 'S1E2')
         )
       ) AS users
FROM ts_tutorial
```

Likewise, use THETA_SKETCH_UNION to find the number of visitors that watched any of the episodes:

```
SELECT THETA_SKETCH_ESTIMATE(
         THETA_SKETCH_UNION(
           DS_THETA(theta_uid) FILTER(WHERE "show" = 'Wednesday' AND "episode" = 'S1E1'),
           DS_THETA(theta_uid) FILTER(WHERE "show" = 'Wednesday' AND "episode" = 'S1E2')
         )
       ) AS users
FROM ts_tutorial
```


And finally, there is THETA_SKETCH_NOT which computes the set difference of two or more segments. The result describes how many visitors watched episode 1 of Bridgerton but not episode 2.

```
SELECT THETA_SKETCH_ESTIMATE(
         THETA_SKETCH_NOT(
           DS_THETA(theta_uid) FILTER(WHERE "show" = 'Wednesday' AND "episode" = 'S1E1'),
           DS_THETA(theta_uid) FILTER(WHERE "show" = 'Wednesday' AND "episode" = 'S1E2')
         )
       ) AS users
FROM ts_tutorial
```

### Conclusions
Counting distinct things for large data sets can be done with Theta sketches in Apache Druid.
This allows us to use rollup and discard the individual values, just retaining statistical approximations in the sketches.
With Theta sketch set operations, affinity analysis is easier, for example, to answer questions such as which segments correlate or overlap by how much.
