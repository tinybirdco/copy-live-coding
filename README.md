# Deduplicating OLAP with scheduled data snapshots - Live Coding Session

OLAP databases are optimized for fast ingestion and bulk processing, so it is always better to transform update/delete scenarios into append + deduplicate.

Duplicates happen when:

- Upserts
- Constant ingestion: lots of rows inserted at different times with the same primary key

Strategies to deduplicate:

1. At query time.

1. Using a ReplacingMergeTree.

1. Using Materialized Views.

1. Using Snapshots. -> Covered in this repo and Live Coding Session.

See [Deduplication Strategies in ClickHouse](https://www.tinybird.co/docs/guides/deduplication-strategies.html) for more details.

## Scheduled Data Copy API

[Docs](https://www.tinybird.co/docs/api-reference/pipe-api.html#scheduled-data-copy-beta)

We will showchase how to create a copy pipe without scheduling:

```bash
curl -X POST \
    -H "Authorization: Bearer $TOKEN" \
        "$HOST/v0/pipes/:pipe/nodes/:node/copy" \
    -d "target_datasource=:destination_datasource"
```

And how to run a one time copy:

```bash
curl -X POST \
    -H "Authorization: Bearer $TOKEN" \
        "$HOST/v0/pipes/:pipe/copy" \
```

And then the most common use case, that is creating a Scheduled Data Copy:

```bash
curl -X POST \
    -H "Authorization: Bearer $TOKEN" \
        "$HOST/v0/pipes/:pipe/nodes/:node/copy" \
    -d "target_datasource=:destination_datasource" \
    -d "schedule_cron=*/15 * * * *"
```

### Note on variables

We are using `$TOKEN` and `$HOST` in the script.

```env
TOKEN = <PIPE:CREATE> and <DATASOURCES:APPEND:destination_datasource> token
HOST = https://api.tinybird.co | https://api.us-east.tinybird.co
```

To speed up the testing you can authenticate with `tb auth` and then run

```bash
TOKEN=$(cat .tinyb | jq .token | sed 's/"//g') 
HOST=$(cat .tinyb | jq .host | sed 's/"//g') 
```

to save the token and host as environment variables.

## Example 1: Changing a SK

Due to the queries we want to run —checking the status of a flight— it makes more sense having flight_number as the first value in the Sorting Key:

```diff
- ENGINE_SORTING_KEY "timestamp, meal_choice, priority_boarding, transaction_id"
+ ENGINE_SORTING_KEY "flight_number, timestamp"
```

Let's push that new Data Source.

```bash
cd data-project
cp datasources/flight_reservations.datasource datasources/flight_reservations_sk.datasource
sed -i '' 's/ENGINE_SORTING_KEY "timestamp, meal_choice, priority_boarding, transaction_id"/ENGINE_SORTING_KEY "flight_number, timestamp"/g' data-project/datasources/flight_reservations_sk.datasource
tb push datasources/flight_reservations_sk.datasource
```

Let's add a new node with a `SELECT * FROM flight_reservations`

```bash
curl -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "$HOST/v0/pipes" \
    -d '{
        "name":"change_sk",
        "description": "pipe to copy data",
        "nodes": [
            {"sql": "SELECT * FROM flight_reservations", "name": "copy_node", "description": "just a select * to get all rows" }
        ]
    }'
```

And now make it a copy node pointing to `flight_reservations_sk`:

```bash
curl -X POST \
    -H "Authorization: Bearer $TOKEN" \
        "$HOST/v0/pipes/change_sk/nodes/copy_node/copy" \
    -d "target_datasource=flight_reservations_sk"
```

Run a one time copy:

```bash
curl -X POST \
    -H "Authorization: Bearer $TOKEN" \
        "$HOST/v0/pipes/change_sk/copy" \
```

## Example 2: Scheduling snapshots
