---
title: Build a Multi-Region Web Application with CockroachDB
summary: Learn how to build a multi-region web application on CockroachDB using Flask and SQLAlchemy.
toc: true
---

# Developing a Multi-Region Application on CockroachDB

In this guide, we walk you through developing and deploying a multi-region web application built on CockroachDB.

Throughout the guide, we reference the source code for a full-stack web application built for the fictional vehicle-sharing company [MovR](movr.html). The source code for this application is open source and available on GitHub, in the [movr-flask repo](https://github.com/cockroachlabs/movr-flask). The code is well-commented, with docstrings defined at the beginning of each class and function definition. The repo's [README](https://github.com/cockroachlabs/movr-flask/README.md) also includes instructions on debugging and deploying the application.

## Overview

MovR offers its users a platform for sharing vehicles, like scooters, bicycles, and skateboards, in cities around the world. As a global company, MovR serves a global user base, and needs a globally-available application to serve its users.

### Latency in global applications

When building a global application, it's important to consider the latency of read and write requests to the application and the database. Limiting latency improves the user's experience, and can help an application avoid [transaction contention](performance-best-practices-overview.html#understanding-and-avoiding-transaction-contention).

If MovR's application and database servers are deployed in a single region, latency can pose a problem to end users located in cities outside that deployment's region. If the servers are deployed across multiple regions, latency can pose a serious problem if client requests sent to any server in the deployment, without consideration for the client's location.

To limit database latency for a global application, the database deployment should be globally distributed and the data [geo-partitioned](training/geo-partitioning.html). Geo-partitioning guarantees that certain data stays in a particular region, preferably the region closest to the client. To limit application latency for global applications, the application should be locality-aware and globally distributed behind a global load balancer.  

In the sections that follow, we cover some best practices in database schema creation and database deployment with an example database and deployment. We also cover some basics for developing a locality-aware, global application, with detailed deployment steps.


## Before you begin

Before you begin, make sure that you have installed the following on your local machine:

- [CockroachDB](install-cockroachdb-mac.html)
- [Python 3](https://www.python.org/downloads/)
- [`pipenv`](https://docs.pipenv.org/en/latest/install/#installing-pipenv) (for debugging)
- [Docker](https://docs.docker.com/v17.12/docker-for-mac/install/) (for production deployment)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) (for production deployment)

There are a number of Python libraries that you also need for this tutorial, including `flask`, `sqlalchemy`, and `cockroachdb`. Rather than downloading these dependencies directly from PyPi to your machine, you should list them in dependency configuration files (e.g. a `requirements.txt` file), which we will go over later in the tutorial.

Next, clone the [movr-flask](https://github.com/cockroachlabs/movr-flask) repo. After configuring just a few files, you'll be able to test out the application locally.

## The database

The MovR application is built on a multi-region deployment of CockroachDB that uses a database named `movr`. This database is a slightly simplified version of the [`movr` database](movr.html) that is built into the `cockroach` binary. The `movr` database for the application contains the following tables:

- `users`
- `vehicles`
- `rides`

<img src="{{ 'images/v19.2/movr_v2.png' | relative_url }}" alt="MovR database schema" style="border:1px solid #eee;max-width:100%" />

These tables contain information about the users and vehicles registered with MovR, and the rides associated with those users and vehicles.

The database initialization and `CREATE` statements are all defined in `dbinit.sql`, a SQL file that you will use to load the database to a running cluster, later in this tutorial.

### Geo-partitioning

Distributed CockroachDB deployments consist of multiple regional deployments of CockroachDB nodes that communicate as a single, logical database (in CockroachDB terminology, a **cluster**). At startup, each node in a cluster is assigned a [locality](https://www.cockroachlabs.com/docs/v19.2/cockroach-start.html#locality). After startup, nodes with the same locality can be assigned to the same [replication zone](https://www.cockroachlabs.com/docs/stable/configure-replication-zones.html). When you partition data, you break up tables into segments, by row, based on a common value or characteristic. With CockroachDB, you can constrain a partition of data to a replication zone.

For a concrete example of geo-partitioning, suppose a database is deployed across multiple cloud provider regions, each node assigned a `region` locality. Now consider the `movr` database. Notice that each table contains a `city` column, which assigns a location to each row of data. For example, if a user is registered in New York, their row in the `users` table will have a `city` value of `new york`. If that user takes a ride in Seattle, that ride's row in the `rides` table has a `city` value of `seattle`. You can partition the tables of the `movr` database into regions that correspond to the replication zones, with their rows assigned to partitions based on the row's `city` value.

For example:

~~~ sql
PARTITION BY LIST (city) (
             |     PARTITION us_west VALUES IN (('seattle'), ('san francisco'), ('los angeles')),
             |     PARTITION us_east VALUES IN (('new york'), ('boston'), ('washington dc')),
             |     PARTITION europe_west VALUES IN (('amsterdam'), ('paris'), ('rome'))
             | );
~~~

You can then constrain each partition to a replication zone, using a zone constraint. For the `users` table, this looks like:

~~~ sql
ALTER PARTITION europe_west OF INDEX movr.public.users@primary CONFIGURE ZONE USING
             |     constraints = '[+region=gcp-europe-west1]';
             | ALTER PARTITION us_east OF INDEX movr.public.users@primary CONFIGURE ZONE USING
             |     constraints = '[+region=gcp-us-east1]';
             | ALTER PARTITION us_west OF INDEX movr.public.users@primary CONFIGURE ZONE USING
             |     constraints = '[+region=gcp-us-west1]'
~~~

See below for some create statements for tables in the `movr` database.

### The `users` table

Here is the create statement for the `users` table:

~~~
CREATE TABLE users (
      id UUID NOT NULL,
      city STRING NOT NULL,
      first_name STRING NULL,
      last_name STRING NULL,
      email STRING NULL,
      username STRING NULL,
      password_hash STRING NULL,
      is_owner BOOL NULL,
      CONSTRAINT "primary" PRIMARY KEY (city ASC, id ASC),
      UNIQUE INDEX users_username_key (username ASC),
      FAMILY "primary" (id, city, first_name, last_name, email, username, password_hash, is_owner),
      CONSTRAINT check_city CHECK (city IN ('amsterdam':::STRING, 'boston':::STRING, 'los angeles':::STRING, 'new york':::STRING, 'paris':::STRING, 'rome':::STRING, 'san francisco':::STRING, 'seattle':::STRING, 'washington dc':::STRING))
  );
~~~

Note the following:
- There is a composite primary key: `city` and `id`. [Primary key constraints](primary-key.html) imply [unique](unique.html) and [`NOT NULL`](not-null.html) constraints. In order to partition on a column value, the column must be indexed. For performance, the `city` column precedes the `id` column. This guarantees that the primary index of the `users` table is ordered by the `city` first, and then by `id`.
- There is a [secondary index](indexes.html) on the `username` column, which optimizes scans with a filter on the `username` column. Although explicitly stated here, CockroachDB automatically applies a [unique](unique.html) constraint to columns that are indexed.
- There is a [check constraint](check.html) on the `city` column, which verifies that the value of the `city` column is valid. When [querying partitions](partitioning.html#filtering-on-an-indexed-column), constraints can also optimize scans that have filters on the constrained columns.

### The `vehicles` table

~~~
CREATE TABLE vehicles (
      id UUID NOT NULL,
      city STRING NOT NULL,
      type STRING NULL,
      owner_id UUID NULL,
      date_added DATE NULL,
      status STRING NULL,
      last_location STRING NULL,
      color STRING NULL,
      brand STRING NULL,
      CONSTRAINT "primary" PRIMARY KEY (city ASC, id ASC),
      CONSTRAINT fk_city_ref_users FOREIGN KEY (city, owner_id) REFERENCES users(city, id),
      INDEX vehicles_auto_index_fk_city_ref_users (city ASC, owner_id ASC, status ASC) PARTITION BY LIST (city) (
          PARTITION us_west VALUES IN (('seattle'), ('san francisco'), ('los angeles')),
          PARTITION us_east VALUES IN (('new york'), ('boston'), ('washington dc')),
          PARTITION europe_west VALUES IN (('amsterdam'), ('paris'), ('rome'))
      ),
      FAMILY "primary" (id, city, type, owner_id, date_added, status, last_location, color, brand),
      CONSTRAINT check_city CHECK (city IN ('amsterdam':::STRING, 'boston':::STRING, 'los angeles':::STRING, 'new york':::STRING, 'paris':::STRING, 'rome':::STRING, 'san francisco':::STRING, 'seattle':::STRING, 'washington dc':::STRING))
  );
~~~

Note the following:

- Like the `users` table, the `vehicles` table has a composite primary key.
- The `vehicles` table has a foreign key constraint on the `users` table.
- The table has a secondary foreign key index (`vehicles_auto_index_fk_city_ref_users`). By default, CockroachDB creates secondary indexes for all foreign key constraints.

### The `rides` table

~~~
  CREATE TABLE rides (
      id UUID NOT NULL,
      city STRING NOT NULL,
      vehicle_id UUID NULL,
      rider_id UUID NULL,
      rider_city STRING NOT NULL,
      start_location STRING NULL,
      end_location STRING NULL,
      start_time TIMESTAMPTZ NULL,
      end_time TIMESTAMPTZ NULL,
      length INTERVAL NULL,
      CONSTRAINT "primary" PRIMARY KEY (city ASC, id ASC),
      CONSTRAINT fk_city_ref_users FOREIGN KEY (rider_city, rider_id) REFERENCES users(city, id),
      CONSTRAINT fk_vehicle_city_ref_vehicles FOREIGN KEY (city, vehicle_id) REFERENCES vehicles(city, id),
      INDEX rides_auto_index_fk_city_ref_users (rider_city ASC, rider_id ASC) PARTITION BY LIST (rider_city) (
          PARTITION us_west VALUES IN (('seattle'), ('san francisco'), ('los angeles')),
          PARTITION us_east VALUES IN (('new york'), ('boston'), ('washington dc')),
          PARTITION europe_west VALUES IN (('amsterdam'), ('paris'), ('rome'))
      ),
      INDEX rides_auto_index_fk_vehicle_city_ref_vehicles (city ASC, vehicle_id ASC) PARTITION BY LIST (city) (
          PARTITION us_west VALUES IN (('seattle'), ('san francisco'), ('los angeles')),
          PARTITION us_east VALUES IN (('new york'), ('boston'), ('washington dc')),
          PARTITION europe_west VALUES IN (('amsterdam'), ('paris'), ('rome'))
      ),
      FAMILY "primary" (id, city, rider_id, rider_city, vehicle_id, start_location, end_location, start_time, end_time, length),
      CONSTRAINT check_city CHECK (city IN ('amsterdam':::STRING, 'boston':::STRING, 'los angeles':::STRING, 'new york':::STRING, 'paris':::STRING, 'rome':::STRING, 'san francisco':::STRING, 'seattle':::STRING, 'washington dc':::STRING))
  );
~~~

## Setting up a development environment

At this point, you should be familiar with the `movr` database schema. Let's set up the CockroachDB cluster, and initialize the database.

### Setting up a virtual, multi-region CockroachDB cluster

In production, you want to start a secure CockroachDB cluster, with nodes on machines located in different areas of the world. For debugging and development purposes, you can just use the [`cockroach demo`](cockroach-demo.html) command. This command starts up an insecure, virtual nine-node cluster. After you are done debugging the application, you can move on to a [multi-region database deployment][#multi-region-database-deployment].

To set up the virtual, multi-region cluster:

1. Run `cockroach demo`, with the `--nodes` and `--demo-locality` flags. The localities specified below assume GCP region names.

    ~~~ shell
    $ cockroach demo \
    --nodes=9 \
    --demo-locality=region=gcp-us-east1:region=gcp-us-east1:region=gcp-us-east1:region=gcp-us-west1:region=gcp-us-west1:region=gcp-us-west1:region=gcp-europe-west1:region=gcp-europe-west1:region=gcp-europe-west1
    ~~~

    ~~~
    root@127.0.0.1:<some_port>/movr>
    ~~~

    Keep this terminal window open. Closing it will shut down the virtual cluster.

1. Copy the connection string at the prompt (e.g., `root@127.0.0.1:<some_port>/movr`).

1. In a separate terminal window, run the following command to load `dbinit.sql` to the demo database:

    ~~~ shell
    $ cockroach sql --insecure --url='postgresql://root@127.0.0.1:<some_port>/movr' < dbinit.sql
    ~~~

    This file contains the `movr` database definition, and SQL instructions to geo-partition the database.

1. Verify that the database schema loaded properly:

    ~~~ sql
    > SHOW TABLES;
    ~~~
    ~~~
      table_name
    +------------+
      rides
      users
      vehicles
    (3 rows)
    ~~~


### Setting up a virtual environment

In production, you want to containerize your application and deploy it with k8s. For debugging, use `pipenv`, a tool that manages dependencies with `pip` and creates virtual environments with `virtualenv`.

1. Run the following command to initialize the project's virtual environment:

    ~~~ shell
    $ pipenv --three
    ~~~

    `pipenv` creates a `Pipfile` in the current directory. Open this `Pipfile`, and edit it to read as follows:

    ~~~ toml
    [[source]]
    name = "pypi"
    url = "https://pypi.org/simple"
    verify_ssl = true

    [dev-packages]

    [packages]
    cockroachdb = "*"
    psycopg2-binary = "*"
    SQLAlchemy = "*"
    SQLAlchemy-Utils = "*"
    Flask = "*"
    Flask-SQLAlchemy = "*"
    Flask-WTF = "*"
    Flask-Bootstrap = "*"
    Flask-Login = "*"
    WTForms = "*"
    gunicorn = "*"
    geopy = "*"

    [requires]
    python_version = "3.7"
    ~~~

1. Run the following command to install the packages listed in the `Pipfile`:

    ~~~ shell
    $ pipenv install
    ~~~

1. Pipenv automatically sets any variables defined in a `.env` file as environment variables in a Pipenv virtual environment. To connect to a SQL database (including CockroachDB!) from a client, you need a [SQL connection string](https://en.wikipedia.org/wiki/Connection_string).

    So, open the `.env`, and then define the connection string in that file as the `DB_URI` environment variable. Note that for SQLAlchemy, the connection string protocol needs to be specific to the CockroachDB dialect.

    For example:

    ~~~
    DB_URI = 'cockroachdb://root@127.0.0.1:52382/movr'
    ~~~

    You can also specify other variables in this file that you'd rather not hard-code in the application, like API keys and secret keys used by the application. You can leave the variables as they are for now.


1. Activate the virtual environment:

    ~~~ shell
    $ pipenv shell
    ~~~

    The prompt should now read `~bash-3.2$`. From this shell, you can run any Python3 application with the required dependencies that you listed in the `Pipfile` and the environment variables that you listed in the `.env` file. You can exit the shell subprocess at any time with a simple `exit` command.

1. To test out the application, you can simply run the server file:

    ~~~ shell
    $ python3 server.py
    ~~~

    You can alternatively use [gunicorn](https://gunicorn.org/) for the server.

    ~~~ shell
    $ gunicorn -b localhost:8000 server:app
    ~~~

1. Navigate to the URL provided to test out the application.


### Setting up the Python project

The MovR application needs to handle requests from clients, namely web browsers. To translate these kinds of requests into database transactions, the application stack consists of the following components:

- A multi-node, geo-distributed CockroachDB cluster, with the replication zones for cities where MovR is supported.
- A database schema that defines the tables, indexes, and partitions that hold user, vehicle, and ride data.
- A file that defines Python classes that map to databases on our CockroachDB cluster.
- A backend API that defines the application's connection to the database and the database transactions.
- A Flask server that handles requests from client mobile applications and web browsers.
- HTML files that define web pages that the Flask server can host.

We have already set up the database and our Python environment, so we can now look at the Python project.

Here is the project directory structure:

~~~ shell
movr
├── Dockerfile  ## Defines the Docker container build
├── LICENSE  ## A license file for your application
├── Pipfile ## Lists PyPi dependencies for pipenv
├── Pipfile.lock
├── README.md  ## Contains instructions on running the application
├── __init__.py
├── dbinit.sql  ## Initializes the database and partitions the tables by region
├── mcingress.yaml ## Defines the multi-cluster ingress K8s object, for multi-region deployment
├── movr
│   ├── __init__.py
│   ├── models.py  ## Defines classes that map to tables in the movr database
│   ├── movr.py  ## Defines the primary backend API
│   └── transactions.py ## Defines callback functions
├── movr.yaml  ## Defines service and deployment K8s objects for multi-region deployment
├── requirements.txt  ## Lists PyPi dependencies for Docker container
├── server.py  ## Defines a Flask web application that to handle requests from clients and render HTML files
├── static  ## Static resources for the web frontend
│   ├── movr-lockup.png
│   └── movr-symbol.png
├── templates  ## HTML templates for the web frontend
│   ├── _nav.html
│   ├── home.html
│   ├── layout.html
│   ├── login.html
│   ├── register.html
│   ├── rides.html
│   ├── user.html
│   ├── users.html
│   ├── vehicles-add.html
│   └── vehicles.html
└── web
    ├── __init__.py
    ├── config.py   ## Contains Flask configuration settings
    ├── forms.py  ## Defines FlaskForm classes
    ├── geoutils.py  ## Defines utility functions for the web application
    └── gunicorn.py  ## Contains gunicorn configuration settings
~~~

In the sections that follow, we go over each of the files and folders in the project. We'll spend most time on backend and database components.

## Using SQLAlchemy with CockroachDB

Object Relational Mappers (ORM's) map classes to tables, class instances to rows, and class methods to transactions on the rows of a table. The `sqlalchemy` package includes some base classes and methods that you can use to connect to your database's server from a Python application, then map tables in that database to Python classes. In this tutorial, you'll use SQLAlchemy's [Declarative](https://docs.sqlalchemy.org/en/13/orm/extensions/declarative/) extension, which is built on the `mapper()` and `Table` functions. You'll also use the `cockroachdb` Python package, which includes some functions that help you handle [transactions](transactions.html) in a running CockroachDB cluster.

### Declaring mappings

Let's start by mapping Python classes to the tables in the `movr` database. By now, you should be familiar with the `movr` database and each of the tables in the database (`users`, `vehicles`, and `rides`).

Open `models.py`, under the `movr` folder.

Take a look at the first 10 lines of the file:

~~~ python
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Index, String, DateTime, Integer, Boolean, Float, Interval, ForeignKey, CheckConstraint
from sqlalchemy.types import DECIMAL, DATE
from sqlalchemy.dialects.postgresql import UUID, ARRAY
import datetime
from werkzeug.security import generate_password_hash, check_password_hash
from flask_login import UserMixin

Base = declarative_base()
~~~

The first thing the file imports is `declarative_base`, a constructor for the [declarative base class](https://docs.sqlalchemy.org/en/13/orm/extensions/declarative/api.html#sqlalchemy.ext.declarative.declarative_base), built on SQLAlchemy's Declarative extension. All mapped objects inherit from this object. Below the imports, we assign a declarative base object to the `Base` variable.

This file also imports some other standard SQLAlchemy data structures that represent database objects (like columns and indexes), data types, and constraints, in addition to a standard Python library to help with default values (i.e. `datetime`). Since the application handles user logins and authentication, we also need to import some security libraries. We'll cover those in more detail when we talk about the Flask implementation of this application.

Recall that each instance of a table class represents a row in the table, so we name our table classes as if they were individual rows of their parent table, since that's what they'll become when called.

Starting with `User`:

~~~ python
class User(Base):
    __tablename__ = 'users'
    id = Column(UUID, primary_key=True)
    city = Column(String, primary_key=True)
    first_name = Column(String)
    last_name = Column(String)
    email = Column(String)
    username = Column(String, unique=True)
    password_hash = Column(String)
    is_owner = Column(Boolean)

    def __repr__(self):
        return "<User(city='%s', id='%s', name='%s')>" % (self.city, self.id, self.first_name + ' ' + self.last_name)
~~~

We've defined the `User` class to contain the following attributes:

- `__tablename__`, which holds the stored name of the table in the database. SQLAlchemy requires this attribute for all classes that map to tables.
- All of the other attributes of the `User` class (`id`, `city`, `first_name`, etc.) are stored as `Column` objects that represent columns in a database table. The constructor for each `Column` takes the column data type as its first argument, and any constraints as additional arguments. To help define column objects, SQLAlchemy also includes classes for SQL data types and column constraints. For the columns in this table, we just use `UUID` and `String` data types. The `primary_key` constraint on the first two columns defines a composite primary key with the two columns.

The `__repr__` function defines the string representation of the object.

After you define the `User` table class, you can go ahead and define classes for the rest of the tables in the `movr` database, which are only slightly more complex.

Now look at a definition for the `Vehicle` class:

~~~ python
class Vehicle(Base):
    __tablename__ = 'vehicles'
    id = Column(UUID, primary_key=True)
    city = Column(String, primary_key=True)
    type = Column(String)
    owner_id = Column(UUID, ForeignKey('users.id'))
    date_added = Column(DATE, default=datetime.date.today)
    status = Column(String)
    last_location = Column(String)
    color = Column(String)
    brand = Column(String)

    def __repr__(self):
        return "<Vehicle(city='%s', id='%s', type='%s', status='%s')>" % (self.city, self.id, self.type, self.status)
~~~

The `vehicles` table contains more columns and data types than the `users` table. It also contains a [foreign key constraint](foreign-key.html) (on the `users` table), and a default value.

Lastly, there's the `Ride` class:

~~~ python
class Ride(Base):
    __tablename__ = 'rides'
    id = Column(UUID, primary_key=True)
    city = Column(String, ForeignKey('vehicles.city'), primary_key=True)
    rider_id = Column(UUID, ForeignKey('users.id'))
    rider_city = Column(String, ForeignKey('users.city'))
    vehicle_id = Column(UUID, ForeignKey('vehicles.id'))
    start_location = Column(String)
    end_location = Column(String)
    start_time = Column(DateTime)
    end_time = Column(DateTime)
    length = Column(Interval)

    def __repr__(self):
        return "<Ride(city='%s', id='%s', rider_id='%s', vehicle_id='%s')>" % (self.city, self.id, self.rider_id, self.vehicle_id)
~~~

The `rides` table has four foreign key constraints, two on the `users` table and two on the `vehicles` table.

## Defining transactions

After you create a class for each table in the database, you can start defining functions that bundle together common SQL operations as atomic [transactions](transactions.html).

### Using the `cockroachdb` Python library

The `cockroachdb` Python library handles transactions in SQLAlchemy with the `run_transaction()` function. This function takes an [`Engine`](https://docs.sqlalchemy.org/en/13/orm/engine.html) object and a callback function as inputs. It then creates a new [`Session`](https://docs.sqlalchemy.org/en/13/orm/session.html) object, bound to the `Engine`, and executes the callback as a database transaction for the `Session`.

`run_transaction()` abstracts the details of [transaction retries](transactions.html#transaction-retries) away from your application code. Transaction retries are more frequent in CockroachDB than in some other databases because it uses [optimistic concurrency control](https://en.wikipedia.org/wiki/Optimistic_concurrency_control) rather than locking. Because of this, a CockroachDB transaction may have to be tried more than once before it can commit. This is part of how we ensure that our transaction ordering guarantees meet the ANSI [SERIALIZABLE](https://en.wikipedia.org/wiki/Isolation_(database_systems)#Serializable) isolation level.

`run_transaction()` has the following additional benefits:

- Because it must be passed a [sqlalchemy.orm.session.sessionmaker](https://docs.sqlalchemy.org/en/latest/orm/session_api.html#session-and-sessionmaker) object (*not* a [session][session]), it ensures that a new session is created exclusively for use by the callback, which protects you from accidentally reusing objects via any sessions created outside the transaction.
- It abstracts away the [client-side transaction retry logic](transactions.html#client-side-intervention) from your application, which keeps your application code portable across different databases. For example, the sample code given on this page works identically when run against Postgres (modulo changes to the prefix and port number in the connection string).
- It will run your transactions to completion much faster than a naive implementation that created a fresh transaction after every retry error. Because of the way the CockroachDB dialect's driver structures the transaction attempts (using a [`SAVEPOINT`](savepoint.html) statement under the hood, which has a slightly different meaning in CockroachDB than in some other databases), the server is able to preserve some information about the previously attempted transactions to allow subsequent retries to complete more easily.

 Because all callback functions are passed to `run_transaction()`, the `Session` method calls within those callback functions are written a little differently than the typical SQLAlchemy application. Most importantly, those functions must not change the session and/or transaction state. This is in line with the recommendations of the [SQLAlchemy FAQs](https://docs.sqlalchemy.org/en/latest/orm/session_basics.html#session-frequently-asked-questions), which state (with emphasis added by the original author) that

 > As a general rule, the application should manage the lifecycle of the session *externally* to functions that deal with specific data. This is a fundamental separation of concerns which keeps data-specific operations agnostic of the context in which they access and manipulate that data.

 and

 > Keep the lifecycle of the session (and usually the transaction) **separate and external**.

 In keeping with the above recommendations from the official docs, we **strongly recommend** avoiding any explicit mutations of the transaction state inside the callback passed to `run_transaction`, since that will lead to breakage. Specifically, we do not make calls to the following functions from inside `run_transaction`:

 - [`sqlalchemy.orm.Session.commit()`](https://docs.sqlalchemy.org/en/latest/orm/session_api.html?highlight=commit#sqlalchemy.orm.session.Session.commit) (or other variants of `commit()`): This is not necessary because `cockroachdb.sqlalchemy.run_transaction` handles the savepoint/commit logic for you.
 - [`sqlalchemy.orm.Session.rollback()`](https://docs.sqlalchemy.org/en/latest/orm/session_api.html?highlight=rollback#sqlalchemy.orm.session.Session.rollback) (or other variants of `rollback()`): This is not necessary because `cockroachdb.sqlalchemy.run_transaction` handles the commit/rollback logic for you.
 - [`Session.flush()`][session.flush]: This will not work as expected with CockroachDB because CockroachDB does not support nested transactions, which are necessary for `Session.flush()` to work properly. If the call to `Session.flush()` encounters an error and aborts, it will try to rollback. This will not be allowed by the currently-executing CockroachDB transaction created by `run_transaction()`, and will result in an error message like the following: `sqlalchemy.orm.exc.DetachedInstanceError: Instance <FooModel at 0x12345678> is not bound to a Session; attribute refresh operation cannot proceed (Background on this error at: http://sqlalche.me/e/bhk3)`.

 In the example application provided, all calls to `run_transaction()` are found within the methods of the `MovR` class (defined in `movr.py`), which represents the connection to the running database. Requests to the web application frontend (defined in `server.py`), are routed to the `MovR` class methods. We discuss `movr.py` and `server.py` in more detail in later sections.

### Defining transaction callback functions

 To separate concerns, we define all callback functions passed to `run_transaction()` calls in a separate file, called  `transactions.py`. These callback functions wrap `Session` method calls, like [`session.query()`](https://docs.sqlalchemy.org/en/13/orm/session_api.html#sqlalchemy.orm.session.Session.query) and [`session.add()`](https://docs.sqlalchemy.org/en/13/orm/session_api.html#sqlalchemy.orm.session.Session.add), to perform database operations within a transaction.

This file imports all of the table classes that we defined in `models.py`, in addition to some SQLAlchemy and standard Python data structures needed to generate correctly-typed row values that the ORM can write to the database.

~~~ python
from movr.models import Vehicle, Ride, User
from sqlalchemy import cast, Numeric
import datetime
import uuid
~~~

#### Reading

A common query that a client might want to run is a read of the `rides` table.

~~~ python
def get_rides_txn(session, rider_id):
    rides = session.query(Ride).filter(
        Ride.rider_id == rider_id).order_by(Ride.start_time).all()
    return list(map(lambda ride: {'city': ride.city, 'id': ride.id, 'vehicle_id': ride.vehicle_id, 'start_time': ride.start_time, 'end_time': ride.end_time, 'rider_id': ride.rider_id, 'length': ride.length}, rides))
~~~

The `get_rides_txn()` function takes a `Session` object and a `rider_id` string as its inputs, and it outputs a list of dictionaries containing the columns-value pairs of a row in the `rides` table. To retrieve the data from the database bound to a particular `Session` object, we use `Session.query()`, a method of the `Session` class. This method returns a `Query` object, with methods for filtering and ordering query results.

Note that the `get_rides_txn()` function returns the rides for a specific rider, which can be in more than one city. The function does not filter the query by `city`, and therefore does not limit the query to ride data stored in a particular region.

Another common query would be to read the registered vehicles in a particular city, to see which vehicles are available for riding. Unlike `get_rides_txn()`, the `get_vehicles_txn()` function takes the `city` string as an input.

~~~ python
def get_vehicles_txn(session, city):
    vehicles = session.query(Vehicle).filter(
        Vehicle.city == city, Vehicle.status != 'removed').all()
    return list(map(lambda vehicle: {'city': vehicle.city, 'id': vehicle.id, 'owner_id': vehicle.owner_id, 'type': vehicle.type, 'last_location': vehicle.last_location + ', ' + vehicle.city, 'status': vehicle.status, 'date_added': vehicle.date_added, 'color': vehicle.color, 'brand': vehicle.brand}, vehicles))
~~~

This function filters the query call by city. Because the data in the vehicles table is partitioned by city, the function only queries a specific partition of data, constrained to a particular region.

#### Writing

Let's move on to some write transactions. There are two basic types of write operations: creating new rows and updating existing rows. In SQL terminology, these are `INSERT`'s and `UPDATE`'s.  All transaction callback functions that update existing rows include a `session.query()` call. All functions adding new rows call `session.add()`. Some functions do both.

`start_ride_txn()`, which is called when a user starts a ride, adds a new row to the `rides` table and then updates a row in the `vehicles` table.

~~~ python
def start_ride_txn(session, city, rider_id, rider_city, vehicle_id):
    v = session.query(Vehicle).filter(Vehicle.city == city,
                                      Vehicle.id == vehicle_id).first()
    r = Ride(city=city, id=str(uuid.uuid4()), rider_id=rider_id,
             rider_city=rider_city, vehicle_id=vehicle_id, start_location=v.last_location, start_time=datetime.datetime.now(datetime.timezone.utc))
    session.add(r)
    v.status = "unavailable"
~~~

The function takes the `city` string, `rider_id` UUID, `rider_city` string, and `vehicle_id` UUID as inputs. It queries the `vehicles` table for all vehicles of a specific ID, and in a specific city. It also creates a `Ride` object, representing a row of the `rides` table. To add the ride to the table in the database bound to the `Session`, use `Session.add()`. To update the `vehicles` table, just modify the object attribute, like any other python variable. `run_transaction()`, called by each method in `movr.py`, commits the changes to the database.

Be sure to review the other callback functions in `transactions.py` before moving on to the next section.

Now that we've created the table classes and some transaction functions, we can look at the interface that connects web requests to the running CockroachDB cluster.

## Connecting to the database

The `MovR` class, defined in `movr.py`, handles connections to CockroachDB using SQLAlchemy's [`Engine`](https://docs.sqlalchemy.org/en/13/core/connections.html#sqlalchemy.engine.Engine) and [`Session`](https://docs.sqlalchemy.org/en/13/orm/session_api.html#sqlalchemy.orm.session.Session) classes. The `MovR` class methods function as the "backend API" for the application. Frontend requests get routed to the these methods.

### Creating a backend API

The `MovR` class, defined in `movr.py`, handles connections to CockroachDB using SQLAlchemy's `Engine` class.

~~~ python
from movr.transactions import start_ride_txn, end_ride_txn, add_user_txn, add_vehicle_txn, get_users_txn, get_user_txn, get_vehicles_txn, get_rides_txn, remove_user_txn, remove_vehicle_txn
from cockroachdb.sqlalchemy import run_transaction
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.dialects import registry
registry.register(
    "cockroachdb", "cockroachdb.sqlalchemy.dialect", "CockroachDBDialect")


class MovR:
    def __init__(self, conn_string):
        self.engine = create_engine(conn_string, convert_unicode=True)
~~~

This file imports the transaction callback functions that we defined in the `transactions.py` file. The `MovR` class methods use the `run_transaction()` function, imported from the `cockroachdb` Python library, to execute the callbacks as transactions.

The file also imports some `sqlalchemy` libraries to create instances of the `Engine` and `Session` classes, and to [register the CockroachDB as a dialect](https://docs.sqlalchemy.org/en/13/core/connections.html#registering-new-dialects).

When called, the constructor creates an instance of the `Engine` class using an input connection string that specifies the database dialect and connection arguments. The constructor then creates a `Session` object that binds to the active `Engine` object.

We've already defined the transaction logic in the callback functions in `transactions.py`. We can now wrap calls to the `run_transaction()` function with the `MovR` class methods.

For example, look at the `start_ride()` function:

~~~ python
def start_ride(self, city, rider_id, rider_city, vehicle_id):
        return run_transaction(sessionmaker(bind=self.engine), lambda session: start_ride_txn(session, city, rider_id, rider_city, vehicle_id))
~~~

The function takes some keyword arguments, and then returns the `run_transaction()` call, using a new session and the callback function `start_ride_txn()`. `start_ride()` passes `sessionmaker(bind=self.engine)` as the first argument in `run_transaction()`, creating a new `Session` object that binds to the `Engine` instance initialized by the `MovR` constructor. The function also takes `city`, `rider_id`, `rider_city`, and `vehicle_id` as inputs. These are the same keyword arguments, and should be of the same types, as the inputs for `start_ride_txn()`.

## Building the web application

By now you should have an idea of what components we need to connect to a running database, map our database objects to objects in Python, and then interact with the database transactionally. All the application needs now is a front end. We'll use Flask for the web server and routing, and some basic Bootstrapped HTML templating for the web UI.

### Configuring the server environment

Flask offers some pretty robust configuration options, and you can take several different approaches to setting those options. We store the configuration in simple `object`-based classes.

Open the file named `config.py`. This file imports the `os` library,

~~~ python
import os


class Config(object):
    DEBUG = os.environ['DEBUG']
    SECRET_KEY = os.environ['SECRET_KEY']
    API_KEY = os.environ['API_KEY']
    DB_URI = os.environ['DB_URI']
    PREFERRED_URL_SCHEME = ('https', 'http')[DEBUG == 'True']
~~~

### Routing

Flask uses `@app.route()` decorators to route client requests at specific URL's to the backend.

We define all Flask routing functions directly in `server.py`.

### Creating a Web UI

For the purposes of this tutorial, we limit the web UI to some static HTML web pages, rendered using Flask's built-in Jinja-2 engine. We won't spend too much time covering the web UI. Just note that the forms take input from the user, and that input is passed to the backend, for database operations. For more details, see the code.

We've also added some Bootstrap syntax and Google Maps, for UX purposes. As you can see, the Google Maps API requires a key. For debugging, you can define this key in the `.env` file. If you decide to use an embedded service like Google Maps in production, you should restrict your Google Maps API key to a specific hostname or IP address, as the API key will be publicly visible in the HTML. In production, we use Kubernetes secrets to the store the API keys.

## Production deployment

After you finish developing and debugging a multi-region application in your local development environment, you are ready to deploy the application.

### Multi-region database deployment

In production, you want to start a secure CockroachDB cluster, with nodes on machines located in different areas of the world. To deploy CockroachDB in multiple regions, using [Cockroach Cloud](https://cockroachlabs.cloud):

1. Create a Cockroach Cloud account.

1. Request a multi-region CockroachCloud cluster on GCP, in regions us-west1, us-east1, and europe-west1.

1. After the cluster is created, open the console, and select the cluster.

1. Select **SQL Users** from the side panel, select **Add user**, give the user a name and a password, and then add the user. You can use any user name except "root".

1. Select **Networking** from the side panel, and then select **Add network**. Give the network any name you'd like, select either a **New network** or a **Public network**, check both **UI** and **SQL**, and then add the network. In this example, we use a public network.

1. Select **Connect** at the top-right corner of the cluster console.

1. Select the **User** that you created, and then **Continue**.

1. Copy the connection string, with the user and password specified.

1. **Go back**, and retrieve the connection strings for the other two regions.

1. Download the cluster cert to your local machine (it's the same for all regions).

1. Open a new terminal, and run the `dbinit.sql` file on the running cluster to initialize the database. You can connect to the database from any node on the cluster for this step.

    ~~~ shell
    $ cockroach sql --url any-connection-string < dbinit.sql
    ~~~

    {{site.data.alerts.callout_info}}
    You need to specify the password in the connection string!
    {{site.data.alerts.end}}

    e.g.,
    ~~~ shell
    $ cockroach sql --url \ 'postgresql://user:password@region.cockroachlabs.cloud:26257/defaultdb?sslmode=verify-full&sslrootcert=certs-dir/movr-app-ca.crt' < dbinit.sql
    ~~~

{{site.data.alerts.callout_info}}
You can also deploy CRDB manually. For instructions, see the [Manual Deployment](manual-deployment.html) page of the Cockroach Labs documentation site.
{{site.data.alerts.end}}

### Multi-region application deployment (GKE)

To deploy an application in multiple regions in production, we recommend that you use a [managed Kubernetes engine](https://kubernetes.io/docs/setup/#production-environment), like [Amazon EKS](https://aws.amazon.com/eks/), [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/), or [Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/). To route requests to the container cluster deployed closest to clients, you should also set up a multi-cluster [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/). In this tutorial, we use [kubemci](https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress) to configure a Cloud HTTP Load Balancer to container clusters on GKE.

To serve a secure web application, you need a public domain name!

To deploy the application globally, we recommend that you use a major cloud provider with a global load-balancing service and a Kubernetes engine. For our deployment, we use GCP. To serve a secure web application, you also need a public domain name!

1. If you don't have a glcoud account, create one at https://cloud.google.com/.

1. Create a gcloud project on the [GCP console](https://console.cloud.google.com/).

1. Enable the [Google Maps Embed API](https://console.cloud.google.com/apis/library), create an API key, restrict the API key to all subdomains of your domain name (e.g. `https://site.com/*`), and retrieve the API key.

1. Configure/authorize the gcloud CLI to use your project and region.

    ~~~ shell
    $ gcloud init
    $ gcloud auth login
    $ gcloud auth application-default login
    ~~~

1. If you haven't already, install kubectl.

    ~~~ shell
    $ gcloud components install kubectl
    ~~~

1. Build and run the docker image locally.

    ~~~ shell
    $ docker build -t gcr.io/<gcp_project>/movr-app:v1 .
    ~~~

    If there are no errors, the container built successfully.

1. Push the Docker image to the project’s gcloud container registry.

    e.g.,
    ~~~ shell
    $ docker push gcr.io/<gcp_project>/movr-app:v1
    ~~~

1. Create a K8s cluster for all three regions.

    ~~~ shell
    $ gcloud config set compute/zone us-east1-b && \
      gcloud container clusters create movr-us-east
    $ gcloud config set compute/zone us-west1-b && \
      gcloud container clusters create movr-us-west
    $ gcloud config set compute/zone europe-west1-b && \
      gcloud container clusters create movr-europe-west
    ~~~

1. Add the container credentials to kubeconfig.

    ~~~ shell
    $ KUBECONFIG=~/mcikubeconfig gcloud container clusters get-credentials --zone=us-east1-b movr-us-east
    $ KUBECONFIG=~/mcikubeconfig gcloud container clusters get-credentials --zone=us-west1-b movr-us-west
    $ KUBECONFIG=~/mcikubeconfig gcloud container clusters get-credentials --zone=europe-west1-b movr-europe-west
    ~~~

1. For each cluster context, create a secret for the connection string, Google Maps API, and the certs, and then create the k8s deployment and service using the `movr.yaml` manifest file. To get the context for the cluser, run `kubectl config get-contexts -o name`.

    ~~~ shell
    $ kubectl config use-context <context-name> && \
    kubectl create secret generic movr-db-cert --from-file=cert=<full-path-to-cert> && \
    kubectl create secret generic movr-db-uri --from-literal=DB_URI="connection-string" && \
    kubectl create secret generic maps-api-key --from-literal=API_KEY="APIkey" \
    kubectl create -f ~/movr-flask/movr.yaml
    ~~~

    (Do this for each cluster context!)

1. Reserve s static IP address for the ingress.

    ~~~ shell
    $ gcloud compute addresses create --global movr-ip
    ~~~

1. Download [kubemci](https://github.com/GoogleCloudPlatform/k8s-multicluster-ingress) and make it executable.

    ~~~ shell
    $ chmod +x ~/kubemci
    ~~~

1. Use kubemci to make the ingress.

    ~~~ shell
    $ ~/kubemci create movr-mci \
    --ingress=<path>/movr-flask/mcingress.yaml \
    --gcp-project=<gcp_project> \
    --kubeconfig=<path>/mcikubeconfig
    ~~~

    **Note:** kubemci requires full paths.

1. In GCP's "Load balancing" console (found under "Network Services"), select and edit the load balancer that you just created.

    1. Edit the backend configuration.
        - Expand the advanced configurations, and add a custom header: `X-City: {client_city}`. This forwards an additional header to the application telling it what city the client is in. The header name (`X-City`) is hardcoded into the example application.

    1. Edit the frontend configuration, and add a new frontend.
        - Under "Protocol", select HTTPS.
        - Under "IP address", select the static IP address that you reserved earlier (e.g., "movr-ip").
        - Under "Certificate", select "Create a new certificate".
        - In the "Create a new certificate" page, give a name to the certificate (e.g., "movr-ssl-cert"). Check "Create Google-managed certificate", and then under "Domains", enter a domain name that you own and want to use for your application.
    1. Review and finalize the load balancer, and then "Update".

    **Note:** It will take several minutes to provision the SSL certificate that you just created for the frontend.

1. Check the status of the ingress.

    ~~~ shell
    $ ~/kubemci list --gcp-project=<gcp_project>
    ~~~

1. In the GCP Cloud DNS console (under Network Services), create a new zone. You can name the zone whatever you want, but be sure to enter the same domain name for which you created a certificate earlier.

1. Select your zone, and copy the nameserver addresses (under "Data") for the recordset labeled "NS".

1. Add the nameserver addresses to the authorative nameserver list for your domain name, through your domain name provider.

    **Note:** It can take up to 48 hours for changes to the authorative nameserver list to take effect.

1. Navigate to the domain name and test out your application.

1. Clean up (at your leisure).

## Upgrading your deployment

After you deploy your application, you will likely need to push changes that you've made locally to the deployed code. When pushing changes, be aware that you defined the database separate from the application. If you change a datatype, for example, in your application, you will also need to modify the database schema to be compatible with your application's requests. We recommend that you used a supported schema migration tool.

## Developing your own application

This tutorial, and the example application source code, should help you get started developing multi-region applications on CockroachDB. Although we use Flask for the web framework, there are many other web frameworks compatible with SQLAlchemy. CockroachDB is also fully compatible with other popular ORM's.

## See also

- The [SQLAlchemy](https://docs.sqlalchemy.org/en/latest/) docs
- [Transactions](transactions.html)
