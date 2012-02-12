---
title: Replication in Redis
layout: post
---

# Replication #
The replication is done by sending each command executed by the master to
all its slaves to run locally. This procedure is in function
`replicationFeedSlaves()`. 

## Slave-side ##

When a Redis instance starts up, it will first synchronously load the available
aof or rdb file into memory, then if this instance is configured to be a
slave of another instance, it will try to connect to the master. This
is done in `replicationCron()`. This cron function is responsible for
reconnecting to master and detecting transfer failure depending on the
current replication state.

When the slave's state is `REDIS_REPL_CONNECT`, the
`replicationCron()` will call `connectWithMaster()`. This function
asynchronously connects to the master and register a callback
`syncWithMaster()` which will be called when the connection is
available. Then the slave's replication state is set to
`REDIS_REPL_CONNECTING`.

When the `syncWithMaster()` is called, the slave will
synchronously send a `SYNC` command to the master and register a
callback `readSyncBulkPayload()` which is triggered when there the
connection are readable. And the slave's replication state is set to
`REDIS_REPL_TRANSTER`, which means the slave is waiting the
transmission to complete.

Then the `readSyncBulkPayload()` will asynchronously read the `SYNC`
payload received from the master and append it to a temp file which
will be renamed to the rdb file when finished. 

Even though the transfered rdb file is loaded into memory
synchronously, the clients are served from time to time. After that,
the slave's replication state is set to `REDIS_REPL_CONNECTED`.


## Master-side##
Upon receiving a `SYNC` command,
the master creates a dump file in the background, which will be sent to
the slave when done. In the mean time, the master accumulates all the
write commands executed during the `BGSAVE` process,
which will be sent after the dump file is sent. 

### Background saving ###
The master will fork a `BGSAVE`
process if no one exists, and set the slave's replication state as 
`REDIS_REPL_WAIT_BGSAVE_END`, which means this slave is waiting the
`BGSAVE` process to finish and 
every write command executed by the master during this period will be
accumnulated in this slave's reply buffer.
 
If there is already one `BGSAVE` in progress
and another slave buffering the commands executed since the master forked,
the current slave can simply reuse the existing slave's
reply buffer and change its state to `REDIS_REPL_WAIT_BGSAVE_END`

Else the server will set the client's state to
`REDIS_REPL_WAIT_BGSAVE_START` and wait for the next `BGSAVE`, because
the commands executed since the `BGSAVE` process forked haven't been
buffered.

The `serverCron()` checks whether there is a finished `BGSAVE` process
every 100 ms and then call `backgroundSaveDoneHandler()`.

This function will call `updateSlavesWaitingBgsave()` with the exit code of
the `BGSAVE` process. 

If the `BGSAVE` process has finished successfully, the dump file will
be sent to each slave waiting in `REDIS_REPL_WAIT_BGSAVE_END` state
using `sendBulkToSlave()`.

If there are any slaves in `REDIS_REPL_WAIT_BGSAVE_START` state, they
will be set to `REDIS_REPL_WAIT_BGSAVE_END` state and another `BGSAVE` will be
started.

### Transtering dump file and accmulated commands to client ###
The `updateSlavesWaitingBgsave()` function registers a file event for 
`sendBulkToSlave()` to send to dump file to the slave asynchronously. 
So the transmission of dump file won't affect the master's performance
to much, but I haven't done benchmark about how long it will take to
sync with a master with tens of G's dataset.

After successfully transfering the dump file, the master
will send the buffered write commands to the slave. Then the slave is
successfully synced with the master.

{% highlight c %}
// redis.h
/* Slave replication state - slave side */
#define REDIS_REPL_NONE 0 /* No active replication */
#define REDIS_REPL_CONNECT 1 /* Must connect to master */
#define REDIS_REPL_CONNECTING 2 /* Connecting to master */
#define REDIS_REPL_TRANSFER 3 /* Receiving .rdb from master */
#define REDIS_REPL_CONNECTED 4 /* Connected to master */

/* Synchronous read timeout - slave side */
#define REDIS_REPL_SYNCIO_TIMEOUT 5

/* Slave replication state - from the point of view of master
 * Note that in SEND_BULK and ONLINE state the slave receives new updates
 * in its output queue. In the WAIT_BGSAVE state instead the server is waiting
 * to start the next background saving in order to send updates to it. */
#define REDIS_REPL_WAIT_BGSAVE_START 3 /* master waits bgsave to start feeding it */
#define REDIS_REPL_WAIT_BGSAVE_END 4 /* master waits bgsave to start bulk DB transmission */
#define REDIS_REPL_SEND_BULK 5 /* master is sending the bulk DB */
#define REDIS_REPL_ONLINE 6 /* bulk DB already transmitted, receive updates */

{% endhighlight %}

![sync](/assets/images/redis/sync.png)

The above graph is about the basic interactions between slave and
master during the `SYNC` process.


