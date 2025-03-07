Cassandra Plugins for Akka Persistence
======================================

[![Join the chat at https://gitter.im/akka/akka-persistence-cassandra](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/akka/akka-persistence-cassandra?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Replicated [Akka Persistence](https://doc.akka.io/docs/akka/current/scala/persistence.html) journal and snapshot store backed by [Apache Cassandra](http://cassandra.apache.org/).

[![Build Status](https://travis-ci.org/akka/akka-persistence-cassandra.svg?branch=master)](https://travis-ci.org/akka/akka-persistence-cassandra)

Dependencies
------------

### Latest release

To include the latest release of the Cassandra plugins for **Akka 2.5.x** into your `sbt` project, add the following lines to your `build.sbt` file:

    libraryDependencies += Seq(
      "com.typesafe.akka" %% "akka-persistence-cassandra" % "0.62",
      "com.typesafe.akka" %% "akka-persistence-cassandra-launcher" % "0.62" % Test
    )

This version of `akka-persistence-cassandra` depends on Akka 2.5.23. It has been published for Scala 2.11, 2.12 and 2.13.  The launcher artifact is a utility for starting an embedded Cassandra, useful for running tests. It can be removed if not needed.

To include the latest release of the Cassandra plugins for **Akka 2.4.x** into your `sbt` project, add the following lines to your `build.sbt` file:

    libraryDependencies += "com.typesafe.akka" %% "akka-persistence-cassandra" % "0.30"

This version of `akka-persistence-cassandra` depends on Akka 2.4.20. It has been published for Scala 2.11 and 2.12.

Those versions are compatible with Cassandra 3.0.0 or higher, and it is also compatible with Cassandra 2.1.6 or higher (versions < 2.1.6 have a static column bug) if you configure `cassandra-journal.cassandra-2x-compat=on` in your `application.conf`. With Cassandra 2.x compatibility some features will not be enabled, e.g. `eventsByTag`.

Journal plugin
--------------

### Features

- All operations required by the Akka Persistence [journal plugin API](https://doc.akka.io/docs/akka/current/scala/persistence.html#journal-plugin-api) are fully supported.
- The plugin uses Cassandra in a pure log-oriented way i.e. data are only ever inserted but never updated (deletions are made on user request only).
- Writes of messages are batched to optimize throughput for `persistAsync`. See [batch writes](https://doc.akka.io/docs/akka/current/scala/persistence.html#batch-writes) for details how to configure batch sizes. The plugin was tested to work properly under high load.
- Messages written by a single persistent actor are partitioned across the cluster to achieve scalability with data volume by adding nodes.
- [Persistence Query](https://doc.akka.io/docs/akka/current/scala/persistence-query.html) support by `CassandraReadJournal`

### Configuration

To activate the journal plugin, add the following line to your Akka `application.conf`:

    akka.persistence.journal.plugin = "cassandra-journal"

This will run the journal with its default settings. The default settings can be changed with the configuration properties defined in [reference.conf](https://github.com/akka/akka-persistence-cassandra/blob/v0.62/core/src/main/resources/reference.conf):

### Caveats

- Detailed tests under failure conditions are still missing.
- Range deletion performance (i.e. `deleteMessages` up to a specified sequence number) depends on the extend of previous deletions
    - linearly increases with the number of tombstones generated by previous permanent deletions and drops to a minimum after compaction
- Events by tag uses Cassandra Materialized Views which are a new feature that has yet to stabalise. Use at your own risk, see [here](https://lists.apache.org/thread.html/d81a61da48e1b872d7599df4edfa8e244d34cbd591a18539f724796f@%3Cdev.cassandra.apache.org%3E) for a recent discussion on the Cassandra dev mailing list.


These issues are likely to be resolved in future versions of the plugin.

Snapshot store plugin
---------------------

### Features

- Implements the Akka Persistence [snapshot store plugin API](https://doc.akka.io/docs/akka/current/scala/persistence.html#snapshot-store-plugin-api).

### Configuration

To activate the snapshot-store plugin, add the following line to your Akka `application.conf`:

    akka.persistence.snapshot-store.plugin = "cassandra-snapshot-store"

This will run the snapshot store with its default settings. The default settings can be changed with the configuration properties defined in [reference.conf](https://github.com/akka/akka-persistence-cassandra/blob/v0.62/core/src/main/resources/reference.conf):

Persistence Queries
-------------------

It implements the following [Persistence Queries](https://doc.akka.io/docs/akka/current/scala/persistence-query.html):

* persistenceIds, currentPersistenceIds
* eventsByPersistenceId, currentEventsByPersistenceId
* eventsByTag, currentEventsByTag (only for Cassandra 3.x)

Persistence Query usage example to obtain a stream with all events tagged with "someTag" with Persistence Query:

    val queries = PersistenceQuery(system).readJournalFor[CassandraReadJournal](CassandraReadJournal.Identifier)
    queries.eventsByTag("someTag", Offset.noOffset)

Migrations
----------

### Migrations from 0.54 to 0.59

In version 0.55 additional columns were added to be able to store meta data about an event without altering
the actual domain event.

The new columns `meta_ser_id`, `meta_ser_manifest`, and `meta` are defined in the [new journal table definition](https://github.com/akka/akka-persistence-cassandra/blob/v0.55/core/src/main/scala/akka/persistence/cassandra/journal/CassandraStatements.scala#L45-L47) and [new snapshot table definition](https://github.com/akka/akka-persistence-cassandra/blob/v0.55/core/src/main/scala/akka/persistence/cassandra/snapshot/CassandraStatements.scala#L31-L33).

These columns are used when the event is wrapped in `akka.persistence.cassandra.EventWithMetaData` or snapshot is wrapped in `akka.persistence.cassandra.SnapshotWithMetaData`. It is optional to alter the table and add the columns. It's only required to add the columns if such meta data is used.

It is also not required to add the materialized views, not even if the meta data is stored in the journal table. If the materialized view is not changed the plain events are retrieved with the `eventsByTag` query and they are not wrapped in `EventWithMetaData`. Note that Cassandra [does not support](http://docs.datastax.com/en/cql/3.3/cql/cql_reference/cqlAlterMaterializedView.html) adding columns to an existing materialized view.

If you don't alter existing messages table and still use `tables-autocreate=on` you have to set config:

```
cassandra-journal.meta-in-events-by-tag-view = off
``` 

When trying to create the materialized view (tables-autocreate=on) with the meta columns before corresponding columns have been added the messages table an exception "Undefined column name meta_ser_id" is raised, because Cassandra validates the ["CREATE MATERIALIZED VIEW IF NOT EXISTS"](https://docs.datastax.com/en/cql/3.3/cql/cql_reference/cqlCreateMaterializedView.html#cqlCreateMaterializedView__if-not-exists) even though the view already exists and will not be created. To work around that issue you can disable the meta columns in the materialized view by setting `meta-in-events-by-tag-view=off`.


### Migrations from 0.51 to 0.52

`CassandraLauncher` has been pulled out into its own artifact, and now bundles Cassandra into a single fat jar, which is bundled into the launcher artifact. This has allowed Cassandra to be launched without it being on the classpath, which prevents classpath conflicts, but it also means that Cassandra can't be configured by changing files on the classpath, for example, a custom `logback.xml` in `src/test/resources` is no longer sufficient to configure Cassandra's logging. To address this, `CassandraLauncher.start` now accepts a list of classpath elements that will be added to the classpath, and provides a utility for locating classpath elements based on resource name.

To depend on the new `CassandraLauncher` artifact, remove any dependency on `cassandra-all` itself, and add:

```scala
"com.typesafe.akka" %% "akka-persistence-cassandra-launcher" % "0.52"
```

to your build.  To modify the classpath that Cassandra uses, for example, if you have a `logback.xml` file in your `src/test/resources` directory that you want Cassandra to use, you can do this:

```scala
CassandraLauncher.start(
   cassandraDirectory,
   CassandraLauncher.DefaultTestConfigResource,
   clean = true,
   port = 0,
   CassandraLauncher.classpathForResources("logback.xml")
)
```

### Migrations from 0.23 to 0.50

The Persistence Query API changed slightly, see [migration guide for Akka 2.5](https://doc.akka.io/docs/akka/current/scala/project/migration-guide-2.4.x-2.5.x.html#persistence-query).

### Migrations from 0.11 to 0.12

Dispatcher configuration was changed, see [reference.conf](https://github.com/akka/akka-persistence-cassandra/blob/v0.62/core/src/main/resources/reference.conf):

### Migrations from 0.9 to 0.10

The event data, snapshot data and meta data are stored in a separate columns instead of being wrapped in blob. Run the following statements in `cqlsh`:

    drop materialized view akka.eventsbytag;
    alter table akka.messages add writer_uuid text;
    alter table akka.messages add ser_id int;
    alter table akka.messages add ser_manifest text;
    alter table akka.messages add event_manifest text;
    alter table akka.messages add event blob;
    alter table akka_snapshot.snapshots add ser_id int;
    alter table akka_snapshot.snapshots add ser_manifest text;
    alter table akka_snapshot.snapshots add snapshot_data blob;

### Migrations from 0.6 to 0.7

Schema changes mean that you can't upgrade from version 0.6 for Cassandra 2.x of the plugin to the 0.7 version and use existing data without schema migration. You should be able to export the data and load it to the [new table definition](https://github.com/akka/akka-persistence-cassandra/blob/v0.7/src/main/scala/akka/persistence/cassandra/journal/CassandraStatements.scala#L25).

### Migrating from 0.3.x (Akka 2.3.x)

Schema and property changes mean that you can't currently upgrade from 0.3 to 0.4 SNAPSHOT and use existing data without schema migration. You should be able to export the data and load it to the [new table definition](https://github.com/akka/akka-persistence-cassandra/blob/v0.9/src/main/scala/akka/persistence/cassandra/journal/CassandraStatements.scala).

