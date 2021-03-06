﻿# Understanding Eventual Consistency

Think of what actually happens with a traditional database like SQL Server.

- When an item is created, updated, or deleted from a table, any indexes associated with table also have to be updated.
- The more indexes you have on a table, the slower your write operations will be.
- If you create a new index on an existing table, it isn't used at all until it is fully built.  If no other index can answer a query, then a slow table scan occurs.
- If others attempt to query from an existing index while it is being modified, the reader will *block* until the modification is complete, because of the requirement for **Consistency being higher priority than Availability**
- This can often lead to slow reads, timeouts, and deadlocks.

The NoSQL concept of "Eventual Consistency" is designed to alleviate these concerns.  It is optimized for reads by prioritizing **Availability higher than Consistency**.  RavenDB is not unique in this regard, but it is somewhat special in that it still has the ability to be consistent.  If you are retrieving single document, such as reviewing an order or an end user viewing their profile, these operations are ACID compliant, and are not affected by the "eventual consistency" design.

To understand "eventual consistency", think about a typical user looking at a list of products on your web site.  At the same time, the sales staff of your company is modifying the catalog, adding new products, changing prices, etc.  One could argue that it's probably not super important that the list be fully consistent with these changes.  After all, a user visiting the site a couple of seconds earlier would have received data without the changes anyway.  The most important thing is to deliver product results quickly.  Blocking the query because a write was in progress would mean a slower response time to the customer, and thus a poorer experience on your web site, and perhaps a lost sale.

So, in RavenDB:

- Writes occur against the *document store*.
- Single `Load` operations go directly to the *document store*.
- Queries occur against the *index store*
- As documents are being written, data is being copied from the document store to the index store, for those indexes that are already defined.
- At any time you query an index, you will get whatever is already in that index, regardless of the state of the copying that's going on in the background.  This is why sometimes indexes are "stale".
- If you query without specifying an index, and Raven needs a new index to answer your query, it will start building an index on the fly and return you *some* of those results right away.  It only blocks long enough to give you one page of results.  It then continues building the index in the background so next time you query you will have more data available.

So now lets give an example that shows the down side to this approach.

- A sales person goes to a "products list" page that is sorted alphabetically.
- On the first page, they see that "Apples" aren't currently being sold.
- So they click "add product", and go to a new page where they enter "Apples".
- They are then returned to the "products list" page and they still don't see any Apples because the index is stale.  What the hack - right?

Addressing this problem requires the understanding that not all viewers of data should be considered equal.  That particular sales person might demand to see the newly added product, but a customer isn't going to know or care about it with the same level of urgency.

So on the "products list" page that the sales person is viewing, you might do something like:

{CODE userissues_1@UsersIssues\UnderstandingEventualConsistency.cs /}

While on the customer's view of the catalog, you would *not* want to add that customization line.

If you wanted to get super precise, you could use a slightly more optimized strategy:

- When saving the "add product" page before going back to the "list products" page, call `WaitForIndexesAfterSaveChanges` to ensure that the changes made in the session are already indexed.

{CODE userissues_2@UsersIssues\UnderstandingEventualConsistency.cs /}

- This will ensure that you only wait as long as absolutely necessary to get just that one product's changes included in the results list along with the other results from the index.
