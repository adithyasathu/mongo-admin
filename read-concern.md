Read concern is a companion to write concern, an acknowledgment mechanism for read operations where developers can direct their application to perform read operations where the returned documents meet the requested durability guarantee.

Consider a replica set where applications insert a document, which is acknowledged by the primary.

Before the document can replicate to the secondaries, the application reads that document.

Suddenly, the primary fails.

The document we have read has not yet replicated to the secondaries.

When the old primary comes back online, that data is going to be rolled back as part of the synchronization process.

While you can access the rolled back data manually, the application effectively has a piece of data that no longer exists in the cluster.

This might cause problems in the application, depending on your architecture.

Read concern provides a way of dealing with the issue of data durability during a failover event.

Just like how write concern lets you specify how durable write operations should be, read concern lets your application specify a durability guarantee for documents written by a read operation.

The read operation only returns data acknowledged as having been written to a number of replica set members specified in the read concern.

You can choose between returning the most recent data in a cluster or the data received by a majority of members in the cluster.

One really important note.

A document that does not meet the specified read concern is not a document that is guaranteed to be lost.

It just means that at the time of the read, the data had not propagated to enough nodes to meet the requested durability guarantee.

There are four read concern levels.

Local returns the most recent data in the cluster.

Any data freshly written to the primary qualifies for local read concern.

There are no guarantees that the data will be safe during a failover event.

Local reads are the default for read operations against the primary.

Available read concerns the same as local read concern for replica set deployments.

Available read concern is the default for read operations against secondary members.

Secondary reads are an aspect of MongoDB read preference, which is covered in a later lesson.

The main difference between local and available read concern is in the context of sharded clusters.

We're going to talk more about sharded clusters later.

It's enough to know that available read concern has special behavior in sharded clusters.

Majority read concern only returns data that has been acknowledged as written to a majority of replica set members.

So here in our three-member replica set, our read operations would only return those documents written to both the primary and the secondary.

The only way that documents returned by a read concern majority read operation could be lost is if a majority of replica set members went down and the documents were not replicated to the remaining replica set members, which is a very unlikely situation, depending on your deployment architecture.

Majority read concern provides the stronger guarantee compared to local or available writes.

But the trade-off is that you may not get the freshest or latest data in your cluster.

As a special note, the MMAPv1 storage engine does not support write concern of majority.

Linearizeable read concern was added in MongoDB 3.4, and has special behavior.

Like read concern majority, it also only returns data that has been majority committed.

Linearizeable goes beyond that, and provides read your own write functionality.

So when should you use what read concern?

Really depends on your application requirements.

Fast, safe, or the latest data in your cluster.

Pick two.

That's the general idea.

If you want fast reads of the latest data, local or available read concern should suit you just fine.

But you are going to lose your durability guarantee.

If you want fast reads of the safest data, then majority read concern is a good middle ground.

But again, you may not be getting the latest written data to your cluster.

If you want safe reads of the latest data at the time of the read operation, then linearizeable is a good choice, but it's more likely to be slower.

And it's single document reads only.

One thing to be really clear about, read concern doesn't prevent deleting data using a CRUD operation, such as delete.

We're only talking about the durability of data returned by a read operation in context of a failover event.

In a later lesson, we're going to talk about read preference, where you can route your read operations to secondary members.

Read preference for secondaries does allow you to specify local or available read concern, but you're going to lose the guarantee of getting the most recent data.

So just remember that with secondary reads, local and available read concern is really just fast, but not necessarily the latest.

  <img width="894" alt="Screen Shot 2020-04-01 at 10 51 54 PM" src="https://user-images.githubusercontent.com/22003086/78134418-db6e8f00-746b-11ea-9631-b6687dddc458.png">

 
**RECAP** read concern uses an acknowledgment mechanism for read operations where applications can request only that data which meets the requested durability guarantee.

There are four available read concern options.

Local and available are identical for replica sets.

Their behavior's more distinct in sharded clusters.

Majority read concern is your middle ground between durability and fast reads, but it can return stale data.

Linearizeable has the strongest durability guarantees, and always returns the freshest data in context of a read operation in exchange for potentially slow read operations as well as the overall restrictions and requirements for using it.

Finally, you can use read concern in conjunction with write concern to specify durability guarantees on both reads and writes.

Remember that the two work together, but are not dependent on each other.


**Read preference** allows your applications to route read operations to specific members of a replica set.

Read preference is principally a driver side setting.

Make sure to check your driver documentation for complete instructions on specifying a read preference for your read operations.

Take this three member replica set, for example.

By default, your applications read and write data on the primary.

With replica sets, data is replicated to all data bearing members.

So both of these secondaries would eventually have copies of the primary data.

What if we, instead, wanted to have our application prefer reading from a secondary member?

With the read preference, we can direct the application to route its query to a secondary member of the replica set, instead of the primary.

There are five supported read preference modes.

Primary read preference routes all read operations to the primary only.

This is the default read preference.

PrimaryPreferred routes read operations to the primary.

But if the primary is unavailable, such as during an election or fail-over event, the application can route reads to an available secondary member instead.

Secondary routes read operations only to the secondary members in the replica set.

SecondaryPreferred routes read operations to the secondary members.

But if no secondary members are available, the operation then routes to the primary.

Nearest routes read operations to the replica set member with the least network latency to the host, regardless of the members type.

This typically supports geographically local read operations.

With secondary reads, always keep in mind that depending on the amount of replication latency in a replica set, you can receive stale data.

For example, let's say this replica set receives a write operation that updates this document, where name is Mongo 101, to have a status of published.

At this time, the secondary still has the old version of this document where the status is pending.

If I issue a secondary read, using nearest read preference selects the secondary member to read from, I'm going to get an older version of this document.

This table gives you an idea of some of the scenarios where you'd use a given read preference.

The big takeaway here, is that secondary reads always have a chance of returning stale data.

How stale that data is, depends entirely on how much of a delay there is between your primary and your secondaries.

Geographically distributed replica sets are more likely to suffer from stale reads, for example, than a replica set where all the members are in the same geographic region, or even the same data center.

<img width="916" alt="Screen Shot 2020-04-01 at 10 58 46 PM" src="https://user-images.githubusercontent.com/22003086/78134933-b1699c80-746c-11ea-8c7a-f1ade7f9a039.png">


**RECAP** read preference lets you choose what replica set members to route read operations to.

The big drawback of using a read preference, other than primary, is the potential for stale read operations.

And the nearest read preference is most useful if you want to support geographically local reads.

Just remember that it can come with the potential of reading stale data.

You can use read concern with write concern or read concern without write concern.
