# Real time Crypto Trade ETL pipeline

The project is a real-time data pipeline that accepts streaming data from binance using its [Websocket Streams](https://github.com/binance/binance-spot-api-docs/blob/master/web-socket-streams.md) and streams the `@trade` stream. 
we combine the streams and access `BTCUSDT`, `ETHUSDT`, and `BNBUSDT` symbols. 

The entire pipeline is dockerized. 

The data is stored in postgres
1. `raw_trades` -> stores raw json from binance and additional timestamps.
2. `metrics` -> stores processed windowed metrics. 

---

The flow is as follows

![Basic workflow](image-1.png)

---
### Tech used

- Python 3.12
- Apache Kafka (Kraft)
- Apache Spark
- Postgres
- Kafdrop
- Docker and Docker Compose
- [***uv***](https://github.com/astral-sh/uv) (for python environment management)

```
Folder
│   .gitignore
│   .python-version
│   compare.py
│   main.py
│   pyproject.toml
│   README.md
│   uv.lock
│
├───docker
│   │   docker-compose.yml
│   │
│   └───scripts
│           create-topics.sh
│
├───ingestion
│       producer.py
│       websocket_client.py
│
├───postgres
│       create_metrics.sql
│       create_raw_trades.sql
│
└───spark
        spark_streaming.py
```

## How to run

### Prerequisites
Ensure you have Python 3.12 \
Docker and Docker Compose \
[uv](https://github.com/astral-sh/uv)

### Setup python environment.
From the project root folder \
`uv sync` \
this will create the virtual environments and install dependencies form `pyproject.toml` \
activate the environment by running in root\
`.venv\Scripts\activate` \
and deactivate by \
`.venv\Scripts\deactivate` 

usually simply creating a new terminal in vs code automatically activates the venv (ctrl + shift + `) to start a new terminal

---
### docker and producer.py

make sure you are in root folder \
`cd docker` -> `docker compose up -d`

the `docker/scripts/create-topics.sh` bash file runs by itself after 10 seconds to ensure all the kafka and docker instances are running fully.

run `producer.py` after 10 seconds once running `docker compose kafka-init logs` shows `Topic setup complete.`

when in root folder, run \
`cd ingestion` \
`uv run python producer.py`

---
the tables are created using docker compose, spark jdbc doesnt create the table anymore, it only appends to it.

to verify the data in postgres, run \
`docker exec -it postgres psql -U spark -d crypto` 

`SELECT * FROM raw_trades ORDER BY trade_time DESC LIMIT 5;` to check the raw_trades table 

`SELECT * FROM metrics ORDER BY window_start DESC LIMIT 5;` to check the metrics table

---

this is just the basic pipeline. 
stuff thats still left to do includes 
- some visualization (grafana possibly)
-  maybe some ml
- db table creation in a way that jdbc isnt used to ensure  
- refine it, figure out the kinks, maybe add checkpoints. i might make it an etl pipeline analytics... like getting the analytics of the pipeline itself? idk
- maybe increase the number of producers to include kline or tickers or aggtrade. 

---

The main idea of this project was to **learn**. I wanted to understand how `kafka` works, how `pyspark` works. 

This is just a basic pipeline. it does not handle errors as of now, i will be working on it. i believe it still has some errors.

---

crypto db \
`raw_trades` table

```
crypto=# SELECT * FROM raw_trades order by trade_time desc limit 5;
 symbol  |  trade_id  |  price   | quantity | is_buyer_maker |         event_time         |         trade_time         |       ingestion_time       |         spark_time         | exchange_latency_ms | source_latency_ms | end_to_end_latency_ms
---------+------------+----------+----------+----------------+----------------------------+----------------------------+----------------------------+----------------------------+---------------------+-------------------+-----------------------
 BNBUSDT | 1368320328 |   860.12 |     0.01 | t              | 2025-12-30 14:19:27.054+00 | 2025-12-30 14:19:27.054+00 | 2025-12-30 14:19:27.138+00 | 2025-12-30 14:19:27.293+00 |                  84 |                84 |                   155
 BTCUSDT | 5718795347 | 88114.05 |    9e-05 | t              | 2025-12-30 14:19:27.029+00 | 2025-12-30 14:19:27.028+00 | 2025-12-30 14:19:27.11+00  | 2025-12-30 14:19:27.293+00 |                  82 |                81 |                   183
 BTCUSDT | 5718795346 | 88114.06 |  0.00021 | f              | 2025-12-30 14:19:26.713+00 | 2025-12-30 14:19:26.713+00 | 2025-12-30 14:19:26.794+00 | 2025-12-30 14:19:27.293+00 |                  81 |                81 |                   499
 BNBUSDT | 1368320327 |   860.12 |    0.017 | t              | 2025-12-30 14:19:26.634+00 | 2025-12-30 14:19:26.633+00 | 2025-12-30 14:19:26.714+00 | 2025-12-30 14:19:27.293+00 |                  81 |                80 |                   579
 BNBUSDT | 1368320326 |   860.13 |    0.017 | f              | 2025-12-30 14:19:26.566+00 | 2025-12-30 14:19:26.566+00 | 2025-12-30 14:19:26.647+00 | 2025-12-30 14:19:27.293+00 |                  81 |                81 |                   646
(5 rows)
```
crypto db \
`metrics` table 
```
select * from metrics order by avg_exchange_latency_ms desc limit 5;
 symbol  |      window_start      |       window_end       | trade_count |    total_volume    |     avg_price      | min_price | max_price |     price_stddev     |        vwap        |     buy_volume      |    sell_volume    |    buy_sell_ratio    | trade_intensity |   liquidity_proxy   | avg_exchange_latency_ms | avg_source_latency_ms | avg_end_to_end_latency_ms
---------+------------------------+------------------------+-------------+--------------------+--------------------+-----------+-----------+----------------------+--------------------+---------------------+-------------------+----------------------+-----------------+---------------------+-------------------------+-----------------------+---------------------------
 BTCUSDT | 2025-12-30 14:25:30+00 | 2025-12-30 14:25:35+00 |         219 | 0.4748599999999996 |  88182.53479452053 |   88178.7 |  88188.17 |   3.1742025407134453 |  88182.11787874337 |  0.4261599999999996 |            0.0487 |    8.750718685831615 |            43.8 | 0.09497199999999992 |       144.4840182648402 |    143.90867579908675 |         578.6666666666666
 ETHUSDT | 2025-12-30 14:25:10+00 | 2025-12-30 14:25:15+00 |        1187 | 31.454499999999996 | 2976.1526116259442 |   2975.48 |   2976.62 |   0.3007588827129864 | 2976.1620891128573 |   4.290800000000001 |           27.1637 |  0.15796080799007503 |           237.4 |   6.290899999999999 |      135.56360572872788 |    134.78011794439763 |        1888.0379106992418
 BTCUSDT | 2025-12-30 14:25:05+00 | 2025-12-30 14:25:10+00 |         658 | 1.5268700000000026 |   88168.3994832826 |     88160 |  88179.38 |    7.464012413926943 |  88165.15815262595 |  1.0239900000000002 | 0.502880000000003 |   2.0362511931275735 |           131.6 | 0.30537400000000053 |       127.2917933130699 |    126.49544072948328 |         1266.629179331307
 ETHUSDT | 2025-12-30 14:25:25+00 | 2025-12-30 14:25:30+00 |         559 |  27.72209999999981 |  2976.450769230776 |   2976.15 |   2976.73 |   0.1837783905390691 |  2976.415193654179 |              1.3936 | 26.32849999999983 | 0.052931234213875036 |           111.8 |  5.5444199999999615 |       123.5474060822898 |    122.48121645796064 |         896.0304114490161
 BNBUSDT | 2025-12-30 14:26:00+00 | 2025-12-30 14:26:05+00 |          87 |  5.094999999999993 |  859.9150574712637 |    859.87 |    859.98 | 0.038515281268610375 |  859.9589715407268 | 0.29900000000000004 | 4.795999999999993 |  0.06234361968306932 |            17.4 |  1.0189999999999986 |      122.16091954022988 |    121.74712643678161 |        1739.6436781609195
(5 rows)
```