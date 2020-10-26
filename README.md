### THIS README IS AUTO-TRANSLATED USING Google Translate.
If you would like to assist me with proper Russian to English translation, please feel free to send PR's or contact me directly. Thanks.


# Universal implementation of Healthchecker on Golang

The utility intending to be a universal daemon capable of performing periodic checks (health checks) of various IT systems,
send alerts and perform some actions if the check status changes.

Configuration storage implemented using the [Koanf](https://github.com/knadh/koanf) library.
By default, the configuration loading from `config.yaml` file in the current directory.

```
$ ./checker

Start dev
^ _ ^

Usage:
  checker [command]

Available Commands:
  check       Run scheduler and execute checks
  gentoken    Generate auth token
  help        Help about any command
  list        List config elements
  testcfg     unmarshal config file into config structure
  version     Print the version number of Checker

Flags:
  -b, --bots start listening messenger bots (boolean default true)
  -c, --config string config file
  -f, --configformat string config file format (default "yaml")
  -s, --configsource string config file source: file or consul
  -w, --configwatchtimeout string config watch period (default "5s")
  -D, --debugLevel string Debug level: Debug, Info, Warn, Error, Fatal, Panic (default "info")
  -h, --help help for checker

Use "checker [command] --help" for more information about a command.
```

Configuration file can be in any format supported by the [Koanf](https://github.com/knadh/koanf) library.
Also, parameters can be loaded from CHECKER_* environment variables (see the Koanf's documentation).

The `-s` switch allows you to switch the receiving of the config to Consul or S3.
For S3, settings are taken from these ENV variables:

AWS_ACCESS_KEY_ID - key ID
AWS_SECRET_ACCESS_KEY - secret key
AWS_REGION - region
AWS_BUCKET - bucket name
AWS_OBJECT_KEY - path to the object from the bucket root

For Consul, two ENV variables are read: CONSUL_ADDR and CONSUL_PATH. From the first, the URL of the Consul server is taken, from the second - the path to the KV key with the config.
The KV key must contain the complete configuration, in the formats `yaml`,` json`, `toml`,` hcl` (set by the -f key), loading from the KV tree structure is not supported.

Each time period set by the `--configwatchtimeout` key (by default` 5s`) Checker tries to reread the config. If the config is loaded successfully, its validity and compliance with the current configuration are compared.
If the config is valid and differs from the current configuration, it replaces the current configuration, and the scheduler and bots are restarted.
The config loaded from the file system is also automatically monitored for updates.

The configuration is loaded according to the template (json):
```json
  {
    "defaults": {},
    "actors": {},
    "alerts": {},
    "projects": {},
    "consul_catalog": {}
  }
```

Secret parameters (passwords, tokens) can be saved in the Hashicorp Vault, at the moment it supports downloading secrets for telegram bots, JWT authorization, passwords for SQL databases and http checks.
Format: `vault: secret / path / to / token: field`. The value of the field `field` from the path` secret / path / to / token` will be used as the secret.
Secrets retrieved from the Vault are cached for 5 minutes to reduce the load on the Vault.

The `defaults` block in the` parameters` subblock describes the default check parameters that are applied to the settings of projects, if they have not been reassigned in the `parameters` block of a specific project.

Separate parameters `timer_step` and` http_port` in the `defaults` block contain the period of checks by the internal timer for the presence of actions that need to be performed at the moment, and the default port for the HTTP server.
If the PORT environment variable is set, the port number from it is used.

## The `parameters` block contains the following settings:

### in defaults and projects
```
run_every: The frequency of testing and running alerts.

// TODO check the feature
min_health: the minimum number of live checks within the healthchck that does not put the project in critical status

allow_fails: the number of checks that have failed to the critical status that can pass before the alert is sent to the critical channel

mode: notification mode, in loud mode alerts are sent to carts, in quiet mode they are only output to stdout.

noncrit_alert: the name of the notification method for non-critical alerts

crit_alert: the name of the alert method for critical alerts

command_channel: the name of the notification method for receiving a command into the bot (by default, the noncritical_channel parameter is taken)

// TODO add certificate checking for all tls, not just https
ssl_expiration_period: checking the proximity of the expiration time of SSL certificates during http checks

// TODO check the feature
periodic_report_time: submission period
mentions: who to notify in alerts for this project. it is convenient for all chat participants to keep it muted, and to notify specific persons on specific problems.

```

It should be borne in mind that the `run_every` parameter must be a multiple of the` timer_step` parameter.

For example, if the internal timer fires every 5 seconds, the check can be performed every number of seconds in multiples of 5 (60 seconds, 75 seconds, etc.)

## An "actor" is an action that must be performed when the verification status changes (actor_up'actor_down).
Actors (actions) are described in the ʻactors` block.

// TODO


## Description of notification methods is contained in the ʻalerts` block.

Three types of notifications are supported: telegram, slack / mattermost, log.

The block should contain sub-blocks, with settings specific to each notification method:

```
// Common parameters
name: The name of the notification method
type: The type of notification method (log, telegram, slack or mattermost)

// telegram parameters
bot_token: token
noncritical_channel: Channel for non-critical notifications
critical_channel: Channel for critical alerts

// slack / mattermost parameters
mattermost_webhook_url: webhook url. Used for all types of alerts and ChatOps.

```

If there is no `alerts` block, all alerts will be sent only to the log.

### Manage alerts

With the help of messages to the bot, you can manage alerts and the mode of checking projects.
Command line switch
The following commands are supported:

*/qa* with a regular chat message - completely disables all notifications (analogue of quiet in the defaults block)

*/la* with a regular message in the chat - turns on all notifications (analogous to loud in the defaults block)

Commands for managing alerts for the specified item.
The `/qp,/lp <project_name>` and `/qu,/lu <UUID>`  commands control project alerts and specific checks.
They can be sent as a regular chat message, or as a response to a specific alert.

In case of response to an alert, the project name or verification UUID is extracted from this alert.


## Description of checks is contained in the `healthchecks` block of the project.

The `healthchecks` block must contain blocks describing check sets and optionally a` parameters` block.
These settings overlap the project level and root level settings.
Each set of checks has a name in the `name` field and a description of the checks in the `checks` block.

Checks of different types are supported (mandatory parameters are marked with `*` sign).
- [HTTP](#http-check)
- [ICMP ping](#icmp-ping-check)
- [TCP port](#tcp-ping-check)
- [GetFile](#getfile-check)
- [Database queries execution](#database-queries)
- [Database field age](#database-field-age)
- [Database replication](#database-replication)
- [Redis (pub/sub)](#redis-pubsub)

### HTTP check
```
*type: "http"
*url: URL to check (GET method)
code: a set of possible HTTP codes for a successful response (slice int, for example `[200,420]` by default only 200)
answer: Text to search in the HTTP Body of the response
answer_present: check whether the text is present (by default "present") or not ("absent")
headers: An array of HTTP headers added to HTTP request:
    {
        "User-Agent": "custom_user_aget"
    }

timeout: time to wait for a response
auth: block containing credentials if http basic authentication is required.
    "auth": {
        "user": "username",
        "password": "S3cr3t!"
    }
skip_check_ssl: do not check the validity of the server SSL certificate
stop_follow_redirects: do not follow HTTP redirects
cookies: an array of http.Cookie objects (you can pass any parameters from https://golang.org/src/net/http/cookie.go
    "cookies": [
        {
          "name": "test_cookie",
          "value": "12345"
        }
    ]
```


### ICMP Ping Check
```
*type: "icmp"
*host: hostname or IP address to check
*timeout: time to wait for a response (compared to the average RTT for all attempts)
*count: number of requests sent
```

### TCP Ping check
Checks that the port is open and responds at the right time

```
*type: "tcp"
*host: hostname or IP address to check
*port: TCP port number
*timeout: time to wait for a response
attempts: number of attempts to open the port (default 3)
```

### GetFile check
Downloads the file and checks its md5 hash.

Each file is downloaded on the local file system, and deleted after verification.
It is necessary to consider possible restrictions on the size of the underlying file system.
```
*type: "getfile"
*host: url from where to download the file
*hash: md5 hash to compare file to
```

### Database Queries
Checking execution of database queries (MySQL, PostgreSQL, Clickhouse)

```
*type: check type - mysql_query, pgsql_query, clickhouse_query
*host: database server address
port: port to connect (if omitted, default ports are used)
timeout: connection and request execution timeout (connection time and request time are checked separately)
*sql_query_config: contains query parameters
**dbname: base name
**username: username
**password: password
query: the query to execute. if omitted, `select 1` is executed and the response is not validated
response: the response against which the value returned from the base is checked.
_one_ field is expected in the response. If omitted, only the fact of a successful request is checked.
```
```json
    {
      "type": "mysql_query",
      "host": "192.168.132.101",
      "port": 3306,
      "timeout": "1s",
      "sql_query_config": {
        "username": "username",
        "dbname": "dbname",
        "password": "vault:secret/cluster/userA/pass:value",
        "query": "select reg_date from users order by id asc limit 1;",
        "response": "1278938100"
      }
    }

```

### Database field age
Checking the age of a record in the database (MySQL, PostgreSQL, Clickhouse).
This check expects _one_ field containing an integer in UnixTime format.

```
*type: check type - clickhouse_query_unixtime, mysql_query_unixtime, pgsql_query_unixtime
*host: database server address
port: port to connect (if omitted, default ports are used)
timeout: timeout for connection and request execution
*sql_query_config: contains query parameters
*dbname: database name
*username: username
*password: password
query: the query to execute. if omitted, `select 1` is executed and the response is not validated
difference: maximum difference from the current time. if omitted, no check is performed.
```
```json
    {
      "type": "clickhouse_query_unixtime",
      "host": "192.168.126.50",
      "port": 9000,
      "sql_query_config": {
        "username": "username",
        "dbname": "dbname",
        "password": "she1Haiphae5",
        "query": "select max (serverTime) from forex.quotes1sec",
        "difference": "15m"
      },
      "timeout": "5s"
    }
```

### Database Replication
Checking that database replication is working (MySQL, PostgreSQL).

Checking algorithm: a record with random `id` and `test_value` is inserted into the table on the leading server.
Values are selected in the range 1-5 for `id` and 1-9999 for `test_value`.
If the insert was successful, then values read from all the servers in the `serverlist` field with the corresponding `id`.
If replication works, then the result on each server must match `test_value`.

Configuring is similar to query validation, but with tablename and serverlist parameters instead of the query/response parameters.
The tablename contains the name of the table to insert the test record ("repl_test" by default). The serverlist block contains a list of servers to check.
It is best to include all servers in the cluster (including the master) in the list for better control.

```
*type: check type - mysql_replication, pgsql_replication
```
Configuration example:
```json
    {
      "type": "pgsql_replication",
      "host": "master.pgsql.service.staging.consul",
      "port": 5432,
        "sql_repl_config": {
        "username": "username",
        "dbname": "dbname",
        "password": "ieb6aj2Queet",
        "tablename": "repl_test",
        "serverlist": [
          "pgsql-main-0.node.staging.consul",
          "pgsql-main-1.node.staging.consul",
          "pgsql-main-2.node.staging.consul"
        ]
      }
    }
```

The table with following DDL should be created:
```
    CREATE TABLE repl_test (
       id int primary key,
       test_value int
    )
```

### Redis Pub/Sub
After a successful subscription, Checker waits for any message (of type other than Subscription/Pong) in each of the configured channels.
When calculating the timeout for this kind of check, you must take into account:

1) time of connection to the server being checked
2) the time to complete the subscription and wait for confirmation in the Subscription message, the time to receive the data message.

```
* type: check type - redis_pubsub
* host: server address
port: port to connect (if omitted, default ports are used)
timeout: timeout for connection and request execution
* pubsub_config: contains request parameters
* channel: the name of the channel to subscribe
password: password
```

```json
    {
      "type": "redis_pubsub",
      "host": "master.redis.service.staging.consul",
      "pubsub_config": {
        "channels": [
          "ticks_EURUSD",
          "ticks_USBRUB"
        ]
      },
      "timeout": "5s"
    }
```

### Passive checks
If an active check is undesirable or impossible for some reason, a passive check will allow you to track the check status.

```
    {
      "name": "passive check of service A",
      "type": "passive",
      "timeout": 5m
    }

Check refresh requests should be a GET request to the endpoint `http://checker/check/ping/<check uuid>`.
A list of all UUIDs can be obtained with a GET request to the endpoint http://checker/list, or with the CLI command `checker list`.
To get the list via WEB, JWT authorization is required, [see](#web-api):

  curl -H "Authorization: <token>" http://checker/list

```

## importing services from Consul
// TODO describe
consul_catalog

## Metrics

Metrics in prometheus format are published at the / metrics endpoint.

The `sched_ *` metrics reflect the work of the internal scheduling cycle.

Metrics ʻalerts_by_event_type` - statistics on alerts in the context of various events.

Metrics ʻevents_by_ * `- statistics on events in the context of various projects and audits.

Metrics `check_duration` - statistics on the execution time of checks.


## Web API

`/check/ping/<check-uuid>` - update passive check status
`/check/status/<check-uuid>` - request the check status

Some web endpoints require JWT authorization. JWT token is generated using the CLI command `checker gentoken`.
The token is generated using encryption key specified in the configuration in the `defaults.token_encryption_key` parameter, or in the ENV variable.
ENV variable has higher priority.
Also loading a key from a Vault is supported.