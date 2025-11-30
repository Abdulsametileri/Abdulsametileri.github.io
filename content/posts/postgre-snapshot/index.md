---
title: "How we implement Debezium style initial data (snapshot) progress in our postgresql cdc library"
date: "2025-11-30"
description: ""
summary: ""
tags: [ "go", "postgresql" ]
categories: [ "go", "postgresql" ]
draft: true
---

> In this article, we will talk about how we implement snapshot mode to our [go-pq-cdc](https://github.com/Trendyol/go-pq-cdc) library and how it works under the hood.

## What is go-pq-cdc?

* [go-pq-cdc](https://github.com/Trendyol/go-pq-cdc) is our PostgreSQL CDC _(Change Data Capture)_ library. It is based on [postgresql' logical replication protocol](https://www.postgresql.org/docs/current/logical-replication.html). It's a [Debezium](https://debezium.io/) alternative but in better way in terms of resource consumption and performance [benchmarks](https://github.com/Trendyol/go-pq-cdc/tree/main/benchmark).

* CDC version of this library is used in production for a year. Thanks to [Serhat Karabulut](https://www.linkedin.com/in/serhat-karabulut/). He writes this project and supported a lot for all process.

* PostgreSQL to [Kafka](https://github.com/Trendyol/go-pq-cdc-kafka) and [Elasticsearch](https://github.com/Trendyol/go-pq-cdc-elasticsearch) also available and used in production.

## What is logical replication? 

> [Logical replication](https://www.postgresql.org/docs/current/logical-replication.html) is a method of replicating data objects and their changes, based upon their replication identity (usually a primary key). It uses a publish-subscribe model.

## Publication

* [Publication](https://www.postgresql.org/docs/current/logical-replication-publication.html#LOGICAL-REPLICATION-PUBLICATION) is a set of **changes** _(insert, update, delete, truncate)_ generated from a table or a group of tables.
  
  ```go
  publication.Config{
    Name:              "cdc_publication",
    Operations: publication.Operations{
        publication.OperationInsert,
        publication.OperationDelete,
        publication.OperationTruncate,
        publication.OperationUpdate,
    },
    Tables: publication.Tables{
        publication.Table{
            Name:            "books",
            ReplicaIdentity: publication.ReplicaIdentityDefault,
            Schema:          "public",
        },
        publication.Table{
            Name:            "users",
            ReplicaIdentity: publication.ReplicaIdentityFull,
            Schema:          "public",
        },
    },
  }
  ```

We mentioned that a publication is a set of operations. But how do we identify which rows were updated or deleted? This is where [`Replication Identity`](https://www.postgresql.org/docs/current/logical-replication-publication.html#LOGICAL-REPLICATION-PUBLICATION-REPLICA-IDENTITY) comes into play.

* By default, this is the primary key, if there is one.
* If the table doesn't have a suitable key, then the entire row becomes the key.

In our library, we support two replication identity options:

```go
ReplicaIdentityOptions = []string{ReplicaIdentityDefault, ReplicaIdentityFull}
ReplicaIdentityMap     = map[string]string{
    "d": ReplicaIdentityDefault, // primary key
    "f": ReplicaIdentityFull,    // full row
}
```

## Replication Slot

- [Replication Slot](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION-SLOTS) is a server-side object that tracks a replica's progress and **retains WAL files** to prevent them from being deleted before the replica has consumed them. This concept existed before logical replication was introduced, but it's also used in logical replication.

- In the logical replication context, a replication slot holds the [Log Sequence Number (LSN)](https://www.postgresql.org/docs/18/datatype-pg-lsn.html)—a position in the transaction log (WAL/XLOG). Every time the subscriber successfully receives and processes data, the slot's LSN is advanced. This ensures no data loss and enables accurate restart from the last processed position.

Here's how you configure a replication slot in go-pq-cdc:

```go
Slot: slot.Config{
    Name:  "cdc_slot"
}
```

> There is a library-specific config `slot.slotActivityCheckerInterval` that continuously tracks the slot activity. If the replication slot becomes inactive, go-pq-cdc automatically reclaims the slot and resumes data capturing for high availability.

## How it is used in go-pq-cdc?

Full example [here](https://github.com/Trendyol/go-pq-cdc/tree/main/example/simple).

### Configuration

First, set up the configuration with publication, slot, and other settings:

```go
cfg := config.Config{
    // PostgreSQL connection credentials (Host, Port, User, Password, Database)
    Publication: publication.Config{
        CreateIfNotExists: true,
        Name:              "cdc_publication",
        Operations: publication.Operations{
            publication.OperationInsert,
            publication.OperationDelete,
            publication.OperationTruncate,
            publication.OperationUpdate,
        },
        Tables: publication.Tables{
            publication.Table{
                Name:            "users",
                ReplicaIdentity: publication.ReplicaIdentityDefault,
                Schema:          "public",
            },
        },
    },
    Slot: slot.Config{
        CreateIfNotExists:           true,
        Name:                        "cdc_slot",
    },
    Metric: config.MetricConfig{
        Port: 8081,
    },
    Logger: config.LoggerConfig{
        LogLevel: slog.LevelInfo,
    },
}

connector, err := cdc.NewConnector(ctx, cfg, Handler)
if err != nil {
    slog.Error("new connector", "error", err)
    os.Exit(1)
}

defer connector.Close()
connector.Start(ctx)
```

### Handler Function

Subscribe to changes by providing a handler function that processes different message types:

```go
func Handler(ctx *replication.ListenerContext) {
	switch msg := ctx.Message.(type) {
	case *format.Insert:
		slog.Info("insert message received", "new", msg.Decoded)
	case *format.Delete:
		slog.Info("delete message received", "old", msg.OldDecoded)
	case *format.Update:
		slog.Info("update message received", "new", msg.NewDecoded, "old", msg.OldDecoded)
	}

	if err := ctx.Ack(); err != nil {
		slog.Error("ack", "error", err)
	}
}
```

> By calling `ctx.Ack()`, we signal that we've processed this message and the WAL LSN position should be advanced. There's no need to keep that message change anymore in the wal segment.

## Snapshot Feature

In database terminology, a snapshot refers to a copy of a database (or table in a database) that is taken at a particular point in time—just like taking a snapshot with a camera ([From Streaming Databases Book](https://www.oreilly.com/library/view/streaming-databases/9781098154820/)).

The **Snapshot Feature** enables **initial data capture** from PostgreSQL tables before starting Change Data Capture (CDC). This ensures that your downstream systems receive both:

1. **Existing data** (via snapshot)
2. **Real-time changes** (via CDC)

Without snapshot support, CDC only captures changes that occur *after* the replication slot is created, missing all pre-existing data.

There are three snapshot mode options available:

- **initial:** Take snapshot only if no previous snapshot exists, then start CDC. 
- **snapshot_only:** Take snapshot and exit _(no CDC, no replication slot required)_.
- **never:** Skip snapshot, start CDC immediately. This is the default behavior.

Let's see how it's used.

### How snapshot mode 'Initial' is used

Full example [here](https://github.com/Trendyol/go-pq-cdc/tree/main/example/basic-snapshot-initial-mode).

### Configuration

```go
cfg := config.Config{
    // PostgreSQL connection credentials (Host, Port, User, Password, Database)
    Publication: publication.Config{
        CreateIfNotExists: true,
        Name:              "cdc_publication",
        Operations: publication.Operations{
            publication.OperationInsert,
            publication.OperationDelete,
            publication.OperationTruncate,
            publication.OperationUpdate,
        },
        Tables: publication.Tables{
            publication.Table{
                Name:            "users",
                ReplicaIdentity: publication.ReplicaIdentityDefault,
                Schema:          "public",
            },
        },
    },
    Slot: slot.Config{
        CreateIfNotExists:           true,
        Name:                        "cdc_slot",
    },
    Snapshot: config.SnapshotConfig{ // -- NEWLY ADDED!
        Enabled:           true,
        Mode:              config.SnapshotModeInitial, 
        ChunkSize:         100,
        ClaimTimeout:      30 * time.Second,
        HeartbeatInterval: 5 * time.Second,
    }
    Metric: config.MetricConfig{
        Port: 8081,
    },
    Logger: config.LoggerConfig{
        LogLevel: slog.LevelInfo,
    },
}

connector, err := cdc.NewConnector(ctx, cfg, Handler)
if err != nil {
    slog.Error("new connector", "error", err)
    os.Exit(1)
}

defer connector.Close()
connector.Start(ctx)
```

### Handler Function

Subscribe to snapshot and changes by providing a handler function that processes different message types:

```go
func Handler(ctx *replication.ListenerContext) {
	switch msg := ctx.Message.(type) {
	case *format.Insert:
		slog.Info("insert message received", "new", msg.Decoded)
	case *format.Delete:
		slog.Info("delete message received", "old", msg.OldDecoded)
	case *format.Update:
		slog.Info("update message received", "new", msg.NewDecoded, "old", msg.OldDecoded)
	case *format.Snapshot:
		handleSnapshot(msg)
	}

	if err := ctx.Ack(); err != nil {
		slog.Error("ack", "error", err)
	}

func handleSnapshot(s *format.Snapshot) {
	switch s.EventType {
	case format.SnapshotEventTypeBegin:
		log.Printf("📸 SNAPSHOT BEGIN | LSN: %s | Time: %s",
			s.LSN.String(),
			s.ServerTime.Format("15:04:05"))

	case format.SnapshotEventTypeData:
		slog.Info("snapshot message received", "data", s.Data)

	case format.SnapshotEventTypeEnd:
		log.Printf("📸 SNAPSHOT END | LSN: %s | Time: %s",
			s.LSN.String(),
			s.ServerTime.Format("15:04:05"))
	}
}
```

> We are exposing snapshot begin and end marker for better control.

> Note: There is no ACK process in snapshot. `ctx.Ack()` does nothing, returns nil.

### How snapshot mode 'Snapshot Only' is used

Full example [here](https://github.com/Trendyol/go-pq-cdc/tree/main/example/basic-snapshot-only-mode).

### Configuration
- No need to provide slot, publication etc. 

```go
cfg := config.Config{
    // PostgreSQL connection credentials (Host, Port, User, Password, Database)
    Snapshot: config.SnapshotConfig{
        Enabled:           true,
        Mode:              config.SnapshotModeSnapshotOnly,
        Tables: publication.Tables{ // --> Snapshot tables
            publication.Table{
                Name:            "users",
                ReplicaIdentity: publication.ReplicaIdentityDefault,
                Schema:          "public",
            },
        }, 
        ChunkSize:         100,
        ClaimTimeout:      30 * time.Second,
        HeartbeatInterval: 5 * time.Second,
    }
    Metric: config.MetricConfig{
        Port: 8081,
    },
    Logger: config.LoggerConfig{
        LogLevel: slog.LevelInfo,
    },
}

connector, err := cdc.NewConnector(ctx, cfg, Handler)
if err != nil {
    slog.Error("new connector", "error", err)
    os.Exit(1)
}

defer connector.Close()
connector.Start(ctx)
```

### Handler Function

Subscribe to snapshot messages by providing a handler function:

```go
func Handler(ctx *replication.ListenerContext) {
	msg := ctx.Message.(*format.Snapshot)

	switch msg.EventType {
	case format.SnapshotEventTypeBegin:
		log.Printf("📸 SNAPSHOT BEGIN | LSN: %s | Time: %s",
			msg.LSN.String(),
			msg.ServerTime.Format("15:04:05"))

	case format.SnapshotEventTypeData:
		slog.Info("snapshot message received", "data", msg.Data)

	case format.SnapshotEventTypeEnd:
		log.Printf("📸 SNAPSHOT END | LSN: %s | Time: %s",
			msg.LSN.String(),
			msg.ServerTime.Format("15:04:05"))
	}
}
```


### Roadmap
---


snapshot scale? <-- TODO
repeatable read transaction level
pg export snapshot
pg_current_wal_lsn() 

advisory lock
healtcheck chunk claim
cdc continiation