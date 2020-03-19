sample mongo.conf with replication set enables

```
storage:
  dbPath: /var/mongodb/db/node1
net:
  bindIp: 192.168.103.100,localhost
  port: 27011
security:
  authorization: enabled
  keyFile: /var/mongodb/pki/m103-keyfile  # key file shared by all replication node instances
systemLog:
  destination: file
  path: /var/mongodb/db/node1/mongod.log
  logAppend: true
processManagement:
  fork: true
replication:  ## replication with set name
  replSetName: m103-example
  ```
  
### oplog.rs in local db

oplog.rs is the central point of our replication mechanism.

This is the oplog collection that will keep track of all statements being replicated in our replica set.

Every single piece of information and operations that need to be replicated will be logged in this collection.

oplog.rs collection is a capped collection.

Capped collection means that the size of this collection is limited to a specific size.

If we collect the stats of our oplog.rs collection into this variable, there's a flag called .capped that will tell us that this collection is, indeed, capped.

```
use local
show collections
```
Querying the oplog after connected to a replica set:
```
use local
db.oplog.rs.find()
```
Getting information about the oplog. Remember the oplog is a capped collection, meaning it can grow to a pre-configured size before it starts to overwrite the oldest entries with newer ones. The below will determine whether a collection is capped, what the size is, and what the max size is.

Storing oplog stats as a variable called stats:
```
var stats = db.oplog.rs.stats()
 ```
Verifying that this collection is capped (it will grow to a pre-configured size before it starts to overwrite the oldest entries with newer ones):
```
stats.capped
```
Getting current size of the oplog:
```
stats.size
```
Getting size limit of the oplog:
```
stats.maxSize
```
Getting current oplog data (including first and last event times, and configured oplog size):
```
rs.printReplicationInfo()
```

Create new namespace m103.messages:
```
use m103
db.createCollection('messages')
```
Query the oplog, filtering out the heartbeats ("periodic noop") and only returning the latest entry:
```
use local
db.oplog.rs.find( { "o.msg": { $ne: "periodic noop" } } ).sort( { $natural: -1 } ).limit(1).pretty()
```

As operations get logged into the oplog, like inserts or deletes or create collection kind of operations, the oplog.rs collection starts to accumulate the operations and statements, until it reaches the oplog size limit.

Once that happens, the first operations in our oplog start to be overwritten with newer operations.

The time it takes to fill in fully our oplog and start rewriting the early statements determines the replication window.

The replication window is important aspect to monitor because we'll impact how much time the replica set can afford a node to be down without requiring any human intervention to auto recover.

Every node in our replica set has its own oplog.

As writes and operations gets to the primary node, these are captured in the oplog.

And then the secondary nodes replicate that data and apply it on their own oplog.

Another aspect to know about this collection is that given the idempotent nature of the instructions, one single update may result in several different operations in this collection.

good thing is that our oplog.rs size can be changed.

#### changing conf without restarting node

Assigning the current configuration to a shell variable we can edit, in order to reconfigure the replica set:
```
cfg = rs.conf()
```
Editing our new variable cfg to change topology - specifically, by modifying cfg.members:
```
cfg.members[3].votes = 0
cfg.members[3].hidden = true
cfg.members[3].priority = 0
```
Updating our replica set to use the new configuration cfg:
```
rs.reconfig(cfg)
```


Enabling read commands on a secondary node:
```
rs.slaveOk()
```
* Connecting to the replica set will automatically connect to a primary node.
* We cannot write data directly to a secondary.
* When all secondary nodes shutdown/unhealthy, last node(primary) would step down to become a secondary when a majority of nodes in the set were not available.This is actually fail safe protection for data integrity.
* Mongo node maintainence like version upgrade can be done without restarting or downtime by upgrading the secondaries first and calling `rs.stepDown()` on primary which would force an election and one of the secondary become primary and the former primary can go though maintanence.

##### Change priority to force election 
   
 Storing replica set configuration as a variable cfg:
```
cfg = rs.conf()
```
Setting the priority of a node to 0, so it cannot become primary (making the node "passive"):
```
cfg.members[2].priority = 0
```
Updating our replica set to use the new configuration cfg:
```
rs.reconfig(cfg)
```
Checking the new topology of our set:
```
rs.isMaster()
```
Forcing an election in this replica set (although in this case, we rigged/cheated the election so only one node could become primary):
```
rs.stepDown()
```
Checking the topology of our set after the election:
```
rs.isMaster()
```
