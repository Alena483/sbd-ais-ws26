# Stage A: 
Run 1: 20 machines (200 readings/s)  
Rows in table readings (this run) - 19,260  
Read by the detector - 16,974  
Reported as missing - 2,286  
16,974 + 2,286 = 19,260.   
The missing readings are in the table, the detector simply never read 2,286 of them. A row gets its id, when it is inserted, but other transactions see it only after the commit. The four gateways insert and commit their batches concurrently, so commit order differs from id order. 
Timeline from the task: gateway 1 inserts rows 100-114 and commits 100 ms later; gateway 2 inserts 115-129 and commits at once. If the detector polls in between, it sees only rows 115-129 and remember last_id = 129. When rows 100-114 become visible, id > 129 never returns them: they are skipped forever. The model notices the gap in seq per machine and counts it as missing.

Run 2: all 60 machines (600 readings/s)  
Delay grew almost linearly from 2.9 s to 42.6 s; bbacklog grew to almost 19,000 rows while the plant was running, and only shrank after the plant stopped. It happened because the model costs 3 ms per reading, so one detector handles at most 333 readings/s, but 600/s arrive. The backlog grows by 270 rows/s. This is a throughput problem, not a delivery problem.
Missing stayed low (33): The detector was far behind, so it read old rows that were already committed. Late commits only cause skipped rows near the newest end of the table, and the detector was not there. So the lag hid the problem, but it did not fix it. A faster detector (or several pollers) that reads the newest rows would skip rows again.

# Stage B
Option 1: keep PostgreSQL, use several pollers  
More pollers help with the model's speed, but 100,000 inserts per second plus constant queries is too much for one database.
The skipped-rows problem stays. As a result, we would need to wait before reading new rows to avoid skipped ones, and this adds delay.Every new consumer (archive, dashboard) sends more queries to PostgreSQL.
Option 2: gateways write every reading into Kafka  
We can add partitions and detector instances; each detector instance reads its own partitions. Kafka can easily handle
  100,000 small messages per second. he key is the machine id, so all readings of one machine go to the same partition and stay in the right order. There is no commit problem, because the consumer reads the log by offset, one message after another. If enough instances run, the lag stays near 0. Archive, dashboard and detector are separate consumer groups with their own offsets; they do not influence each other and put no load on PostgreSQL. A new group can also read the old data again from the beginning.
Risk: if a partition moves to another detector, that detector has no state for these machines and must warm up again (100 readings). So we should keep the number of instances stable.  
Decision - Option 2: the gateways write every reading into Kafka:
1) No more skipped readings. In PostgreSQL, the detector looks only at rows with id bigger than the last one. This breaks when rows are committed late. Kafka works differently - every message gets a position in the log in the order it was written, and the consumer reads the log strictly by offset.
2) The work is split by partitions. Each detector instance reads its own partitions, so we can add instances and partitions when the plant grows from 60 to 2,000 machines.
3) The archive and the dashboard use their own consumer groups. Each group has its own offset, so it can be slow or fast without any effect on the others.


600 readings/s: 600 * 0.003 = 1.8, so we need at least 2 detector instances.    
100,000 readings/s: 100,000 * 0.003 = 300, so we need at least 300 instances. 
The number of partitions must be at least the number of instances, because one partition is read by only one consumer of a group.

When we don`t have much data, then PostgreSQL is enough. We need Kafka when there is much more data, when many consumers read the same data, when the order per key is important, or when we want to replay old data.