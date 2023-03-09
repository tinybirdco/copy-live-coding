# Copy API Live Coding Session

## Scheduled Data Copy API

[Docs](https://www.tinybird.co/docs/api-reference/pipe-api.html#scheduled-data-copy-beta)

We will showchase how to create a copy pipe without scheduling:

```bash
# TOKEN = <PIPE:CREATE> and <DATASOURCES:APPEND:destination_datasource> token
# HOST = https://api.tinybird.co | https://api.us-east.tinybird.co

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

## Example 1: Changing a SK

Due to the queries we want to run —checking the status of a flight— it makes more sense having flight_number as the first value in the Sorting Key:

```diff
- s/ENGINE_SORTING_KEY "timestamp, transaction_id, flight_number"
+ ENGINE_SORTING_KEY "flight_number, timestamp"
```

Let's push that new Data Source.

```bash
cd data-project
cp datasources/flight_transactions.datasource datasources/flight_transactions_SK.datasource
sed -i '' 's/ENGINE_SORTING_KEY "timestamp, transaction_id, flight_number"/ENGINE_SORTING_KEY "flight_number, timestamp"/g' data-project/datasources/flight_transactions_SK.datasource
tb push datasources/flight_reservations_SK.datasource
```

```bash
# TOKEN = <PIPE:CREATE token> and <DATASOURCES:APPEND:destination_datasource> 
# HOST = https://api.tinybird.co | https://api.us-east.tinybird.co

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

## Example 2: Scheduling snapshots

```bash
tb push datasources/flight_snapshots.datasource
```
