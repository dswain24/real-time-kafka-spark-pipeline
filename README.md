```markdown
# Real-Time Streaming Pipeline (Kafka + Spark)
## Overview
This project simulates a real-time data streaming pipeline using dummy event data. A Python producer
sends fake
events (user interactions) into a Kafka topic. A Spark Structured Streaming job reads events from Kafka,
parses
them, and writes them to the console (or can be extended to S3 or a database).
## Tech Stack
- Python
- Kafka (via Docker)
- Apache Spark (Structured Streaming)
- Dummy JSON events (no real user data)
## Folder Structure
```
real-time-kafka-spark-pipeline/
requirements.txt
docker-compose.yml
producer.py
spark_streaming.py

```
## How It Works
1. Docker runs a local Kafka broker and Zookeeper.
2. `producer.py` sends randomly generated JSON events to the `events-topic` Kafka topic.
3. `spark_streaming.py` reads from `events-topic`, parses the JSON, and prints the records to the
console.
## Prerequisites
- Python 3.9+
- Docker Desktop installed and running
- Java installed (required for Spark)
## Setup and Run Steps
1. Create and activate a virtual environment:
```bash
python -m venv venv
# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate
```
2. Install dependencies:
```bash
pip install -r requirements.txt
```
3. Start Kafka using Docker:
```bash
docker-compose up -d
```
4. In one terminal, run the producer:
```bash
python producer.py
```
You should see lines like:
```
Sent: {'user_id': 123, 'event_type': 'click', 'amount': 45.67}
```
5. In another terminal, run the Spark streaming job:
```bash
python spark_streaming.py
```
You should see parsed rows printed in the console output.
## Common Errors and Debugging Tips
**Error 1: `ModuleNotFoundError: No module named 'kafka'`**
- Cause: `kafka-python` is not installed in your environment.
- Fix: Run `pip install -r requirements.txt` (while your virtual environment is active).
**Error 2: `Failed to connect to broker at localhost:9092`**
- Cause: Kafka is not running or Docker is not started.
- Fix:
- Ensure Docker Desktop is running.
- Run `docker-compose up -d` from the project folder.
- Run `docker ps` to confirm Kafka and Zookeeper containers are up.
**Error 3: Spark Java errors or `JAVA_HOME` not set**
- Cause: Java is required for Spark to run.
- Fix:
- Install Java (e.g., OpenJDK).
- Set `JAVA_HOME` environment variable if needed.

**Error 4: No streaming output in Spark**
- Cause: Producer may not be sending data, or topic names do not match.
- Fix:
- Make sure `topic = 'events-topic'` in `producer.py` and `.option('subscribe', 'events-topic')` in
`spark_streaming.py`.
- Confirm producer is running and printing `Sent:` lines.
## How to Stop Everything
- Stop producer and Spark processes with `Ctrl + C` in each terminal.
- Stop Kafka containers:
```bash
docker-compose down
```
```
requirements.txt
```
kafka-python==2.0.2
pyspark==3.5.0
```
docker-compose.yml
```yaml
version: '3.8'
services:
zookeeper:
image: confluentinc/cp-zookeeper:7.5.0
environment:
ZOOKEEPER_CLIENT_PORT: 2181
ZOOKEEPER_TICK_TIME: 2000
ports:
- '2181:2181'
kafka:
image: confluentinc/cp-kafka:7.5.0
depends_on:
- zookeeper
ports:
- '9092:9092'
environment:
KAFKA_BROKER_ID: 1
KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
KAFKA_LISTENERS: 'PLAINTEXT://0.0.0.0:9092'
KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://localhost:9092'
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```
producer.py
```python
import json
import time
import random
from kafka import KafkaProducer
def create_producer():
return KafkaProducer(
bootstrap_servers='localhost:9092',
value_serializer=lambda v: json.dumps(v).encode('utf-8'),
)
if __name__ == '__main__':
producer = create_producer()
topic = 'events-topic'
print(f'Sending messages to topic: {topic}')

while True:
event = {
'user_id': random.randint(1, 1000),
'event_type': random.choice(['click', 'view', 'purchase']),
'amount': round(random.uniform(5.0, 500.0), 2),
}
producer.send(topic, event)
print('Sent:', event)
time.sleep(1)
```
spark_streaming.py
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, DoubleType
if __name__ == '__main__':
spark = (SparkSession.builder
.appName('KafkaSparkStreamingExample')
.getOrCreate())
spark.sparkContext.setLogLevel('WARN')
schema = StructType([
StructField('user_id', IntegerType(), True),
StructField('event_type', StringType(), True),
StructField('amount', DoubleType(), True),
])
df_raw = (spark.readStream
.format('kafka')
.option('kafka.bootstrap.servers', 'localhost:9092')
.option('subscribe', 'events-topic')
.load())
df_parsed = (df_raw
.selectExpr('CAST(value AS STRING) as json_str')
.select(from_json(col('json_str'), schema).alias('data'))
.select('data.*'))
query = (df_parsed
.writeStream
.outputMode('append')
.format('console')
.option('truncate', 'false')
.start())
query.awaitTermination()




