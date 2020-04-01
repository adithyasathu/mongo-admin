MongoDB write concern is an acknowledgment mechanism that developers can add to write operations.

Higher levels of acknowledgment produce a stronger durability guarantee.

Durability means that the write has propagated to the number of replica set member nodes specified in the write concern.

The more replica set members that acknowledge the success of a write, the more likely that the write is durable in the event of a failure.

Majority here is defined as a simple majority of replica set members.

So divide by two, and round up.

The trade-off with increased durability is time.

More durability requires more time to achieve the specified durability guarantee since you have to wait for those acknowledgments.


A write concern of zero means that the application doesn't wait for any acknowledgments.

The write might succeed or fail.

The application doesn't really care.

It only checks that it can connect to the node successfully.

Think of this like a fire-and-forget strategy-- very fast, but with no safety checks in place.

The default write concern is one.

That means the application waits for an acknowledgment from a single member of the replica set, specifically, the primary.

This is a baseline guarantee of success.

Write concerns greater than one increase the number of acknowledgments to include one or more secondary members.

Higher levels of write concern correspond to a stronger guarantee of write durability.

For example, I can set a write concern of three to require acknowledgment from all three replica set members.

Majority is a keyword that translates to a majority of replica set members.

It's a simple majority.

So divide the number of members by two and round up.

So this three-member replica set has a majority of two.

A five-member replica set would have a majority of three-- so on and so forth.

The nice thing with majority is that you don't have to update your write concern if you increase the size of your replica set.

There's also a write concern level for replica set tags.

That's a little advance for this course series, but you can check out our documentation on write concern for more information.

We're just talking about replica sets in this lesson.

But MongoDB supports write concern for both standalone and sharded clusters.

For sharded clusters in particular, write concern is pushed down to the shard level.

Finally, I want to make it very clear that no matter what write concern you specify, MongoDB always replicates data to every data-bearing node in the cluster.

Write concern is just there for you to have a way of tracking the durability of inserted data.

There are two additional write concern options that MongoDB provides.

The first is wtimeout.

This lets you set a maximum amount of time the application waits before marking an operation as failed.

An important note here-- hitting a wtimeout error does not mean that the write operation itself has failed.

It only means that the application did not get the level of durability that it requested.

The second is j, or journal.

This option requires that each replica set member to receive the write and commit to the journal filed for the write to be considered acknowledged.

Starting with MongoDB 3.2.6, a write concern of majority actually implies j equals true.

The advantage of setting j equals true for any given write concern is that you have a stronger guarantee that not only were the writes received, they've been written to an on-disk journal.

If you set j to false, or if journaling is disabled on the mongod, the node only needs to store the data in memory before reporting success.


In general, the core write commands all support write concern.

Now, we've talked about this all at a pretty high level.

Write concern makes a lot more sense when you watch it in action.

In an earlier lesson, we showed you how an application writes "behave" during a failover event.

Let's return to that example now but to talk about write concern.

We have our three-member replica set with an application inserting data into the primary.

As you might remember, data is replicated from the primary to the secondaries.

The default write concern is one.

So even though we don't set the write concern explicitly, the application implicitly assigns it a write concern of one.

At this point, we know that our primary has received the write operation.

We've received one acknowledgment based on our write concern of one.

Since that matches the write concern we requested, we're good to go.

But what would happen if the primary failed at this point of time before it completes replicating this write to the secondaries?

The secondaries won't see the write.

While it was successful on a single node, we had no guarantee that the write had propagated to the remaining replica set members.

Even though we got the level of write concern we asked for, the guarantee level was too low to accommodate the scenario.

As far as our application is concerned, this was a successful write.

But when this primary comes back online the right would actually be rolled back.

Let's imagine the same scenario but with a write concern of majority.

That means we now would need an acknowledgment from two members of our three-member replica set.

The application waits until the primary reports that at least one of the secondaries has also acknowledged the write.

With two acknowledgments the application can consider the write a success.

Now if the primary happens to go down, we have a reasonable guarantee that at least one of the secondaries has received the write operation.

If I set the write concern to three, the application would require an acknowledgment from all three members before considering the write successful.

At this point, you might be thinking, well, I want guarantees of data durability.

So shouldn't I always set my write concern as high as possible?

Well, a stronger level of durability guarantee comes with the trade off.

Specifically, you have to wait for the acknowledgments to come in.

That means your write operations may take longer than required if you requested a lesser or even no write acknowledgment.

As the number of replica set nodes increase, write operations may take even longer to return as successful.

Alternatively, what happens if one of your secondaries are down?

Based on our write concern, we need an acknowledgment from three members of the replica set.

This write now blocks until the secondary comes back online which may take longer than is acceptable.

This is where the W timeout option comes in handy.

You can set it to a reasonable time wherein the application stops waiting and moves forward throwing an error related to write concern.

Remember, the timeout error does not mean that the write failed.

We can see here that the primary and at least one secondary did acknowledge.

But because we timed out waiting for the requested write concern, as far as the application is concerned, it did not receive the level of guarantee it requested.

Generally, setting a W of majority is a nice middle ground between fast writes and durability guarantees.

**RECAP** write concern allows your applications to request a certain number of acknowledgments for a write operation for the MongoDB cluster.

These acknowledgments represent increasing durability guarantees for a given write operations.

Write concern comes with a trade off of write speed.

The higher guarantee you need that a right is durable, the more time the overall write operation requires to complete.

MongoDB supports a write concern on all deployment types.

That means standalone, replica sets, and sharded clusters all support write concern.
