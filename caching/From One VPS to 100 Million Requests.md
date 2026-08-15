# From One VPS to 100 Million Requests: Redis, AWS Networking, Caching, and Scaling

When we first learn Redis, the explanation is usually simple:

> Redis stores data in memory, so it is fast.

That is correct, but it doesn't explain what happens when a real application grows.

What happens when we have multiple EC2 instances? Why does local file caching become difficult? Why do we need shared Redis? What happens when Redis itself becomes overloaded? And what if 90% of the traffic requests the same homepage?

This is where Redis becomes a **system-design problem**, not just a caching technology.

---

## 1. Start With a Normal Application on One VPS

Suppose we have a Laravel e-commerce application running on a Hostinger VPS.

```text
Hostinger VPS
│
├── Laravel
├── MySQL
└── Redis
```

Redis can simply run as another Docker container.

```text
Laravel container
      ↓
Redis container
      ↓
MySQL container
```

There is no requirement to purchase Redis Cloud or another external Redis service.

Redis is running on infrastructure we already pay for.

The important benefit is that Redis can reduce database load.

Without Redis:

```text
Request
   ↓
Laravel
   ↓
MySQL
```

Imagine the homepage needs the same products repeatedly.

```text
SELECT * FROM products
WHERE featured = 1;
```

If 10,000 requests arrive, we don't necessarily want MySQL executing essentially the same expensive work 10,000 times.

Instead:

```text
Request
   ↓
Laravel
   ↓
Redis
   ↓ cache miss
MySQL
   ↓
Redis stores result
```

After that:

```text
Request
   ↓
Laravel
   ↓
Redis
   ↓
Result
```

Redis's cache-aside pattern is specifically useful for repeated reads: the application checks Redis, falls back to the primary database on a miss, and puts the result into Redis for subsequent requests.

So even **Redis running on the same VPS** can be valuable.

---

# 2. Now the Application Becomes a Microservice Architecture

Suppose our business grows.

We now have:

* HR service
* Accounts service
* E-commerce service
* Redis
* Database

Instead of putting everything on one machine, we start using AWS.

We create a VPC:

```text
VPC
192.168.0.0/16
```

Inside the VPC, we divide our infrastructure into subnets.

```text
VPC: 192.168.0.0/16

Public Subnets
└── Load Balancer

Private HR Subnet
└── HR EC2

Private Accounts Subnet
└── Accounts EC2

Private Redis Subnet
└── Redis

Private Database Subnet
└── Database
```

For example:

```text
HR
192.168.10.11

Accounts
192.168.20.11

Redis
192.168.30.10

Database
192.168.40.10
```

These machines are in different subnets, but because they belong to the same VPC and the routing/security configuration permits it, they can communicate using private networking.

The public traffic flow becomes:

```text
Internet
    ↓
Load Balancer
    ↓
Application EC2 instances
    ↓
Redis
    ↓
Database
```

Redis and the database do **not** need to be exposed directly to the internet.

---

# 3. Why Redis Becomes More Important When We Auto-Scale

Suppose traffic increases.

Originally we had:

```text
Load Balancer
      ↓
   App EC2 #1
```

Now one EC2 cannot handle all the requests.

So we scale:

```text
                  Load Balancer
                       ↓
             ┌─────────┼─────────┐
             ↓         ↓         ↓
           App #1    App #2    App #3
```

This introduces an interesting caching problem.

Suppose we were using Laravel's local file cache.

App #1 could contain:

```text
homepage_products.cache

A
B
C
```

But that file exists on **App #1's filesystem**.

App #2 has its own filesystem.

App #3 has another filesystem.

Therefore:

```text
App #1
└── Local Cache #1

App #2
└── Local Cache #2

App #3
└── Local Cache #3
```

The caches aren't naturally shared.

This becomes particularly problematic for things such as:

* sessions
* rate limits
* distributed locks
* counters
* queues
* shared application cache

Redis gives us shared state:

```text
App #1 ─────┐
            │
App #2 ─────┼────→ Redis
            │
App #3 ─────┘
```

Now whichever EC2 receives the request can access the same Redis data.

Redis's own documentation points out this distinction: an in-process/local cache can work for one instance, but multiple stateless services warm independently and cannot consistently invalidate one another's caches.

This is one reason Redis becomes especially valuable once **load balancing and horizontal application scaling** enter the architecture.

---

# 4. Redis Protects the Database

Imagine our system receives:

```text
100 million requests/second
```

This is an extreme example, but it makes the architecture easier to understand.

Suppose our cache hit rate is 90%.

Conceptually:

```text
100M requests/sec
       ↓
Application
       ↓
      Redis

90M → cache hits
10M → cache misses / uncached operations
```

Instead of the database being asked to satisfy all 100 million reads, Redis absorbs a huge portion of the repeated read traffic.

Conceptually:

```text
100M requests
     ↓
   Redis
   ↙   ↘
 HIT   MISS
90M     10M
         ↓
      Database
```

The exact numbers in a real application depend on workload, writes, cache policy, TTLs, invalidation strategy, and many other factors. But the principle is what matters:

**A high cache-hit ratio can dramatically reduce pressure on the primary database.**

---

# 5. But Now Redis Has a Problem

We solved:

```text
Database overloaded
```

by introducing Redis.

But now imagine:

```text
90M requests/sec
       ↓
   Redis #1
```

Redis #1 itself becomes the bottleneck.

This is an important system-design pattern:

```text
Problem
   ↓
Solution
   ↓
Solution becomes successful
   ↓
New bottleneck appears
```

So now Redis itself must scale.

---

# 6. Redis Sharding

Instead of:

```text
90M requests
     ↓
 Redis #1
```

we want something conceptually like:

```text
              Redis Cluster
                   ↓
       ┌───────────┼───────────┐
       ↓           ↓           ↓
    Shard A     Shard B     Shard C
```

The data is divided among Redis shards.

For example:

```text
Shard A

product:A
product:D
user:100
```

```text
Shard B

product:B
product:E
user:200
```

```text
Shard C

product:C
product:F
user:300
```

Therefore requests can also be distributed across machines.

Redis Cluster accomplishes this using **16,384 hash slots**. A key is mapped to a slot, and each Redis primary is responsible for a subset of those slots.

Conceptually:

```text
product:A
    ↓
 hash
    ↓
slot 1200
    ↓
Redis A
```

Another key might produce:

```text
product:E
    ↓
 hash
    ↓
slot 9274
    ↓
Redis B
```

The important concept is not memorizing `16,384`.

It is understanding:

> **Redis determines which shard owns a key, allowing different keys and their traffic to be distributed across multiple Redis machines.**

---

# 7. What Happens When We Add Another Redis Machine?

Suppose originally:

```text
Redis A
├── Product A
├── Product B
├── Product C
├── Product D
├── Product E
└── Product F
```

Traffic grows, so we add Redis B.

Simply starting another Redis machine isn't enough.

Redis must redistribute ownership.

Conceptually:

```text
Before

Redis A
A B C D E F
```

After resharding:

```text
Redis A          Redis B

A                D
B                E
C                F
```

Redis Cluster actually accomplishes this by moving **hash-slot ownership** and the keys belonging to those slots.

Redis documentation explains that adding a node involves moving some slots from existing nodes to the new node; the cluster can continue serving operations while slots are moved.

AWS ElastiCache supports online horizontal scaling by adding shards and rebalancing the keyspace while the cluster continues serving requests.

---

# 8. AWS Can Manage This Redis Infrastructure

We could build and operate Redis Cluster ourselves on EC2.

For example:

```text
Private Redis Subnet

Redis EC2 #1
Redis EC2 #2
Redis EC2 #3
...
```

Then we are responsible for operating those Redis nodes.

Alternatively, AWS provides managed Redis-compatible infrastructure through ElastiCache.

At very large scale, the architecture becomes conceptually:

```text
                        Internet
                           ↓
                    Load Balancer
                           ↓
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
           App #1        App #2        App #N
             │             │             │
             └─────────────┼─────────────┘
                           ↓
                     Redis Cluster
                           ↓
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
          Shard 1       Shard 2       Shard N
                           ↓
                       Database
```

AWS documents ElastiCache for Redis as capable of scaling to **hundreds of millions of operations per second**, depending on the cluster/workload, by scaling across nodes and shards rather than relying on one enormous Redis process.

ElastiCache can horizontally scale cluster-mode-enabled deployments by adding or removing shards and rebalancing the keyspace.

So:

```text
100M operations/sec
```

does **not** mean:

```text
One magical Redis server
       ↓
100M operations/sec
```

It means distributed capacity.

---

# 9. Then We Discover the Hot-Key Problem

Now comes one of the most interesting parts.

Suppose our homepage always shows:

```text
Product A
Product B
Product C
```

And these happen to live on Redis #1.

```text
Redis #1
├── Product A
├── Product B
└── Product C
```

Redis #2 contains thousands of less popular products.

```text
Redis #2
├── Product X
├── Product Y
├── Product Z
└── ...
```

Now suppose **90% of visitors request the homepage**.

Traffic might look like:

```text
                 Homepage traffic
                       ↓
                    A B C
                       ↓
                   Redis #1
                   🔥🔥🔥🔥

Redis #2
almost idle
```

This reveals something extremely important:

> **Equal distribution of data does not guarantee equal distribution of traffic.**

Redis #1 might contain only 1 GB of data but receive millions of requests.

Redis #2 might contain 10 GB but receive very little traffic.

That is a **hot shard**.

If one individual key receives enormous traffic, it is a **hot key**.

---

# 10. Adding More Redis Shards Doesn't Automatically Fix Hot Data

Suppose:

```text
product:A
```

alone receives millions of requests.

Its hash slot belongs to Redis #1.

Adding:

```text
Redis #2
Redis #3
Redis #4
```

doesn't magically make one key exist across every shard for normal sharding.

The request is still asking for:

```text
product:A
```

and that key still has an owner.

This is where another caching layer can become useful.

---

# 11. Local Cache Comes Back

Earlier we discovered a problem with local caching:

```text
App #1 cache ≠ App #2 cache
```

But that doesn't mean local caching is useless.

For **extremely hot, mostly-read data**, local caching can be excellent.

Suppose every application EC2 temporarily stores the homepage:

```text
App #1
└── Homepage cache

App #2
└── Homepage cache

App #3
└── Homepage cache
```

Now:

```text
                     Load Balancer
                          ↓
             ┌────────────┼────────────┐
             ↓            ↓            ↓
           App #1       App #2       App #3
             ↓            ↓            ↓
         Local Cache  Local Cache  Local Cache
```

Most homepage requests don't need to reach Redis.

Only when the local cache expires:

```text
Local cache
    ↓ MISS
Redis
    ↓
refresh local cache
```

So the hierarchy becomes:

```text
Request
   ↓
Local application cache
   ↓ miss
Redis
   ↓ miss
Database
```

This might look strange initially.

We introduced Redis because local cache wasn't shared.

Now we're introducing local cache again to protect Redis.

But they solve **different problems**.

Redis provides a **shared cache across application instances**.

Local cache provides an **extremely close cache that can protect Redis from extremely hot reads**.

---

# 12. What Happens When Auto Scaling Creates a New App Instance?

Suppose:

```text
App #1 → local homepage cache
App #2 → local homepage cache
App #3 → local homepage cache
```

Traffic suddenly increases.

Auto Scaling creates:

```text
App #4
```

App #4 starts without its local cache being warm.

It can obtain the data from Redis:

```text
App #4
   ↓
Local cache MISS
   ↓
Redis
   ↓
homepage data
   ↓
App #4 stores locally
```

Now:

```text
App #1 → local homepage cache
App #2 → local homepage cache
App #3 → local homepage cache
App #4 → local homepage cache
```

We generally don't need to copy a cache file from App #1 to App #4.

Each instance can independently warm its local cache from the shared Redis layer.

---

# 13. For a Public Homepage, We Can Go Even Further

If millions of users are requesting nearly identical public homepage content, why should those requests even reach our EC2 instances?

We can introduce a CDN or edge cache.

```text
Users
   ↓
CDN / Edge Cache
   ↓ cache miss
Load Balancer
   ↓
App EC2
   ↓
Local Cache
   ↓ miss
Redis
   ↓ miss
Database
```

Now we have several defensive layers:

```text
                User
                  ↓
             CDN Cache
                  ↓
             App Cache
                  ↓
                Redis
                  ↓
              Database
```

Each layer protects the one underneath it.

---

# 14. The Complete Architecture

Putting everything together:

```text
                         INTERNET
                            │
                            ▼
                    ┌──────────────┐
                    │ CDN / Edge   │
                    │    Cache     │
                    └──────┬───────┘
                           │ cache miss
                           ▼

                  PUBLIC SUBNETS
                           │
                    Load Balancer
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼

                   PRIVATE APP SUBNETS

              HR EC2    Accounts EC2    Shop EC2
                │           │              │
          Local Cache  Local Cache    Local Cache
                │           │              │
                └───────────┼──────────────┘
                            │ cache miss
                            ▼

                   PRIVATE CACHE SUBNETS

                     Redis Cluster

             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          Shard A        Shard B        Shard C
             │              │              │
             └──────────────┼──────────────┘
                            │ cache miss
                            ▼

                    PRIVATE DB SUBNETS

                       Database
```

---

# 15. What Each Layer Is Actually Solving

Now the architecture makes sense because every component exists for a reason.

```text
Load Balancer
→ distribute incoming application traffic

Auto Scaling
→ add/remove application compute capacity

Local Cache
→ protect Redis from extremely hot repeated reads

Redis
→ shared fast cache/state across app instances

Redis Sharding
→ distribute Redis data/work across multiple Redis nodes

Redis Replication
→ availability/read scaling depending on architecture

Database
→ durable source of truth

CDN
→ stop suitable public/static/cacheable traffic before it reaches the application
```

The distinction between **sharding and replication** is particularly important.

Sharding means:

```text
Redis A → Data A
Redis B → Data B
Redis C → Data C
```

Replication means:

```text
Primary
   ↓
Replica
```

The replica contains copies of data for availability and potentially additional read capacity. AWS describes a shard as having a read/write primary and, when configured, read-only replicas.

---

# 16. The Most Important Lesson

System design isn't:

> “Use Redis because Redis is fast.”

It is asking:

```text
What is currently overloaded?
        ↓
Why is it overloaded?
        ↓
Where is the bottleneck?
        ↓
What architecture solves THAT bottleneck?
        ↓
What new problem does that solution create?
```

We started with:

```text
Database overloaded
```

So we introduced:

```text
Redis
```

Then:

```text
Application server overloaded
```

So we introduced:

```text
Load Balancer + Auto Scaling
```

Then:

```text
Local caches aren't shared
```

So Redis became even more useful.

Then:

```text
Redis overloaded
```

So we introduced:

```text
Redis Cluster + Sharding
```

Then:

```text
One extremely popular key/shard overloaded
```

So we considered:

```text
Local caching
```

And if the homepage itself is highly cacheable:

```text
CDN / edge caching
```

The final evolution looks like this:

```text
Single Server
     ↓
Database bottleneck
     ↓
Redis
     ↓
Traffic grows
     ↓
Load Balancer
     ↓
Multiple App Instances
     ↓
Shared Redis
     ↓
Redis bottleneck
     ↓
Redis Sharding
     ↓
Hot-key problem
     ↓
Local Cache
     ↓
Massive public traffic
     ↓
CDN
```


