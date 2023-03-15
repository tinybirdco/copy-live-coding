# Deduplicating OLAP with scheduled data snapshots - Live Coding Session

OLAP databases are optimized for fast ingestion and bulk processing, so it is always better to transform update/delete scenarios into append + deduplicate ones.

When duplicates happen in analytical databases:

- When doing upserts: this is done in ClickHouse tipically using a [ReplacingMergeTree](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replacingmergetree/).
- When having constant ingestion: lots of rows inserted at different times with the same primary key

Strategies to deduplicate:

1. At query time.

1. Using a ReplacingMergeTree.

1. Using Materialized Views.

1. Using Snapshots. -> Covered in this repo and Live Coding Session.

See our guide about [deduplication Strategies in ClickHouse](https://www.tinybird.co/docs/guides/deduplication-strategies.html) for more details.

## Scheduled Data Copy API

[Docs](https://www.tinybird.co/docs/api-reference/pipe-api.html#data-copy-header)

We will showcase how to _create a copy pipe_ without scheduling:

```bash
curl -X POST \
    -H "Authorization: Bearer $TOKEN" \
        "$HOST/v0/pipes/:pipe/nodes/:node/copy" \
    -d "target_datasource=:destination_datasource"
```

to run an _on-demand copy_:

```bash
curl -X POST \
    -H "Authorization: Bearer $TOKEN" \
        "$HOST/v0/pipes/:pipe/copy"
```

And then the most common use case, that is creating a _Scheduled Data Copy_:

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

To speed up the testing you can authenticate with `tb auth`[^1] and then run

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

Run an on-demand copy:

```bash
curl -X POST \
    -H "Authorization: Bearer $TOKEN" \
        "$HOST/v0/pipes/change_sk/copy"
```

We can compare the performance improvement querying our `status-query` endpoint.

## Example 2: Scheduled snapshots

For the shake of the demo we will perform a snapshot every minute, leaving the full picture of the reservations in the `flight_snapshots` Data Source. We identify each snapshot with a DateTime column called `snapshot_id`.

To do so, we will use the pipe we have prepared, `deduplicate.pipe`[^2], and convert it into a scheduled copy pipe.

```bash
curl -X POST \
    -H "Authorization: Bearer $TOKEN" \
        "$HOST/v0/pipes/deduplicate/nodes/all_together/copy" \
    -d "target_datasource=flight_snapshots" \
    -d "schedule_cron=*/1 * * * *"
```

Lastly we can create a new query calling the latest snapshot —or a lambda architecture like, calling the latest snapshot and deduplicating as well the rows that changed from the latest snapshot until now— and see the differences in performance.

[^1]: Although we use mostly use the API in this example, we have included some [CLI](https://www.tinybird.co/docs/cli.html) commands for convenience. [Check Getting started with the CLI](https://www.tinybird.co/docs/quick-start-cli.html) in case you're not yet familiar with it.

[^2]: Be sure that the `deduplicate` pipe is not published as API endpoint. If you did a `tb push`, you can do `tb pipe unpublish deduplicate` before converting it into a copy pipe.
