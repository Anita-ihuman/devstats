# GitHub Archives Visualization

Author: Łukasz Gryglicki <lukaszgryglick@o2.pl>

This is a toolset to visualize GitHub [archives](https://www.githubarchive.org/) using Grafana dashboards.

# Goal

We want to create a toolset for visualizing various metrics for the Kubernetes community.
Everything is open source so that it can be used by other CNCF and non-CNCF open source projects.
The only requirement is that project must be hosted on a public GitHub repositor/repositories.
The work builds on the [Velodrome](https://github.com/kubernetes/test-infra/tree/master/velodrome) tool that was built by [apelisse](https://github.com/apelisse) and others.

This project aims to add new metrics for the existing Grafana dashboards.
We want to support all kind of metrics, including historical ones.
Please see [requested metrics](https://docs.google.com/document/d/1o5ncrY6lVX3qSNJGWtJXx2aAC2MEqSjnML4VJDrNpmE/edit?usp=sharing) to see what kind of metrics are needed.
Many of them cannot be computed based on the data sources currently used.

The current Velodrome implementation uses the GitHub API to get its data. This has some limitations:
- It is not able to get repo and PR state at any given point of history
- It is limited by GitHub API token rate limits.

# GitHub Archives

Our approach is to use GitHub archives instead. The possible alternatives are:

1) BigQuery:
- You can query any data you want, but the structure is quite flat and entire GitHub event payloads are stored as a single column containing JSON text.
- This limits usage due to the need of parsing that JSON in DB queries.
- BigQuery is commercial, paid and is quite expensive.
- It is not a standard SQL.

2) GitHub API:
- You can get the current state of the objects, but you cannot get repo, PR, issue state in the past (for example summary fields, etc).
- It is limited by GitHub API usage per hour, which makes local development harder.
- API limits are very aggressive for unauthorized access, and even with authorized access, you're limited to 5000 API calls/hour.
- It is much slower than processing GitHub archives or BigQuery.
- You must query it via API and it is returning a single result.
- You can use GitHub hook callbacks, but they only fire for current events.

3) GitHub archives
- All GitHub events are packed into multi-json gzipped files each hour and made available from [Github Archive](https://www.githubarchive.org/). To use this data, you need to extract all hours (since the Kubernetes project started) and filter out all data except for events from the 3 kubernetes organizations ([kubernetes](https://github.com/kubernetes), [kubernetes-incubator](https://github.com/kubernetes-incubator), and [kubernetes-client](https://github.com/kubernetes-client).
- This is a lot of data to process, but you have all possible GitHub events in the past, processing more than 3 years of this data takes about 2-2,5 hours, but this must only be done once and then the processed results are available for other's use.
- You have a lot of data in a single file, that can be processed/filtered in memory.
- You are getting all possible events, and all of them include the current state of PRs, issues, repos at given point in time.
- Processing of GitHub archives is free, so local development is easy.
- GitHub archives format changed in 2015-01-01, so it is using older format (pre-2015) before that date, and newer after. For details please see [USAGE.md](https://github.com/cncf/gha2db/blob/master/USAGE.md), specially `GHA2DB_OLDFMT` environment variable.
- I have 1.2M events in my Psql database, and each event contains quite complex structure, I would estimate about 3-6 GitHub API calls are needed to get that data. It means about 7M API calls.
- 7.2M / 5K (API limit per hour) gives 1440 hours which is 2 months. And we're on GitHub API limit all the time. Processing ALL GitHub events takes about 2 hours without ANY limit.
- You can optionally save downloaded JSONs to avoid network traffic in next calls (also usable for local development mode).
- There is an already implemented version in Go, please see usage here [USAGE](https://github.com/cncf/gha2db/blob/master/USAGE.md)
- Dashboards can be displayed here [link](https://cncftest.io/?orgId=1)

# Proposed architecture

Velodrome consists of 3 parts:
- `Fetcher` - it is used to query GitHub API and store results in MySQL database (but only a small part of data available is stored)
- `Transform` - it is used to compute some metrics on MySQL database and save them as InfluxDB series
- `Grafana` - displays data from InfluxDB time series.

Our architecture is quite similar, but we're getting all possible GitHub data for all objects, and all objects historical state as well. It consists of:

1) `structure` (manages database structure, summaries, views)
- [structure](https://github.com/cncf/gha2db/blob/master/cmd/structure/structure.go)
- It is used to create database structure, indexes and to update database summary tables, views etc.
- Ruby version supported both MySQL and Postgres. Tests have shown that Postgres is a way better than MySQL for this.
- Go version only supports Postgres. Ruby version is removed and no longer maintained.
- Postgres supports hash joins that allows multi-million table joins in less than 1s, while MySQL requires more than 3 minutes. MySQL had to use data duplication in multiple tables to create fast metrics.
- Postgres has built-in fast REGEXP extract & match, while MySQL only has slow REGEXP match and no REGEXP extract, requiring external libraries like `lib_mysql_pcre` to be installed.
- Postgres supports materialized views - so complex metrics can be stored by such views, and we only need to refresh them when syncing data. MySQL requires creating an additional table and managing it.
- MySQL has utf8 related issues, I've found finally workaround that requires to use `utf8mb4` and do some additional `mysqld` configuration.

2) `gha2db` (imports GitHub archives to database and eventually JSON files)
- [gha2db](https://github.com/cncf/gha2db/blob/master/cmd/gha2db/gha2db.go)
- This is a `fetcher` equivalent, differences are that it reads from GitHub archive instead of GitHub API and writes to Postgres instead of MySQL
- It saves ALL data from GitHub archives, so we have all GitHub structures fully populated. See [Database structure](https://github.com/cncf/gha2db/blob/master/USAGE.md).
- We have all historical data from all possible GitHub events and summary values for repositories at given points of time.
- The idea is to divide all data into two categories: `const` and `variable`. Const data is a data that is not changing in time, variable data is a data that changes in time, so `event_id` is added as a part of this data primary key.
- Table structure, `const` and `variable` description can be found in [USAGE](https://github.com/cncf/gha2db/blob/master/USAGE.md)
- The program can be parallelized very easy (events are distinct in different hours, so each hour can be processed by other CPU), uses 48 CPUs on cncftest.io.

3) `db2influx` (computes metrics given as SQL files to be run on Postgres and saves time series output to InfluxDB)
- [db2influx](https://github.com/cncf/gha2db/blob/master/cmd/db2influx/db2influx.go)
- This separates metrics complex logic in SQL files, `db2influx` executes parameterized SQL files and write final time-series to InfluxDB.
- Parameters are `'{{from}}'`, `'{{to}}'` to allow computing the given metric for any date period.
- This means that InfluxDB will only hold multiple time-series (very simple data). InfluxDB is extremely good at manipulating such kind of data - this is what it was created for.
- Grafana will read from InfluxDB by default and will use its power to generate all possible aggregates, minimums, maximums, averages, medians, percentiles, charts etc.
- Adding new metric will mean add Postgres SQL that will compute this metric.

4) `sync` (synchronizes GitHub archive data and Postgres, InfluxDB databases)
- [sync](https://github.com/cncf/gha2db/blob/master/cmd/sync/sync.go)
- This program figures out what is the most recent data in Postgres database then queries GitHub archive from this date to current date.
- It will add data to Postgres database (since the last run)
- It will update summary tables and/or (materialized) views on Postgres DB.
- Then it will call `db2influx` for all defined SQL metrics.
- When adding new metrics, it needs to be called here.
- This tool also supports initial computing of All InfluxDB data (instead of default update since last run).
- It will be called from cron job at least every 45 minutes - GitHub archive publishes new file every hour, so we're off by at most 1 hour.

5) Additional stuff, most important being `runq`  and `import_affs` tools.
- [runq](https://github.com/cncf/gha2db/blob/master/cmd/runq/runq.go)
- `runq` gets SQL file name and parameter values and allows to run metric manually from the command line (this is for local development)
- [import_affs](https://github.com/cncf/gha2db/blob/master/cmd/import_affs/import_affs.go)
- `import_affs` takes one parameter - JSON file name (this is a file from [cncf/gitdm](https://github.com/cncf/gitdm): [github_users.json](https://raw.githubusercontent.com/cncf/gitdm/master/github_users.json)
- This tools imports GitHub user names (in addition to logins from GHA) and creates developers - companies affiliations (that can be used by [Companies velocity](https://cncftest.io/dashboard/db/companies-velocity?orgId=1) metric)
- [z2influx](https://github.com/cncf/gha2db/blob/master/cmd/z2influx/z2influx.go)
- `z2influx` is used to fill gaps that can occur for metrics that returns multiple columns and rows, but number of rows depends on date range, it uses [gaps.yaml](https://github.com/cncf/gha2db/blob/master/metrics/gaps.yaml) file to define which metrics should be zero filled.
- There are few shell scripts for example: running sync every N seconds, setup InfluxDB etc.

Detailed usage is here [USAGE](https://github.com/cncf/gha2db/blob/master/USAGE.md)

# Current Velodrome

This toolset can either replace velodrome or just add value to `velodrome`.

They both can use shared InfluxDB (We are naming series in such a way to avoid conflicts with existing ones).

Then we can just add new dashboards that use my `gha2db`/`db2influx` workflow in the existing Grafana, and add a cron job that will keep them up to date.

# Adding new metrics

To add new metrics we need to:
1) Define parameterized SQL (with `{{from}}` and `{{to}}` params) that returns this metric data.
2) Add `db2influx` call for this metric in `sync` tool. 
3) If metrics create data gaps (for example returns multipe rows with different counts depending on data range), add automatic filling gaps in [gaps.yaml](https://github.com/cncf/gha2db/blob/master/metrics/gaps.yaml) (file is used by `z2influx` tool).
4) Add test coverage in [metrics_test.go](https://github.com/cncf/gha2db/blob/master/metrics_test.go).
5) Add Grafana dashboard or row that displays this metric
6) Export new Grafana dashboard to JSON
7) Create PR for the new metric.
8) Explain how metrics SQLs works in USAGE.md (currently this is pending for all metrics defined so far)

# Local development
Local development is much easier:
1) Install psql, influx, grafana - all with default options.
2) Fetch populated psql database [Postgres database dump](https://cncftest.io/web/k8s.sql.xz)
3) Just run Go tools manually: `structure`, `gha2db`, `db2influx`, `sync`, `runq`, `z2influx`, `import_affs`.
4) Run tests locally, plaese see [testing](https://github.com/cncf/gha2db/blob/master/TESTING.md)

# Database structure details

The main idea is that we divide tables into 2 groups:
- const: meaning that data in this table is not changing in time (is saved once)
- variable: meaning that data in those tables can change between GH events, and GH `event_id` is a part of this tables primary key.
- there are also "compute" tables that are auto-updated by `sync`/`structure` tools and affiliations table that are filled by `import_affs` tool.

List of tables:
- `gha_actors`: const, users table
- `gha_actors_emails`: const, holds one or more email addresses for actors, this is filled by `./import_affs` tool.
- `gha_actors_affiliations`: const, holds one or more company affiliations for actors, this is filled by `./import_affs` tool.
- `gha_assets`: variable, assets
- `gha_branches`: variable, branches data
- `gha_comments`: variable (issue, PR, review)
- `gha_commits`: variable, commits
- `gha_companies`: const, companies, this is filled by `./import_affs` tool.
- `gha_events`: const, single GitHub archive event
- `gha_forkees`: variable, forkee, repo state
- `gha_issues`: variable, issues
- `gha_issues_assignees`: variable, issue assignees
- `gha_issues_labels`: variable, issue labels
- `gha_labels`: const, labels
- `gha_milestones`: variable, milestones
- `gha_orgs`: const, orgs
- `gha_pages`: variable, pages
- `gha_payloads`: const, event payloads
- `gha_pull_requests`: variable, pull requests
- `gha_pull_requests_assignees`: variable pull request assignees
- `gha_pull_requests_requested_reviewers`: variable, pull request requested reviewers
- `gha_releases`: variable, releases
- `gha_releases_assets`: variable, release assets
- `gha_repos`: const, repos
- `gha_teams`: variable, teams
- `gha_teams_repositories`: variable, teams repositories connections
- `gha_logs`: this is a table that holds all tools logs (unless `GHA2DB_SKIPLOG` is set)
- `gha_texts`: this is a compute table, that contains texts from comments, commits, issues and pull requests, updated by sync and structure tools
- `gha_issues_pull_requests`: this is a compute table that contains PRs and issues connections, updated by sync and structure tools
- `gha_issues_events_labels`: this is a compute table, that contains shortcuts to issues labels (for metrics speedup), updated by sync and structure tools

# Grafana dashboards

Here are already working dashboards using this repo.

Each dashboard is defined by its metrics SQL, saved Grafana JSON export and link to dashboard running on <https://cncftest.io>  

1) Reviewers dashboard: [reviewers.sql](https://github.com/cncf/gha2db/blob/master/metrics/reviewers.sql), [reviewers.json](https://github.com/cncf/gha2db/blob/master/grafana/dashboards/reviewers.json), [view](https://cncftest.io/dashboard/db/reviewers?orgId=1).
2) SIG mentions dashboard: [sig_mentions.sql](https://github.com/cncf/gha2db/blob/master/metrics/sig_mentions.sql), [sig_mentions.json](https://github.com/cncf/gha2db/blob/master/grafana/dashboards/sig_mentions.json), [view](https://cncftest.io/dashboard/db/sig-mentions?orgId=1).
3) SIG mentions breakdown by categories dashboard: [sig_mentions_cats.sql](https://github.com/cncf/gha2db/blob/master/metrics/sig_mentions_cats.sql), [sig_mentions_breakdown.sql](https://github.com/cncf/gha2db/blob/master/metrics/sig_mentions_breakdown.sql), [sig_mentions_cats.json](https://github.com/cncf/gha2db/blob/master/grafana/dashboards/sig_mentions_cats.json), [view](https://cncftest.io/dashboard/db/sig-mentions-categories?orgId=1).
4) SIG mentions using labels dashboard: [labels_sig.sql](https://github.com/cncf/gha2db/blob/master/metrics/labels_sig.sql), [labels_kind.sql](https://github.com/cncf/gha2db/blob/master/metrics/labels_kind.sql), [labels_sig_kind.sql](https://github.com/cncf/gha2db/blob/master/metrics/labels_sig_kind.sql), [sig_mentions_labels.json](https://github.com/cncf/gha2db/blob/master/grafana/dashboards/sig_mentions_labels.json), [view](https://cncftest.io/dashboard/db/sig-mentions-using-labels?orgId=1).
5) Number of all PRs merged in all Kubernetes repos, from 2014-06 dashboard [all_prs_merged.sql](https://github.com/cncf/gha2db/blob/master/metrics/all_prs_merged.sql), [all_prs_merged.json](https://github.com/cncf/gha2db/blob/master/grafana/dashboards/all_prs_merged.json), [view](https://cncftest.io/dashboard/db/all-prs-merged?orgId=1).
6) Number of PRs merged per repository dashboard [prs_merged.sql](https://github.com/cncf/gha2db/blob/master/metrics/prs_merged.sql), [prs_merged.json](https://github.com/cncf/gha2db/blob/master/grafana/dashboards/prs_merged.json), [view](https://cncftest.io/dashboard/db/prs-merged?orgId=1).
7) PRs from opened to merged, from 2014-06 dashboard [opened_to_merged.sql](https://github.com/cncf/gha2db/blob/master/metrics/opened_to_merged.sql), [opened_to_merged.json](https://github.com/cncf/gha2db/blob/master/grafana/dashboards/opened_to_merged.json), [view](https://cncftest.io/dashboard/db/opened-to-merged?orgId=1).
8) PRs from opened to LGTMed, approved and merged dashboard [time_metrics.sql](https://github.com/cncf/gha2db/blob/master/metrics/time_metrics.sql), [time_metrics.json](https://github.com/cncf/gha2db/blob/master/grafana/dashboards/time_metrics.json), [view](https://cncftest.io/dashboard/db/time-metrics?orgId=1).
9) PR Comments dashboard [pr_comments.sql](https://github.com/cncf/gha2db/blob/master/metrics/pr_comments.sql), [pr_comments.json](https://github.com/cncf/gha2db/blob/master/grafana/dashboards/pr_comments.json), [view](https://cncftest.io/dashboard/db/pr-comments?orgId=1).
10) Companies velocity dashboard [company_activity.sql](https://github.com/cncf/gha2db/blob/master/metrics/company_activity.sql), [companies_velocity.json](https://github.com/cncf/gha2db/blob/master/grafana/dashboards/companies_velocity.json), [view](https://cncftest.io/dashboard/db/companies-velocity?orgId=1).

All of them works live on [cncftest.io](https://cncftest.io) with auto sync tool running.

# Detailed Usage instructions

- [USAGE](https://github.com/cncf/gha2db/blob/master/USAGE.md)

# Benchmarks

Ruby version was dropped, but You can see benchmarks of Ruby using MySQL, Ruby using Postgres and current Go using Postgres here:

[Benchmarks](https://github.com/cncf/gha2db/blob/master/BENCHMARK.md)

In summary: Go version can import all GitHub archives data (not discarding anything) for all Kubernetes orgs/repos, from the beginning on GitHub 2014-06-01 in about 2-2,5 hours!

