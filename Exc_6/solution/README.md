## Part 1: Decision

I choose log-based CDC with Debezium. The payment path is certified and must not change, so dual write and outbox are not the right choice (R1). Polling puts query load on the system of record and can miss payments that commit out of id order, so it breaks R1 and R2. CDC reads the write-ahead log, so it needs no code change and no queries, and it delivers every committed payment in commit order, to any number of consumer groups. The cost is the replication slot: PostgreSQL keeps all WAL that the connector has not read yet, so if the connector stops, the disk can fill up and the database, and with it the payment path, stops. In production I would monitor the connector status, the retained WAL size per slot and the free disk space, and alert long before the disk is full. If my team owned the payment path, I would choose the outbox pattern, because it is just as correct and gives the application control over the event contract.

## Runs

Run 1 ran at ~760 payments/s. Runs 2 and 3 ran with --tps 300, because my laptop (Docker, Kafka, Debezium, the card processor and 4 agents on one machine) could not keep up at full rate.

### Run 1: Debezium's defaults

- Prediction: all three velocity instances share the work and R4 passes.
- Scoreboard (velocity):
  R3  frauds caught          1/14  (7%)   FAIL
  R4  alert latency p50/p95  115.3 s / 133.3 s   FAIL
- Why: nobody configured the topic, so Debezium created it with 1 partition (Kafka UI: Partitions = 1), and the key of an event is the primary key {"id": 57372}. In a consumer group one partition is read by exactly one instance, so only vel-1 got assigned partitions [0]; vel-2 and vel-3 got []. One instance handles at most around 500 payments/s, but 761 arrived, so the lag grew above 200 s: R4 fails, and most frauds were still in the backlog (R3 fails). The anomaly model failed here too, which may be just due to laptop being overloaded at this rate, not to the configuration.

### Run 2: 6 partitions

- Prediction: all three instances work and R4 passes, but R3 fails, because the key is still the primary key.
- Scoreboard (velocity):
  R3  frauds caught          0/7  (0%)   FAIL
  R4  no alerts; worst lag of the velocity agents 5.7 s   FAIL
- Why: the six partitions were split over the instances ([0, 1], [2, 3], [4, 5]) and each kept up with a lag of 0.2-2 s (R4 failed only because of one spike to 5.7 s). But the key is still id, so hash(key) mod 6 spreads the 12 test payments of one stolen card over all partitions and all instances. Each instance sees only about 4 of them in its 60-s window, below the threshold of 8, so no instance sees the whole attack. The anomaly model still caught 9/9 in the first attempt because its single instance reads all six partitions.

### Run 3: cardholder as key

- Prediction: the key becomes {"user_id": ...}, all payments of a cardholder land in one partition and one instance, so R3 passes.
- Scoreboard (velocity):
  R3  frauds caught          6/6  (100%)   PASS
  R3  false alerts           0   PASS
  R4  alert latency p50/p95  1.7 s / 8.7 s   FAIL
- Why: with message.key.columns set to user_id (Kafka UI: key {"user_id": 143250}), all payments of a cardholder go to the same partition and are read by the same instance in commit order, so the velocity model sees the whole attack and flags the 8th payment (R3 passes). The median latency is 1.7 s; the p95 of 8.7 s comes from a lag peak of ~8-12 s in the first minute, while Kafka, Debezium and the agents were still starting on my laptop, and afterwards the lag stayed at 0.3-2 s. So the remaining R4 miss is a resource problem of my laptop, not of the design.

## Conclusion

Debezium chose 1 partition and the primary key as the event key. If nobody had checked them, nothing would have crashed and no error would have appeared: the velocity model would have run on a single instance and fallen further behind every second (in Run 1 the lag grew above 200 s), and once scaled out with the default key it would have silently missed almost all card-testing attacks (Run 2: 0 of 7). The team could have noticed this by monitoring consumer lag, by checking that every instance has assigned partitions, and with a regular quality metric (the share of confirmed frauds that the model caught, compared with the chargeback data).
