# Clustering : Replication

Replication in RavenDB is the process of transferring _data_ from one database to another.  

* We consider those elements to be _data_:  
  * Documents  
  * Revisions  
  * Attachments  
  * Conflicts  

{PANEL: How does replication works}
The replication process sends _data_ over a _TCP connection_ by the modification order, meaning that the oldest modification on the node gets sent first.  
Internally we keep an [ETag](../../glossary/etag) per data element, which is an incrementing integer value, and this is how we are able to determine what is the order of modifications.  
Each replication process has a _source_ , _destination_ and a last confirmed _ETag_ which is basically a cursor to where we have gotten to in the replication process.  
The _data_ is sent from the _source_, in batches, to the _destination_.  
When the _destination_ confirms getting the _data_ the last accepted etag is then advanced and the next batch of _data_ is sent.  
In case of failure of the process, it will re-start with the [initial handshake procedure](../clustering/replication#replication-handshake-procedure), which will make sure we will start replicating from the last accepted _ETag_.
{PANEL/}

{PANEL: Replication Handshake Procedure}
Whenever the replication process between two databases start it has to determine the process state, the same logic applies for new or re-established replication.  
The first message has been sent is a request to establish a TCP connection of type _replication_ with a _protocol version_.  
At this point, the _destination server_ verifies that the protocol version matches and that the request is authorized.  
Once the _source_ gets the OK message it queries the _destination_ about the latest ETag it got from him.  
The _destination_ sends back a _heartbeat_ message with both the latest _ETag_ he got from the _source_ and the current [change vector](../clustering/change-vector) of the database.  
The _ETag_ is used as a starting point for the replication process but it is then been filtered by the _destination's_ current _change vector_,
meaning we will skip documents with higher _Etag_ and lower _change vector_, this is done so we won't send the same data from multiple sources, you can read more about this [below](../clustering/replication#preventing-the-ripple-effect).  
{PANEL/}

{PANEL: Preventing the ripple effect}
RavenDB [_database group_](../../glossary/database-group) is a fully connected graph of replication channels, meaning that if there are `n` nodes in a _database group_ there are `n*(n-1)` replication channels.  
We wanted to prevent the case where inserting data into one database will cause the data to propagate multiple times through multiple paths to all the other nodes.  
We have managed to do so by delaying the propagation of data coming from the replication logic itself.  
If the sole source of incoming data is replication we will not replicate it right away, we will wait up to `15 seconds` before sending data.  
This will allow the destination to inform us about his current change vector and most of the time the data will get filtered at the source.  
On a stable system the steady state will have a _spanning tree_ of replication channels, `n-1` of the fastest channels available that are doing the actual work and the rest are just sending heartbeats.  

How does this work in a failure scenario?  
Let's say that we have a three node _database group_ with nodes `A`, `B` and `C` and that usually `A` replicates to `B` and `C`, `B` and `C` only send heartbeats to each other.
Suddenly there is a communication issue where `A` can't communicate with `B` but `C` is still able to communicate with `B`.
What will happen is that `A` will replicate data to `C` and when its time for `C` to send a heartbeat to `B`, `B's` response will include its [global change vector](../clustering/database-global-change-vector) and `C` will realize it has data to send over.
When `A` re-establish communication with `B` it will be informed about `B` _global change vector_ and won't send over data that is already present in `B`.
{PANEL/}

{PANEL: Replication transaction boundary}
In RavenDB we extended the boundry of a transaction across multiple nodes.  
Let say that you want to modify two documents `X` and `Y` in the same transaction and the commit happens on node `A`, what would happen on node `B`?  
Let's say that `X` is a _Bitcoin_ wallet and `Y` is another _Bitcoin_ wallet and we want to move 1 _Bitcoin_ from `X` to `Y`.  
After the execution of the transaction we will end up with `X'` = `X-1` and `Y'` = `Y+1`, simple enough.  
Now if the data is beeing replicated to another node we will extend the transaction atomicity across multiple nodes.  

What will happen on the server side, node `A`, transaction #17 will contain modification to `X'` with ETag 1024 and modification to `Y'` with ETag 1025.  
Let's say that our replication batch sends all documents from ETag 1 to 1024.  
But since we respect the atomicity of a transaction across nodes we will include the entire data modified by transaction #17 meaning `Y'` that has ETag 1025 would also be sent in the same batch.  
If we would not do so, while node `A` is preparing to send the next replication batch server `B` would have shown that `X'` has one fewer _Bitcoin_ while `Y` remains the same so the total _Bitcoin_ between `X'` and `Y` just droped by one.  

This sounds great but there is one caveat what will happen if `Y'` is modified by another transaction, transaction #18, before the replication sends it over to node `B`?  
RavenDB replication process only sends the latest state of the data meaning that even though `X` and `Y` are modified in the same transaction they may be replicated in diffrent batches if one of them was modified in the meanwhile.  
Consider the case before with the _Bitcoin_ wallet, let's say that `Y'` also needs to moves 0.1 _Bitcoin_ to `Z` and so we end up with another transaction, #18, modifying `Y'`.    
We already commited transaction #17 and modify `Y` to `Y'` now transaction #18 will be deducting 0.1 _Bitcoin_ from `Y'` and we will end up with `Y''` = `Y+0.9`.  
Now what could happen is that the replication batch will send `X'` without `Y''` since it was not modified in the same transaction and node `B` will have data inconsistency issues again.  
Luckily not all is lost, you could enable [revisions](../revisions) and thus we will replicate the revisions alongside the documents and this way `X'` will be sent with `Y'` which is the revision of `Y''` after we added one _Bitcoin_ to it but before we transferred the 0.1 _Bitcoin_ to `Z`.  
Meaning that with revisions enabled we are actually sending the snapshot of the data at the end of the transaction and not just the latest state of the data.  

{PANEL/}

{PANEL: Rehab state and database group observer}

In a [database group](../../glossary/database-group) we might have a node that has a much lower [global change vector](../clustering/database-global-change-vector), and currently being updated by one of the other nodes in the _database group_.  
This may happen for a few reasons, it was just added, it was down for a while or it was just restored from a backup snapshot.  
We consider such node to be in a _rehab_ state until it caught up with the rest of the _database group_.  
While a node is in a _rehab_ state it may not be assigned any tasks such as _backup_, _ETL_ or _subscriptions_.  

Every _cluster_ has an _observer_, the _observer_ is run by the _cluster leader_ and monitors the _database groups_.  
The _observer_ periodically pings all nodes and checks that they are both up and that their data is up to date.  
If the _observer_ notice that a node is down or lagging behind it will mark it as in _rehab_ state and thus it will not get assigned new tasks.  
Moreover, if you have an _enterprise license_ then tasks that were assigned to that node will get re-assigned to other nodes in the _database group_.   
{PANEL/}

{PANEL: External Replication}
We call the task of replicating between two different database groups _external replication_.  
There is actually no difference in the implementation of an _external replication_ and a regular replication process.  
The reason we define them differently is because of the default behavior of a cluster to set up a fully connected _database groups_, meaning all nodes are communicating all other nodes.  
This may be limiting if you wish to design your own replication topology and _external replication_ is the solution for those unique cases.  

{NOTE: Note}
RavenDB clients will not failover to externaly replicated nodes, failover of clients is something that is done within the _database group_ members only.
{NOTE/}

{PANEL/}

{PANEL: Delayed Replication}
In RavenDB we introduced a new kind of replication, _delayed replication_, what it does is replicating data that is delayed by `X` amount of time.  
The _delayed replication_ works just like normal replication but instead of sending data right away it waits `X` amount of time.  
Having a delayed instance of a database allows you to "go back in time" and undo contamination to your data due to a faulty patch script or other human errors.  
While you can and should always use backup for those cases, having a live database makes it super fast to failover to and prevent business lose while you take down the faulty databases and restore them.  
{PANEL/}
