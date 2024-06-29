# Ruby on Rails 100M bank transactions import & reporting

## Instructions

Design & implement a system that can:

- load a data file containing at least 100M financial transactions
- report the minimum, maximum, and average dollar amounts per bank branch.

The data file lines would contain a timestamp, a branch code, and amount.

## Solution

Assumptions:

- we are meant to report dollar amounts per bank branch per month
- the CSV processing is either a one-time or occational job (if it was meant for a long-term usage, we would use at least an MVC pattern, split the database calls into services/providers)
- the solution can be simplyfied for the sake of time spent on this take-home assignment (see TODOs section for next enhancements that can be made)
- there is a single bank branch in each of the Canadian provinces, hence the branch codes AB, BC, ON, etc.

Solution:

- Ruby on Rails backend that:
  - generates the input CSV with 100M rows
  - processes the CSV into transactions and reports using asynchronous processing and ability to scale
  - provides a REST API endpoint to fetch the reports (this is used by our frontend and also can be used as a PowerBI data source)
- React.js frontend that:
  - fetches the reports from the backend
  - displays the reports in a chart
  - allows to view different months and compare different branches
- performance metrics of the Rails app (see Usage part of this file)

Note: This repository and it's approach and current state of the functinoality is not a production-ready application.
It was created as a base to build on top of. The list of things needed to be done to make the application
production ready can be found at the bottom of this file in the TODOs section.

## Usage

```bash
# start project
docker compose up -d

# clear everything from the Sidekiq's queue
docker compose exec redis redis-cli FLUSHALL

# create a database backup
docker compose exec postgres pg_dump -U rails -d rails > ./postgres/init/rails.sql

# generate csv file with 100M randomized transactions
# measure the duration
#
# on the authors development machine, the execution time was as follows:
# - 1K rows: ~0.7 second
# - 1M rows: ~2.7 seconds
# - 100M rows: ~3.15 minutes
#
# for storage reasons, only the 1000_transactions.csv file is included in this git repository
# the 1M file is 21MB and 100M file is 2GB
time docker compose exec rails bin/rails 'generate_transactions_csv[100000000]'

# feed transaction from the CSV into Sidekiq queue
#
# on the authors development machine, the execution time was as follows:
# - 1K rows: ~0.7 second
# - 1M rows: ~5.1 seconds
# - 100M rows: ~7.25 minutes
time docker compose exec rails bin/rails 'feed_transactions_to_sidekiq[100000000_transactions.csv]'

# process Sidekiq queue
#
# note: there is a separate Sidekiq container that is processing anything put into the queue right away
# this command is here for the performance measures
#
# on the authors development machine, the execution time was as follows:
# - 1K items in the queue: <1s second
# - 1M items in the queue: ~15 seconds
# - 100M items in the queue: ~33.5 minutes
time docker compose exec rails bundle exec sidekiq

# generate reports from imported transactions
#
# on the authors development machine, the execution time was as follows:
# - 1K items in the queue: <1s second
# - 1M items in the queue: ~1.3 second
# - 100M items in the queue: ~27.3 seconds
time docker compose exec rails bin/rails generate_transaction_aggregations

# fetch the report for a given month
curl -sS http://localhost:3000/api/report/2024-01-01 | jq

# run everything for 100K transactions in a single command as follows:
# reset queue, database, run csv generation, feed sidekiq, process queue in a single command
clear && \
    docker compose down && \
    docker compose up -d postgres && \
    docker compose exec postgres psql -d postgres://rails:znsoorcM9pGb@localhost:5432 -c 'TRUNCATE transactions; TRUNCATE reports;' && \
    docker compose up -d redis && \
    docker compose exec redis redis-cli FLUSHALL && \
    docker compose up -d && \
    docker compose exec rails bin/rails 'generate_transactions_csv[100000]' && \
    docker compose exec rails bin/rails 'feed_transactions_to_sidekiq[100000_transactions.csv]' && \
    docker compose logs -f sidekiq
# wait for Sidekiq to finish and then run:
docker compose exec rails bin/rails generate_transaction_aggregations
```

### URLs

- Web http://localhost:3000
- Adminer http://localhost:8082/?server=postgres&username=rails&db=rails

### TODOs / Possible enhancements

- add static code analysis / linter (Rubocop)
- tests for CSV parsing and import
- add an authentication layer to both frontend and backend
- split one large CSV into multiple files first, and then feed the files to Sidekiq queue in parallel
- add ability to skip first N rows from the CSV when feeding the data to Sidekiq queue (in case we need to re-run a portion of a large csv)
- scale the sidekiq queue workers
- right now the app is structured for an occasional imports. if we need to import and process the transactions continuosly, we could:
  - create a scheduled Sidekiq job for each of the rakes that we are now running manually
  - feed transactions to the queue in bulk using Redis protocol and in the format that is used by Sidekiq
  - create the aggregations (Reports) from a read replica
- PowerBI adapter
