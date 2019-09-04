---
title: crdb_internal
summary: The crdb_internal database contains read-only views that you can use for introspection into your database's tables, columns, indexes, and views.
toc: true
---

CockroachDB provides a virtual [schema](sql-name-resolution.html#logical-schemas-and-namespaces) called `crdb_internal` that contains information about cluster settings, database objects, and CockroachDB internals related to a specific database. `crdb_internal` tables are read-only, and are recommended for scripting, over standard CockroachDB [SQL statements](sql-statements.html).

{{site.data.alerts.callout_info}}
The `crdb_internal` views typically represent objects that the current user has privilege to access. To ensure you can view all the objects in a database, access it as the `root` user.
{{site.data.alerts.end}}

{{site.data.alerts.warning}}
`crdb_internal` reflects the internals of CockroachDB and may thus change across CockroachDB versions.
{{site.data.alerts.end}}

## Data exposed by `crdb_internal`

To perform introspection on objects, you can read from the related `crdb_internal` table. Unless specified otherwise, queries to  `crdb_internal` assume the [current database](sql-name-resolution.html#current-database).

For example, if the current database is set as [`movr`](movr.html), to return the `crdb_internal` table for partitions for the `movr` database, you can use the following statement:

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM crdb_internal.ranges;
~~~
~~~
  range_id |     start_key     |         start_pretty          |      end_key      |          end_pretty           | database_name |         table_name         | index_name |  replicas   |                                                        replica_localities                                                        | learner_replicas | split_enforced_until | lease_holder | range_size
+----------+-------------------+-------------------------------+-------------------+-------------------------------+---------------+----------------------------+------------+-------------+----------------------------------------------------------------------------------------------------------------------------------+------------------+----------------------+--------------+------------+
         1 |                   | /Min                          | \004\000liveness- | /System/NodeLiveness          |               |                            |            | {1}         | {"region=us-east1,az=b"}                                                                                                         | {}               | NULL                 |            1 |      12174
         2 | \004\000liveness- | /System/NodeLiveness          | \004\000liveness. | /System/NodeLivenessMax       |               |                            |            | {1,2,4,5,8} | {"region=us-east1,az=b","region=us-east1,az=c","region=us-west1,az=a","region=us-west1,az=b","region=europe-west1,az=c"}         | {}               | NULL                 |            5 |        747
         3 | \004\000liveness. | /System/NodeLivenessMax       | \004tsd           | /System/tsd                   |               |                            |            | {2,3,4,5,8} | {"region=us-east1,az=c","region=us-east1,az=d","region=us-west1,az=a","region=us-west1,az=b","region=europe-west1,az=c"}         | {}               | NULL                 |            2 |     241720
         4 | \004tsd           | /System/tsd                   | \004tse           | /System/"tse"                 |               |                            |            | {1}         | {"region=us-east1,az=b"}                                                                                                         | {}               | NULL                 |            1 |     290601
         5 | \004tse           | /System/"tse"                 | \210              | /Table/SystemConfigSpan/Start |               |                            |            | {1}         | {"region=us-east1,az=b"}                                                                                                         | {}               | NULL                 |            1 |          0
         6 | \210              | /Table/SystemConfigSpan/Start | \223              | /Table/11                     |               |                            |            | {1}         | {"region=us-east1,az=b"}                                                                                                         | {}               | NULL                 |            1 |      45202
         7 | \223              | /Table/11                     | \224              | /Table/12                     | system        | lease                      |            | {2,6,7,8,9} | {"region=us-east1,az=c","region=us-west1,az=c","region=europe-west1,az=b","region=europe-west1,az=c","region=europe-west1,az=d"} | {}               | NULL                 |            9 |       1467
         8 | \224              | /Table/12                     | \225              | /Table/13                     | system        | eventlog                   |            | {2,3,4,6,9} | {"region=us-east1,az=c","region=us-east1,az=d","region=us-west1,az=a","region=us-west1,az=c","region=europe-west1,az=d"}         | {}               | NULL                 |            2 |      20576
         9 | \225              | /Table/13                     | \226              | /Table/14                     | system        | rangelog                   |            | {3,5,6,7,9} | {"region=us-east1,az=d","region=us-west1,az=b","region=us-west1,az=c","region=europe-west1,az=b","region=europe-west1,az=d"}     | {}               | NULL                 |            5 |     127402
        10 | \226              | /Table/14                     | \227              | /Table/15                     | system        | ui                         |            | {1}         | {"region=us-east1,az=b"}                                                                                                         | {}               | NULL                 |            1 |          0
        11 | \227              | /Table/15                     | \230              | /Table/16                     | system        | jobs                       |            | {2,3,4,7,9} | {"region=us-east1,az=c","region=us-east1,az=d","region=us-west1,az=a","region=europe-west1,az=b","region=europe-west1,az=d"}     | {}               | NULL                 |            3 |       5257
        12 | \230              | /Table/16                     | \231              | /Table/17                     |               |                            |            | {1}         | {"region=us-east1,az=b"}                                                                                                         | {}               | NULL                 |            1 |          0
        13 | \231              | /Table/17                     | \232              | /Table/18                     |               |                            |            | {1}         | {"region=us-east1,az=b"}                                                                                                         | {}               | NULL                 |            1 |          0
        14 | \232              | /Table/18                     | \233              | /Table/19                     |               |                            |            | {1}         | {"region=us-east1,az=b"}                                                                                                         | {}               | NULL                 |            1 |          0
        15 | \233              | /Table/19                     | \234              | /Table/20                     | system        | web_sessions               |            | {1,3,5,6,7} | {"region=us-east1,az=b","region=us-east1,az=d","region=us-west1,az=b","region=us-west1,az=c","region=europe-west1,az=b"}         | {}               | NULL                 |            7 |          0
        16 | \234              | /Table/20                     | \235              | /Table/21                     | system        | table_statistics           |            | {2,5,6,7,8} | {"region=us-east1,az=c","region=us-west1,az=b","region=us-west1,az=c","region=europe-west1,az=b","region=europe-west1,az=c"}     | {}               | NULL                 |            5 |          0
        17 | \235              | /Table/21                     | \236              | /Table/22                     | system        | locations                  |            | {3,4,5,8,9} | {"region=us-east1,az=d","region=us-west1,az=a","region=us-west1,az=b","region=europe-west1,az=c","region=europe-west1,az=d"}     | {}               | NULL                 |            8 |        326
        18 | \236              | /Table/22                     | \237              | /Table/23                     |               |                            |            | {1}         | {"region=us-east1,az=b"}                                                                                                         | {}               | NULL                 |            1 |          0
        19 | \237              | /Table/23                     | \240              | /Table/24                     | system        | role_members               |            | {1,3,4,5,8} | {"region=us-east1,az=b","region=us-east1,az=d","region=us-west1,az=a","region=us-west1,az=b","region=europe-west1,az=c"}         | {}               | NULL                 |            3 |        146
        20 | \240              | /Table/24                     | \275              | /Table/53                     | system        | comments                   |            | {2,5,6,7,9} | {"region=us-east1,az=c","region=us-west1,az=b","region=us-west1,az=c","region=europe-west1,az=b","region=europe-west1,az=d"}     | {}               | NULL                 |            2 |          0
        21 | \275              | /Table/53                     | \276              | /Table/54                     | movr          | users                      |            | {2,6,9}     | {"region=us-east1,az=c","region=us-west1,az=c","region=europe-west1,az=d"}                                                       | {}               | NULL                 |            6 |       5611
        22 | \276              | /Table/54                     | \277              | /Table/55                     | movr          | vehicles                   |            | {1,6,7}     | {"region=us-east1,az=b","region=us-west1,az=c","region=europe-west1,az=b"}                                                       | {}               | NULL                 |            6 |       3585
        23 | \277              | /Table/55                     | \300              | /Table/56                     | movr          | rides                      |            | {2,5,9}     | {"region=us-east1,az=c","region=us-west1,az=b","region=europe-west1,az=d"}                                                       | {}               | NULL                 |            5 |     176867
        24 | \300              | /Table/56                     | \301              | /Table/57                     | movr          | vehicle_location_histories |            | {3,6,9}     | {"region=us-east1,az=d","region=us-west1,az=c","region=europe-west1,az=d"}                                                       | {}               | NULL                 |            6 |      87799
        25 | \301              | /Table/57                     | \302              | /Table/58                     | movr          | promo_codes                |            | {3,4,7}     | {"region=us-east1,az=d","region=us-west1,az=a","region=europe-west1,az=b"}                                                       | {}               | NULL                 |            7 |     229083
        26 | \302              | /Table/58                     | \377\377          | /Max                          | movr          | user_promo_codes           |            | {1,4,8}     | {"region=us-east1,az=b","region=us-west1,az=a","region=europe-west1,az=c"}                                                       | {}               | NULL                 |            4 |         81
(26 rows)
~~~

## Tables in `crdb_internal`

The virtual schema `crdb_internal` contains virtual tables, including `jobs`, `partitions`, `ranges`, and `zones`.

{{site.data.alerts.callout_info}}
A query can specify a table name without a database name (e.g., `SELECT * FROM crdb_internal.ranges`). See [Name Resolution](sql-name-resolution.html) for more information.
{{site.data.alerts.end}}

## See also

- [`SHOW`](show-vars.html)
- [`SHOW COLUMNS`](show-columns.html)
- [`SHOW CONSTRAINTS`](show-constraints.html)
- [`SHOW CREATE`](show-create.html)
- [`SHOW DATABASES`](show-databases.html)
- [`SHOW GRANTS`](show-grants.html)
- [`SHOW INDEX`](show-index.html)
- [`SHOW SCHEMAS`](show-schemas.html)
- [`SHOW TABLES`](show-tables.html)
- [Logical Schemas and Namespaces](sql-name-resolution.html#logical-schemas-and-namespaces)
