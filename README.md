# Module 7 Homework: Streaming

## NOTE

Below is the exact markdown (MD) file from the original repository (data-engineering-zoomcamp) with **my answers on the bottom**. Please click [here](https://github.com/DataTalksClub/data-engineering-zoomcamp/blob/main/cohorts/2026/07-streaming/homework.md) to go to the actual original file.

---

In this homework, we'll practice streaming with Kafka (Redpanda) and PyFlink.

We use Redpanda, a drop-in replacement for Kafka. It implements the same
protocol, so any Kafka client library works with it unchanged.

For this homework we will be using Green Taxi Trip data from October 2025:

- [green_tripdata_2025-10.parquet](https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_2025-10.parquet)


## Setup

We'll use the same infrastructure from the [workshop](../../../07-streaming/workshop/).

Follow the setup instructions: build the Docker image, start the services:

```bash
cd 07-streaming/workshop/
docker compose build
docker compose up -d
```

This gives us:

- Redpanda (Kafka-compatible broker) on `localhost:9092`
- Flink Job Manager at http://localhost:8081
- Flink Task Manager
- PostgreSQL on `localhost:5432` (user: `postgres`, password: `postgres`)

If you previously ran the workshop and have old containers/volumes,
do a clean start:

```bash
docker compose down -v
docker compose build
docker compose up -d
```

Note: the container names (like `workshop-redpanda-1`) assume the
directory is called `workshop`. If you renamed it, adjust accordingly.


## Question 1. Redpanda version

Run `rpk version` inside the Redpanda container:

```bash
docker exec -it workshop-redpanda-1 rpk version
```

What version of Redpanda are you running?

### Question 1 Answer

ANSWER:
- **rpk version: v25.3.9**

I ran the command listed in the question and got the following output:

```bash
(streaming-workshop-de-zoomcamp-2026) > docker exec -it streaming-workshop-de-zoomcamp-2026-redpanda-1 rpk version
rpk version: v25.3.9
Git ref:     836b4a36ef6d5121edbb1e68f0f673c2a8a244e2
Build date:  2026 Feb 26 07 48 21 Thu
OS/Arch:     linux/amd64
Go version:  go1.24.3

Redpanda Cluster
  node-1  v25.3.9 - 836b4a36ef6d5121edbb1e68f0f673c2a8a244e2
```

This means the version of Redpanda I am running is v25.3.9.

## Question 2. Sending data to Redpanda

Create a topic called `green-trips`:

```bash
docker exec -it workshop-redpanda-1 rpk topic create green-trips
```

Now write a producer to send the green taxi data to this topic.

Read the parquet file and keep only these columns:

- `lpep_pickup_datetime`
- `lpep_dropoff_datetime`
- `PULocationID`
- `DOLocationID`
- `passenger_count`
- `trip_distance`
- `tip_amount`
- `total_amount`

Convert each row to a dictionary and send it to the `green-trips` topic.
You'll need to handle the datetime columns - convert them to strings
before serializing to JSON.

Measure the time it takes to send the entire dataset and flush:

```python
from time import time

t0 = time()

# send all rows ...

producer.flush()

t1 = time()
print(f'took {(t1 - t0):.2f} seconds')
```

How long did it take to send the data?

- 10 seconds
- 60 seconds
- 120 seconds
- 300 seconds

### Question 2 Answer

ANSWER:
- **10 seconds**

Adding the entire code used to bring the parquet into Redpanda would prove cumbersome. I've added the notebook I used in this repository. Note that there are no outputs as I ran the code within a separate codespace as opposed to this repository. The code I used to send the data along with the output is here:

```python
from time import time

t0 = time()

topic_name = 'green_trips'

# send all rows to Kafka
for _, row in df.iterrows():
    ride = ride_from_row(row)
    producer.send(topic_name, value=ride)

producer.flush()

t1 = time()
print(f'took {(t1 - t0):.2f} seconds')
```

```bash
took 12.93 seconds
```

Since it took 12.93 seconds, the closest answer is 10 seconds.

## Question 3. Consumer - trip distance

Write a Kafka consumer that reads all messages from the `green-trips` topic
(set `auto_offset_reset='earliest'`).

Count how many trips have a `trip_distance` greater than 5.0 kilometers.

How many trips have `trip_distance` > 5?

- 6506
- 7506
- 8506
- 9506

### Question 3 Answer

ANSWER:
- **8506** trips > 5 miles (not km - must've been a typo in the question)
- **15491** trips > 5 km

In the question, it says to count the number of trips > 5km of distance, but the count of those trips is not within the answers. I did the counts of trips that were greater than 5 km and also the counts of trips that were greater than 5 miles. I also attached the notebook I used to do this in this repository. Again no outputs will be shown in it as they were ran in a codespace on another repo.

```python
# ran KafkaConsumer with consumer_timeout_ms=5000 # stop after 5s w/ no new records to get counts after the stream was done for the parquet we use


num_trips_gt_5km = 0
num_trips_gt_5m = 0

# according to nyc taxi data dictionary, "The elapsed trip distance in miles reported by the taximeter."
# need to turn miles to km before checking

for record in consumer:
    if record.value.trip_distance > 5.0:
        num_trips_gt_5m += 1
    if record.value.trip_distance * 1.60934 > 5.0:
        num_trips_gt_5km += 1
    print(record.value)

print(f"Number of trips greater than 5km: {num_trips_gt_5km}")

print(f"Number of trips greater than 5m: {num_trips_gt_5m}")
```

```bash
...
Ride(lpep_pickup_datetime='2025-10-31 23:23:00', lpep_dropoff_datetime='2025-10-31 23:37:00', PULocationID=195, DOLocationID=33, passenger_count=-1, trip_distance=3.0, tip_amount=0.0, total_amount=19.6)

Number of trips greater than 5km: 15491
Number of trips greater than 5m: 8506
```

## Part 2: PyFlink (Questions 4-6)

For the PyFlink questions, you'll adapt the workshop code to work with
the green taxi data. The key differences from the workshop:

- Topic name: `green-trips` (instead of `rides`)
- Datetime columns use `lpep_` prefix (instead of `tpep_`)
- You'll need to handle timestamps as strings (not epoch milliseconds)

You can convert string timestamps to Flink timestamps in your source DDL:

```sql
lpep_pickup_datetime VARCHAR,
event_timestamp AS TO_TIMESTAMP(lpep_pickup_datetime, 'yyyy-MM-dd HH:mm:ss'),
WATERMARK FOR event_timestamp AS event_timestamp - INTERVAL '5' SECOND
```

Before running the Flink jobs, create the necessary PostgreSQL tables
for your results.

Important notes for the Flink jobs:

- Place your job files in `workshop/src/job/` - this directory is
  mounted into the Flink containers at `/opt/src/job/`
- Submit jobs with:
  `docker exec -it workshop-jobmanager-1 flink run -py /opt/src/job/your_job.py`
- The `green-trips` topic has 1 partition, so set parallelism to 1
  in your Flink jobs (`env.set_parallelism(1)`). With higher parallelism,
  idle consumer subtasks prevent the watermark from advancing.
- Flink streaming jobs run continuously. Let the job run for a minute
  or two until results appear in PostgreSQL, then query the results.
  You can cancel the job from the Flink UI at http://localhost:8081
- If you sent data to the topic multiple times, delete and recreate
  the topic to avoid duplicates:
  `docker exec -it workshop-redpanda-1 rpk topic delete green-trips`


## Question 4. Tumbling window - pickup location

Create a Flink job that reads from `green-trips` and uses a 5-minute
tumbling window to count trips per `PULocationID`.

Write the results to a PostgreSQL table with columns:
`window_start`, `PULocationID`, `num_trips`.

After the job processes all data, query the results:

```sql
SELECT PULocationID, num_trips
FROM <your_table>
ORDER BY num_trips DESC
LIMIT 3;
```

Which `PULocationID` had the most trips in a single 5-minute window?

- 42
- 74
- 75
- 166


## Question 5. Session window - longest streak

Create another Flink job that uses a session window with a 5-minute gap
on `PULocationID`, using `lpep_pickup_datetime` as the event time
with a 5-second watermark tolerance.

A session window groups events that arrive within 5 minutes of each other.
When there's a gap of more than 5 minutes, the window closes.

Write the results to a PostgreSQL table and find the `PULocationID`
with the longest session (most trips in a single session).

How many trips were in the longest session?

- 12
- 31
- 51
- 81


## Question 6. Tumbling window - largest tip

Create a Flink job that uses a 1-hour tumbling window to compute the
total `tip_amount` per hour (across all locations).

Which hour had the highest total tip amount?

- 2025-10-01 18:00:00
- 2025-10-16 18:00:00
- 2025-10-22 08:00:00
- 2025-10-30 16:00:00


## Submitting the solutions

- Form for submitting: https://courses.datatalks.club/de-zoomcamp-2026/homework/hw7


## Learning in public

We encourage everyone to share what they learned.
Read more about the benefits [here](https://alexeyondata.substack.com/p/benefits-of-learning-in-public-and).

## Example post for LinkedIn

```
Week 7 of Data Engineering Zoomcamp by @DataTalksClub complete!

Just finished Module 7 - Streaming with PyFlink. Learned how to:

- Set up Redpanda as a Kafka replacement
- Build Kafka producers and consumers in Python
- Create tumbling and session windows in Flink
- Analyze real-time taxi trip data with stream processing

Here's my homework solution: <LINK>

You can sign up here: https://github.com/DataTalksClub/data-engineering-zoomcamp/
```

## Example post for Twitter/X

```
Module 7 of Data Engineering Zoomcamp done!

- Kafka producers and consumers
- PyFlink tumbling and session windows
- Real-time taxi data analysis
- Redpanda as Kafka replacement

My solution: <LINK>

Free course by @DataTalksClub: https://github.com/DataTalksClub/data-engineering-zoomcamp/
```