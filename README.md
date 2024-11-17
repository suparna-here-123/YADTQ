# Yet Another Distributed Task Queue
## An implementation using Apache Kafka

### Client
* Client interacts with YADTQ via a FastAPI-backed GUI
* Client can submit and monitor tasks

### Producer 
* To run from CLI - "tail -f a_tasks.txt | python3 producer.py \<topic\> \<number of partitions in topic\>"
* Tasks with details like Task ID, Task Type, Task Arguments, Task Tries are written to a text file
* Producer publishes every task to the topic
* Task distribution to partitions via default round-robin fashion
* Task is recorded in Redis datastore with Task ID, Task Status = 'QUEUED', number of attempts made to execute = 0
* "end" input from text file signals end of tasks in queue - all processes exit

### Failure-Prone Consumer
* To run from CLI - "python3 consumer.py \<consumer name\> \<topic\>"
* Each consumer is part of consumer group 'workers', subscribed to topic
* Executes tasks only in its assigned partition
* Successfully executed tasks are written to Redis datastore with Task ID, result, number of attempts made
* Failed tasks are retried 3 times after which, Task ID, result = None, error logs, nnumber of attempts made are written to Redis
* Every message offset is committed manually only after successfully or thrice executed.
* Above done so that in case of consumer failure, reassigned consumer will process only from latest message received in partition, saving time by not reprocessing older messages
* Task statuses can move from QUEUED -> PROCESSING -> SUCCESS or FAILED
* To simulate failure during task execution, consumer process may exit any time between execution of 3rd to 5th task received.
* Alternatively, Ctrl + C may also be applied to kill consumer process from CLI
* Kafka's load-balancing mechanism reassigns tasks to stand-by or active consumer.

### Guarantee Stand-by Consumer (Optional)
* To run from CLI - "python3 guarantee_consumer.py \<consumer name\> \<topic\>
* This consumer will not fail - done only to show that partitions of failed consumers are assigned to this (can be seen in terminal output)
* Implemented for our understanding

## Heartbeat Monitoring
* To run from CLI - "python3 monitor_hearbeat.py"
* Receives heartbeats from every active and stand-by consumer every 5 seconds
* If any consumer has a heartbeat delayed more than allowed (and configurable) hearbeat interval, it is declared "DEAD"
* Else, consumer is declared "ALIVE"
* Heartbeat logs stored in Redis under "all_heartbeats" key.
