# pgreplay-go [![CircleCI](https://circleci.com/gh/gocardless/pgreplay-go.svg?style=svg&circle-token=d020aaec823388b8e4debe552960450402964ae7)](https://circleci.com/gh/gocardless/pgreplay-go)

> See a discussion of building this tool at https://blog.lawrencejones.dev/building-a-postgresql-load-tester/

This tool is a different take on the existing [pgreplay](
https://github.com/laurenz/pgreplay) project. Where pgreplay will playback
Postgres log files while respecting relative chronological order, pgreplay-go
plays back statements at approximately the same rate they were originally sent
to the database.

When benchmarking database performance, it is better for a replay tool to
continue sending traffic than to halt execution until a strangling query has
complete. If your new cluster isn't performing, users won't politely wait for
your service to catch-up before issuing new commands.

## Benchmark strategy

You have an existing cluster and want to trial new hardware/validate
configuration changes/move infrastructure providers. Your production services
depend on ensuring the new change is safe and doesn't degrade performance.

### 1. Configure source RDS logging

First capture the logs from your running Postgres instance.

Set these parametes on your RDS database:

```
log_destination = csvlog
log_connections = 1
log_disconnections = 1
log_min_error_statement = log
log_min_messages = error
log_statement = all
log_min_duration_statement = 0
```

And then press apply and wait a couple minutes.

### 2. pgreplay-go against copy of production cluster

Now create a copy of the original production cluster using the snapshot from
(2). The aim is to have a cluster that exactly replicates production, providing
a reliable control for our experiment.

The goal of this run will be to output Postgres logs that can be parsed by
[pgBadger](https://github.com/darold/pgbadger) to provide an analysis of the
benchmark run. See the pgBadger readme for details, or apply the following
configuration for defaults that will work:

```sql
ALTER SYSTEM SET log_min_duration_statement = 0;
ALTER SYSTEM SET log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
ALTER SYSTEM SET log_checkpoints = on;
ALTER SYSTEM SET log_connections = on;
ALTER SYSTEM SET log_disconnections = on;
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET log_temp_files = 0;
ALTER SYSTEM SET log_autovacuum_min_duration = 0;
ALTER SYSTEM SET log_error_verbosity = default;
SELECT pg_reload_conf();
```

Run the benchmark using the pgreplay-go binary from the primary Postgres machine
under test. pgreplay-go requires minimal system resource and is unlikely to
affect the benchmark when running on the same machine, though you can run it
from an alternative location if this is a concern. An example run would look
like this:

```
$ pgreplay \
    --errlog-file /data/postgresql-filtered.log \
    --host 127.0.0.1 \
    --metrics-address 0.0.0.0 \
    --start <benchmark-start-time> \
    --finish <benchmark-finish-time>
```

If you run Prometheus then pgreplay-go exposes a metrics that can be used to
report progress on the benchmark. See [Observability](#observability) for more
details.

Once the benchmark is complete, store the logs somewhere for safekeeping. We
usually upload the logs to Google cloud storage, though you should use whatever
service you are most familiar with.

### 5. pgreplay-go against new cluster

After provisioning your new candidate cluster, with the hardware/software
changes you wish to test, we repeat step (4) for our new cluster. You should use
exactly the same pgreplay logs and run the benchmark for the same time-window.

As in step (4), upload your logs in preparation for the next step.

### 6. User pgBadger to compare performance

We use pgBadger to perform analysis of our performance during the benchmark,
along with taking measurements from the
[node_exporter](https://github.com/prometheus/node_exporter) and
[postgres_exporter](https://github.com/rnaveiras/postgres_exporter) running
during the experiment.

pgBadger reports can be used to calculate a query duration histogram for both
clusters - these can indicate general speed-up/degradation. Digging into
specific queries and the worst performers can provide more insight into which
type of queries have degraded, and explaining those query plans on your clusters
can help indicate what might have caused the change.

This is the least prescriptive part of our experiment, and answering whether the
performance changes are acceptable - and what they may be - will depend on your
knowledge of the applications using your database. We've found pgBadger to
provide sufficient detail for us to be confident in answering this question, and
hope you do too.

## Observability

Running benchmarks can be a long process. pgreplay-go provides Prometheus
metrics that can help determine how far through the benchmark has progressed,
along with estimating how long remains.

Hooking these metrics into a Grafana dashboard can give the following output:

![pgreplay-go Grafana dashboard](res/grafana.jpg)

We'd suggest that you integrate these panels into your own dashboard; for
example by showing them alongside key PostgreSQL metrics such as transaction
commits, active backends and buffer cache hit ratio, as well as node-level
metrics to show CPU and IO saturation.

A sample dashboard with the `pgreplay-go`-specific panels has been provided that
may help you get started. Import it into your Grafana dashboard by downloading
the [dashboard JSON file](res/grafana-dashboard-pgreplay-go.json).

## Types of Log

### Simple

This is a basic query that is executed directly against Postgres with no
other steps.

```
2010-12-31 10:59:52.243 UTC|postgres|postgres|4d1db7a8.4227|LOG:  statement: set client_encoding to 'LATIN9'
```

### Prepared statement

```
2010-12-31 10:59:57.870 UTC|postgres|postgres|4d1db7a8.4227|LOG:  execute einf"ug: INSERT INTO runtest (id, c, t, b) VALUES ($1, $2, $3, $4)
2010-12-31 10:59:57.870 UTC|postgres|postgres|4d1db7a8.4227|DETAIL:  parameters: $1 = '6', $2 = 'mit    Tabulator', $3 = '2050-03-31 22:00:00+00', $4 = NULL
```
