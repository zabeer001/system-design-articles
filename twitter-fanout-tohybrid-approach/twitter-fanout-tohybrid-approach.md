## Twitter Feed Design — From Read-Heavy to Hybrid Approach

Twitter’s feed architecture is a classic example from **Designing Data-Intensive Applications (DDIA)** of the trade-off between **fan-out on read** and **fan-out on write**.

---

## 1. First Approach: Fan-out on Read

In the first approach, Twitter built the Home Timeline when the user requested it.

Suppose **Jabir follows 1,000 users**.

When Jabir opens Twitter:

```text
Jabir requests Home Feed
        ↓
Read Jabir's following list
        ↓
Find the 1,000 users Jabir follows
        ↓
Read recent tweets from those users
        ↓
Merge all tweets
        ↓
Sort / rank them
        ↓
Return Jabir's timeline
```

### Why is this called Fan-out on Read?

**Fan-out** means one operation expands into many operations.

Jabir makes only **one feed request**, but Twitter may need to access data related to hundreds or thousands of accounts.

```text
                    Jabir
                      │
                      │ One feed request
                      ↓
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
      Alice          Bob           Sara
        ↓             ↓             ↓
    Read tweets    Read tweets   Read tweets
        \             |             /
         \            |            /
          └──── Merge + Sort ─────┘
                    ↓
               Jabir's Feed
```

The fan-out happens **during the read request**, so it is called:

**Fan-out on Read.**

---

## 2. Why Was It Read-Heavy?

This is the most important part.

One user reading the feed does not necessarily result in only one simple database read.

Instead:

```text
1 feed request
      ↓
Read following list
      ↓
Read tweets from many followed users
      ↓
Merge results
      ↓
Sort/rank results
```

So:

```text
1 logical feed read
        ↓
Many underlying data reads
```

For example:

```text
Jabir follows 1,000 users
```

Conceptually, Twitter may need to gather tweets from many of them:

```text
Read Alice's tweets
Read Bob's tweets
Read Sara's tweets
Read John's tweets
Read Maria's tweets
...
```

The exact implementation does not have to literally execute 1,000 separate SQL queries, but the system still has to retrieve and process data belonging to many followed accounts.

That is what makes the **read path expensive**.

### Repeated reads make it worse

Suppose Jabir refreshes Twitter 20 times during the day.

With fan-out on read:

```text
Refresh #1
→ gather tweets
→ merge
→ sort

Refresh #2
→ gather tweets again
→ merge again
→ sort again

Refresh #3
→ gather tweets again
→ merge again
→ sort again
```

And so on.

Therefore:

```text
20 feed requests
×
many underlying reads
×
merge + ranking work
```

Twitter users generally **read timelines much more frequently than they publish tweets**.

A user might:

```text
Open Twitter
Scroll
Refresh
Open again
Refresh
Scroll again
```

while publishing only one or two tweets.

Because of this workload pattern, putting expensive work on every timeline read becomes inefficient at large scale.

This is why the first architecture was **read-heavy**.

---

## 3. Write Path Was Very Cheap

The advantage of fan-out on read was that publishing a tweet was simple.

When Alice tweets:

```text
Alice publishes Tweet #500
        ↓
Store Tweet #500
        ↓
Done
```

Twitter does not immediately need to update all of Alice's followers.

So:

```text
Fan-out on Read

Write path:
Tweet → Store

Very cheap
```

But:

```text
Read path:
Find followed users
→ fetch tweets
→ merge
→ sort
→ return timeline

Expensive
```

The architectural problem was therefore not simply "fan-out is bad."

The problem was:

> **The fan-out was happening on Twitter's busiest path: timeline reads.**

---

## 4. Twitter Moved Toward Fan-out on Write

Twitter changed the architecture so that much of the work happened when a tweet was created.

Suppose Alice has these followers:

```text
Alice
 ├── Jabir
 ├── Bob
 └── Sara
```

Alice publishes a tweet:

```text
Alice publishes Tweet #500
            ↓
       Store Tweet
            ↓
       Fan-out service
            ↓
    ┌───────┼────────┐
    ↓       ↓        ↓
 Jabir     Bob      Sara
timeline timeline timeline

 #500      #500      #500
```

Instead of waiting until Jabir asks for his feed, Twitter places a reference to Alice's tweet into Jabir's prepared timeline.

Now when Jabir opens Twitter:

```text
Jabir requests Home Feed
        ↓
Read Jabir's prepared timeline
        ↓
Return feed
```

The read path becomes much simpler.

---

## 5. The Cost Was Moved from Read to Write

This is the central trade-off.

### Fan-out on Read

```text
Alice tweets
     ↓
Store tweet
     ↓
Done
```

Later:

```text
Jabir opens feed
     ↓
Find followed users
     ↓
Read many tweets
     ↓
Merge
     ↓
Sort
     ↓
Return feed
```

So:

```text
Write = Cheap
Read  = Expensive
```

### Fan-out on Write

```text
Alice tweets
     ↓
Store tweet
     ↓
Find followers
     ↓
Push tweet reference into follower timelines
```

Later:

```text
Jabir opens feed
     ↓
Read prepared timeline
     ↓
Return
```

So:

```text
Write = More expensive
Read  = Much cheaper
```

Twitter accepted more work during writes because timeline reads were much more frequent.

---

## 6. New Problem: Celebrities

Fan-out on write works well for ordinary accounts.

Suppose Alice has:

```text
500 followers
```

When Alice tweets:

```text
1 tweet
   ↓
500 timeline updates
```

That is manageable.

Now imagine a celebrity has:

```text
50,000,000 followers
```

One tweet could theoretically cause:

```text
1 tweet
   ↓
50,000,000 timeline updates
```

This creates huge **write amplification**.

One celebrity tweet could create enormous fan-out work.

```text
Celebrity publishes tweet
           ↓
        Fan-out
           ↓
Millions of follower timelines
```

This could overload queues, workers, cache infrastructure, and storage systems.

So pure fan-out on write also has a weakness.

---

## 7. Twitter's Hybrid Approach

Twitter therefore uses a hybrid idea.

For ordinary users:

```text
Normal user tweets
        ↓
Fan-out on Write
        ↓
Push tweet into follower timelines
```

For users with huge follower counts:

```text
Celebrity tweets
        ↓
Store tweet
        ↓
Do not immediately push it
to millions of timelines
```

When Jabir requests his feed:

```text
Jabir's prepared timeline
           +
Tweets from celebrities
Jabir follows
           ↓
       Merge / Rank
           ↓
       Final Feed
```

Architecture:

```text
                 Twitter Hybrid Feed

Normal Accounts                     Celebrity Accounts
      │                                    │
      ↓                                    ↓
Fan-out on Write                     Fan-out on Read
      │                                    │
      ↓                                    ↓
Prepared Timeline                  Celebrity Tweets
           \                           /
            \                         /
             └────── Merge ─────────┘
                       ↓
                  Final Timeline
```

---

## 8. Why Hybrid Is Better

Twitter gets the advantages of both approaches.

For most users:

```text
Precompute timeline
        ↓
Fast timeline reads
```

For celebrity accounts:

```text
Avoid millions of writes
        ↓
Fetch their tweets at read time
```

Therefore Twitter avoids both extremes:

```text
Pure fan-out on read
→ too much work on every feed request

Pure fan-out on write
→ huge write amplification for celebrities
```

The hybrid model handles the common case efficiently while treating exceptional users differently.

---

## 9. DDIA's Core Lesson

The main lesson from DDIA is not:

> Fan-out on write is always better.

The lesson is:

> **Architecture should depend on the workload and access pattern.**

If reads dominate:

```text
Move/precompute some work toward writes
```

If a write could affect millions of users:

```text
Avoid fully precomputing that exceptional case
```

Twitter therefore uses approximately this strategy:

```text
Normal users
→ Fan-out on Write

High-follower users
→ Fan-out on Read

Final timeline
→ Merge both
```

This is a strong example of a general system-design principle:

**Optimize the common case, but handle extreme cases separately.**
