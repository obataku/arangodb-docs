---
layout: default
description: In a Leader/Follower setup one or more ArangoDB Followers asynchronously replicatefrom a Leader
redirect_from:
  - architecture-deployment-modes-master-slave-architecture.html # 3.8 -> 3.8
---
Leader/Follower Architecture
============================

Introduction
------------

In a _Leader/Follower_ setup one or more ArangoDB _Followers_ asynchronously replicate
from a _Leader_. 

The _Leader_ is the ArangoDB instance where all data-modification operations should
be directed to. The _Follower_ is the ArangoDB instance that replicates the data from
the Leader.

Components
----------

### Replication Logger

**Purpose**

The _replication logger_ will write all data-modification operations into the
_write-ahead log_. This log may then be read by clients to replay any data
modification on a different server.

**Checking the state**

To query the current state of the _logger_, use the *state* command:
  
    require("@arangodb/replication").logger.state();

The result might look like this:

```js
{
  "state" : {
    "running" : true,
    "lastLogTick" : "2339941",
    "lastUncommittedLogTick" : "2339941",
    "totalEvents" : 2339941,
    "time" : "2019-07-02T10:30:30Z"
  },
  "server" : {
    "version" : "3.5.0",
    "serverId" : "194754235820456",
    "engine" : "rocksdb"
  },
  "clients" : [
    {
      "syncerId" : "158",
      "serverId" : "161976545824597",
      "time" : "1970-01-23T07:59:10Z",
      "expires" : "1970-01-23T09:59:10Z",
      "lastServedTick" : 2339908
    }
  ]
}
```

The *running* attribute will always be true. In earlier versions of ArangoDB the
replication was optional and this could have been *false*. 

The *totalEvents* attribute indicates how many log events have been logged since
the start of the ArangoDB server. The *lastLogTick* value indicates the _id_ of the 
last committed operation that was written to the server's _write-ahead log_.
It can be used to determine whether new operations were logged, and is also used
by the _replication applier_ for incremental fetching of data. The *lastUncommittedLogTick*
value contains the _id_ of the last uncommitted operation that was written to the
server's WAL. For the RocksDB storage engine, *lastLogTick* and *lastUncommittedLogTick* 
are identical, as the WAL only contains committed operations.

The *clients* attribute reveals which clients (Followers) have connected to the
Leader recently, and up to which tick value they caught up with the replication.

**Note**: The replication logger state can also be queried via the
[HTTP API](http/replications.html).

To query which data ranges are still available for replication clients to fetch,
the logger provides the *firstTick* and *tickRanges* functions:

    require("@arangodb/replication").logger.firstTick();

This will return the minimum tick value that the server can provide to replication
clients via its replication APIs. The *tickRanges* function returns the minimum
and maximum tick values per logfile:

    require("@arangodb/replication").logger.tickRanges();

### Replication Applier

**Purpose**

The purpose of the _replication applier_ is to read data from a Leader database's
event log, and apply them locally. The _applier_ will check the Leader database
for new operations periodically. It will perform an incremental synchronization,
i.e. only asking the Leader for operations that occurred after the last synchronization.

The _replication applier_ does not get notified by the Leader database when there
are "new" operations available, but instead uses the pull principle. It might thus
take some time (the so-called *replication lag*) before an operation from the Leader
database gets shipped to, and applied in, a Follower database. 

The _replication applier_ of a database is run in a separate thread. It may encounter
problems when an operation from the Leader cannot be applied safely, or when the
connection to the Leader database goes down (network outage, Leader database is
down or unavailable etc.). In this case, the database's _replication applier_ thread
might terminate itself. It is then up to the administrator to fix the problem and
restart the database's _replication applier_.

If the _replication applier_ cannot connect to the Leader database, or the
communication fails at some point during the synchronization, the _replication applier_
will try to reconnect to the Leader database. It will give up reconnecting only
after a configurable amount of connection attempts.

The _replication applier_ state is queryable at any time by using the *state* command
of the _applier_. This will return the state of the _applier_ of the current database:

```js
require("@arangodb/replication").applier.state();
```

The result might look like this:

```js
{ 
  "state" : { 
    "started" : "2019-03-01T11:36:33Z", 
    "running" : true, 
    "phase" : "running", 
    "lastAppliedContinuousTick" : "2050724544", 
    "lastProcessedContinuousTick" : "2050724544", 
    "lastAvailableContinuousTick" : "2050724546", 
    "safeResumeTick" : "2050694546", 
    "ticksBehind" : 2, 
    "progress" : { 
      "time" : "2019-03-01T11:36:33Z", 
      "message" : "fetching leader log from tick 2050694546, last scanned tick 2050664547, first regular tick 2050544543, barrier: 0, open transactions: 1, chunk size 6291456", 
      "failedConnects" : 0 
    }, 
    "totalRequests" : 2, 
    "totalFailedConnects" : 0, 
    "totalEvents" : 50010, 
    "totalDocuments" : 50000, 
    "totalRemovals" : 0, 
    "totalResyncs" : 0, 
    "totalOperationsExcluded" : 0, 
    "totalApplyTime" : 1.1071290969848633, 
    "averageApplyTime" : 1.1071290969848633, 
    "totalFetchTime" : 0.2129514217376709, 
    "averageFetchTime" : 0.10647571086883545, 
    "lastError" : { 
      "errorNum" : 0 
    }, 
    "time" : "2019-03-01T11:36:34Z" 
  }, 
  "server" : { 
    "version" : "3.4.4", 
    "serverId" : "46402312160836" 
  }, 
  "endpoint" : "tcp://leader.example.org", 
  "database" : "test" 
}
```

The *running* attribute indicates whether the _replication applier_ of the current
database is currently running and polling the Leader at *endpoint* for new events.

The *started* attribute shows at what date and time the applier was started (if at all).

The *progress.failedConnects* attribute shows how many failed connection attempts
the _replication applier_ currently has encountered in a row. In contrast, the
*totalFailedConnects* attribute indicates how many failed connection attempts the
_applier_ has made in total. The *totalRequests* attribute shows how many requests
the _applier_ has sent to the Leader database in total. 

The *totalEvents* attribute shows how many log events the _applier_ has read from the 
Leader. The *totalDocuments* and *totalRemovals* attributes indicate how may document
operations the Follower has applied locally.

The attributes *totalApplyTime* and *totalFetchTime* show the total time the applier
spent for applying data batches locally, and the total time the applier waited on
data-fetching requests to the Leader, respectively.
The *averageApplyTime* and *averageFetchTime* attributes show the average times clocked
for these operations. Note that the average times will greatly be influenced by the
chunk size used in the applier configuration (bigger chunk sizes mean less requests to
the Follower, but the batches will include more data and take more time to create
and apply).

The *progress.message* sub-attribute provides a brief hint of what the _applier_
currently does (if it is running). The *lastError* attribute also has an optional
*errorMessage* sub-attribute, showing the latest error message. The *errorNum*
sub-attribute of the *lastError* attribute can be used by clients to programmatically
check for errors. It should be *0* if there is no error, and it should be non-zero
if the _applier_ terminated itself due to a problem.

Below is an example of the state after the _replication applier_ terminated itself
due to (repeated) connection problems:

```js
{ 
  "state" : { 
    "started" : "2019-03-01T11:51:18Z", 
    "running" : false, 
    "phase" : "inactive", 
    "lastAppliedContinuousTick" : "2101606350", 
    "lastProcessedContinuousTick" : "2101606370", 
    "lastAvailableContinuousTick" : "2101606370", 
    "safeResumeTick" : "2101606350", 
    "progress" : { 
      "time" : "2019-03-01T11:52:45Z", 
      "message" : "applier shut down", 
      "failedConnects" : 6 
    }, 
    "totalRequests" : 19, 
    "totalFailedConnects" : 6, 
    "totalEvents" : 0, 
    "totalDocuments" : 0, 
    "totalRemovals" : 0, 
    "totalResyncs" : 0, 
    "totalOperationsExcluded" : 0, 
    "totalApplyTime" : 0, 
    "averageApplyTime" : 0, 
    "totalFetchTime" : 0.03386974334716797, 
    "averageFetchTime" : 0.0028224786122639975, 
    "lastError" : { 
      "errorNum" : 1400, 
      "time" : "2019-03-01T11:52:45Z", 
      "errorMessage" : "could not connect to leader at tcp://127.0.0.1:8529 for URL /_api/wal/tail?chunkSize=6291456&barrier=0&from=2101606369&lastScanned=2101606370&serverId=46402312160836&includeSystem=true&includeFoxxQueues=false: Could not connect to 'http+tcp://127.0.0.1:852..." 
    }, 
    "time" : "2019-03-01T11:52:56Z" 
  }, 
  "server" : { 
    "version" : "3.4.4", 
    "serverId" : "46402312160836" 
  }, 
  "endpoint" : "tcp://leader.example.org", 
  "database" : "test" 
}
```

**Note**: the state of a database's replication applier is queryable via the HTTP
API, too. Please refer to [HTTP Interface for Replication](http/replications.html)
for more details.

