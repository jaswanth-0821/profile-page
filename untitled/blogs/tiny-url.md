---
layout:
  width: default
  title:
    visible: false
  description:
    visible: false
  tableOfContents:
    visible: false
  outline:
    visible: true
  pagination:
    visible: false
  metadata:
    visible: false
  tags:
    visible: true
  actions:
    visible: true
---

# Tiny url

<h2 align="center">Tiny URL</h2>

## Introduction

It is like, you give a long URL to the system, and you get a short URL which can be shared with others to access the main URL. When you click the short URL, it will redirect to the respective long URL. The main advantage of tiny URL is when you want to share a URL in a chat or social media — having a very long URL is messy, so to keep the message clean we use a short URL which will redirect us to the main URL.

Example: Long url : `https://www.linkedin.com/in/jaswanth-reddy-p/`\
Short url : \`https://tinyurl.com/389zf6pb\`

## Functional and Non Functional Requirements

* Funtional Requirements
  * Core Requirements
    * System should be able to create a short URL given a long URL.
    * User can't delete the URL, but can set an expiry date to the short URL.
  * Optional Requirements
    * Allow user to create custom short URL of max length 16.
    * Give analytics for the clicks.
* Non Functional Requirements
  * avoid short URL collision
  * low latency
  * scalability
  * security
  * high availability

## Traffic and Storage Estimation

Let us assume:

* 1 million write requests per day
* 100:1 read-to-write ratio (for every 1 write request we serve 100 read requests)
* Avg length of long URL is 50 characters

Avg Writes per second : 1 million / 2&#x34;_&#x36;&#x30;_&#x36;0 = 12 urls / sec\
Avg Reads per second : writes per second \* 100 = 12 \* 100 = 1200 / sec

Our Database Contains:

| Field       | Bytes |
| ----------- | ----- |
| Long Url    | 50    |
| Short Url   | 7     |
| Expire Date | 8     |
| Click Count | 4     |

Total 50 + 7 + 8 + 4 \~= 70 bytes per row

Per year assume we will create around 365 million urls = 365,000,000 \* 70 bytes \~= 25 GB per year.

25 GB is very easy in current hardware availability and can be easily handled by a single system (given the assumptions).

## API Design

POST : /api/service/shorten

Request body:

{% code title="request.json" %}
```json
{
  "long url*": "<LONG URL>",
  "Expire date": "<EXPIRE DATE>"
}
```
{% endcode %}

Response:

{% code title="response.json" %}
```json
{
  "Short url": "<SHORT URL>"
}
```
{% endcode %}

GET : /{short url}

Return: long URL corresponding to the short url

## High Level Design

![](https://jaswanth-0821.github.io/assets/img/blogs/tiny_url/tiny_url-high-level-design-light.svg)

How it works

<img src="https://jaswanth-0821.github.io/assets/img/blogs/tiny_url/tiny_url-how-it-works-light.svg" alt="" width="375">

Why 301 redirect? Can't we use 302?

* 301 Redirect: A 301 redirect shows that the requested URL is “permanently” moved to the long URL. Since it is permanently redirected, the browser caches the response, and subsequent requests for the same URL will not be sent to the URL shortening service. Instead, requests are redirected to the long URL server directly.
* 302 Redirect: A 302 redirect means that the URL is “temporarily” moved to the long URL, meaning that subsequent requests for the same URL will be sent to the URL shortening service first, then redirected to the long URL server.

## Deep Dive

Now we are able to shorten the long URL and serve the client. But how are we handling the uniqueness of short URLs? Is this system fast, scalable, and resilient against malicious attacks?

First, how to create unique URLs:

<details>

<summary><mark style="color:red;">Bad way</mark></summary>

Generating a random 7-digit number and encoding it to base62. Initially this will handle the task with simple check-and-assign logic, but as data increases, the collision rate increases which will cause high latency on writes.

</details>

<details>

<summary><mark style="color:$danger;">Good way</mark></summary>

Have a global counter and increase it every time you create a new URL. Use the current counter number as id and encode it to base62, incrementing the count by 1. This will help to avoid collisions.

</details>

<details>

<summary><mark style="color:$success;">Best way</mark></summary>

Using a single counter blocks scaling; multiple counters cause URL collisions.&#x20;

</details>



One approach is to have a global counter stored in Redis which returns a unique\_id. Problems:

* Lots of requests to Redis increases load and creates a SPOF.
* Concurrent requests might receive the same value causing collisions.

To tackle these, assign id batches to each write server. Example: the counter is at 1,000,000 and we have 5 write servers; assign 1000 ids to each server:

* 1,000,000–1,001,000
* 1,001,000–1,002,000
* etc.

This reduces global coordination and avoids collisions. Optionally use a queue to be extra safe.

If Redis is down, we may lose the counter value; in that case use a replica or take the highest number across mini-batches saved on servers.

Another approach is using a distributed coordination service like Zookeeper to solve race conditions, deadlocks, and partial failures. Zookeeper can manage naming, active/failed servers, configuration (which server manages which range of counters), and maintain synchronization across servers.

How to maintain a counter for distributed hosts using Zookeeper:

* From 3500 Billion URL combinations take the 1st billion combinations.
* In Zookeeper maintain the range and divide the 1st billion into 100 ranges of 10 million each. Example:
  * range 1 -> (1–10,000,000)
  * range 2 -> (10,000,001–20,000,000)
  * ...
  * range 100 -> (990,000,001–1,000,000,000) (Optionally add an offset like 100000000000 to each range for counters.)
* When servers are added they ask Zookeeper for an unused range. Suppose W1 is assigned range 1; W1 generates tiny URLs incrementing its counter and encoding. Each number is unique so there's no collision and no need to check the DB for uniqueness before insertion.
* If a server goes down then only that range of data is affected. We can replicate data from master to slave and divert reads to slaves while restoring the master.
* If a database reaches its maximum range, remove it from active write pool, add a new database with a fresh range, and register it with Zookeeper. That database can be used for reads while being prepared for writes.
* When the 1st billion is exhausted, take the 2nd billion and continue the process.

### Which database is best: SQL or NoSQL?

It does not really matter in strict terms, but given the system characteristics:

* The database is simple (no complex joins).
* We need strong/fast reads and horizontal scaling.

NoSQL (key-value stores like DynamoDB) give O(1) reads and easier scaling compared to SQL (B-tree searches \~ O(log n)). DynamoDB provides TTL which can auto-delete expired URLs (no background jobs needed). However, using TTL will lose short URL ids for reuse; if you need to reuse ids, implement manual TTL and track deleted short URL ids.

If reusing deleted IDs is important, save deleted/expired keys into a queue/DB and check there for available ids before asking the global counter for a new id.

## Cache

We can cache frequently accessed URLs using off-the-shelf solutions like Memcached or Redis. Application servers check the cache before hitting backend storage.

Example sizing:

* 1200 req/sec -> 1200 \* 24 \* 60 \* 60 \* 70 bytes/day = \~7 GB/day
* 20% of that (\~1.5 GB) can be cached initially; modern servers with 256 GB RAM can easily fit this.

Deciding which URLs to keep and eviction policy:

* Use Least Recently Used (LRU) to evict the least recently used URLs first.
* Implementation: use a LinkedHashMap (combination of hashmap + doubly linked list) to get O(1) search, insert, and delete. On cache hit, move entry to front; on cache miss, fetch from DB and insert at front; if full, remove from tail.

System design:

![](https://jaswanth-0821.github.io/assets/img/blogs/tiny_url/tiny_url-deepdive-design-1-light.svg)

Number of servers for read > number of servers for write (given high read-to-write ratio). Keeping separate read/write servers allows independent scaling.

Handling DoS: A potential security problem is malicious users sending massive numbers of URL shortening requests. Rate limiting (by IP or other rules) helps filter requests.

![](https://jaswanth-0821.github.io/assets/img/blogs/tiny_url/tiny_url-deepdive-design-2-light.svg)

