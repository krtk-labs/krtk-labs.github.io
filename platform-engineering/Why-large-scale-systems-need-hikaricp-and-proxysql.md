# Why Large-Scale Systems Use Both HikariCP and ProxySQL

When engineers first encounter **HikariCP** and **ProxySQL**, they often assume they solve the same problem:

> *"Both are connection pools. Why do I need both?"*

The answer is **they manage different connections, at different layers, for different purposes.**

Understanding this distinction is essential when designing high-scale systems like Shopify, Uber, Amazon, or IRCTC.

---

# The Short Answer

HikariCP manages **connections between your application and ProxySQL (or MySQL).**

ProxySQL manages **connections between itself and the MySQL servers.**

They are two completely different connection pools.

```
Application Threads
        │
        │
   HikariCP Pool
        │
        │ Frontend Connections
        ▼
    ProxySQL Cluster
        │
        │ Backend Connections
        ▼
     MySQL Cluster
```

One optimizes the **application**.

The other optimizes the **database**.

---

# Two Different Types of Connections

Many developers think there is only one database connection.

There are actually **two separate network connections**.

## 1. Frontend Connection

This exists between

```
Application
     │
     ▼
 ProxySQL
```

This is what HikariCP manages.

---

## 2. Backend Connection

This exists between

```
ProxySQL
     │
     ▼
 MySQL
```

This is what ProxySQL manages.

---

These are completely independent.

A request may borrow one frontend connection while ProxySQL reuses a completely different backend connection.

---

# What Problem Does HikariCP Solve?

HikariCP lives **inside your Java application**.

Suppose your checkout service receives 2,000 requests per second.

Without a connection pool:

```
Thread 1

↓

Create TCP connection

↓

Authenticate

↓

Execute SQL

↓

Disconnect
```

Every request pays the cost of

- TCP handshake
- MySQL authentication
- SSL negotiation
- Session initialization

This is expensive.

Instead HikariCP maintains a pool.

```
20 Ready Connections

Thread

↓

Borrow Connection

↓

Execute SQL

↓

Return Connection
```

No new network connection is created.

This dramatically reduces latency.

---

# What Connection Does HikariCP Create?

The connection looks like

```
Checkout Service

↓

TCP Connection

↓

ProxySQL
```

Notice:

The application **never talks directly to MySQL.**

From Hikari's perspective,

ProxySQL simply behaves like the database.

---

# What Problem Does ProxySQL Solve?

Imagine a company with

- Checkout Service
- Inventory Service
- Payment Service
- Notification Service
- Recommendation Service

Each JVM has

```
20 Hikari Connections
```

Suppose there are

```
100 JVMs
```

Total frontend connections

```
100 × 20

=

2000 connections
```

Without ProxySQL

```
2000 connections

↓

MySQL
```

MySQL must manage all of them.

This consumes

- memory
- CPU
- scheduler time
- thread resources

---

Instead

```
2000 Frontend Connections

↓

ProxySQL

↓

500 Backend Connections

↓

MySQL
```

ProxySQL multiplexes many frontend sessions onto a much smaller backend pool whenever it is safe to do so.

The application believes it owns 2,000 database connections.

MySQL only manages 500.

---

# Why Can't HikariCP Do This?

Because HikariCP only knows about **its own JVM**.

```
Checkout Service

20 Connections
```

It has no knowledge of

```
Inventory Service

Payment Service

Recommendation Service
```

Each Hikari pool is isolated.

There is no coordination.

---

# Why Can't ProxySQL Replace HikariCP?

Without Hikari

Every application request would do

```
Create Client Connection

↓

ProxySQL

↓

Execute SQL

↓

Disconnect
```

The application would repeatedly create TCP connections.

That overhead still exists.

ProxySQL does **not** eliminate the need for efficient client-side pooling.

---

# Responsibilities Compared

| HikariCP | ProxySQL |
|------------|------------|
| Lives inside JVM | Separate proxy server |
| Pools client connections | Pools backend database connections |
| Optimizes application latency | Optimizes database scalability |
| Reduces connection creation cost | Reduces MySQL connection count |
| One pool per JVM | Shared across all applications |

---

# Architecture

```
                     Users
                       │
                       ▼
                Load Balancer
                       │
        ┌──────────────┴──────────────┐
        │                             │
        ▼                             ▼
 Checkout Service              Inventory Service
        │                             │
    HikariCP                     HikariCP
        │                             │
        └──────────────┬──────────────┘
                       │
               Frontend Connections
                       │
                       ▼
                ProxySQL Cluster
                       │
              Backend Connections
                       │
        ┌──────────────┴──────────────┐
        ▼                             ▼
    MySQL Primary              MySQL Replica
```

---

# What Additional Value Does ProxySQL Provide?

Because **every SQL statement passes through it**, ProxySQL becomes a central control plane.

It can provide capabilities that HikariCP never can.

## Read/Write Routing

```
SELECT

↓

Replica
```

```
INSERT

↓

Primary
```

Applications don't need to know which database to use.

---

## Automatic Failover

If the primary database fails,

ProxySQL redirects traffic to another node.

Applications continue using the same endpoint.

---

## Query Rules

It can rewrite queries

```
Old SQL

↓

New SQL
```

without changing application code.

---

## Rate Limiting

Certain expensive queries can be throttled before reaching MySQL.

---

## Centralized Metrics

Since every service talks through ProxySQL,

it has visibility into

- all queries
- all transactions
- all connections

across the organization.

---

# Shopify's Use Case

Shopify wasn't using ProxySQL merely for pooling.

They used it as an **observability platform**.

Applications annotated SQL statements with comments such as

```sql
/* conn_tag:checkout */
```

ProxySQL inspected these comments before forwarding the query.

It associated the tag with the backend connection and measured

- when the connection was borrowed
- when it was returned
- how long it remained occupied

This allowed Shopify to answer questions such as

- Which business process holds database connections the longest?
- Which workflow consumes the largest share of database capacity?
- Where should optimization efforts be focused?

Traditional database monitoring reports query latency.

ProxySQL enabled them to measure **connection occupancy**, which was the real bottleneck.

---

# HikariCP + ProxySQL Together

Think of the architecture as two optimization layers.

```
Application Threads

↓

HikariCP

↓

Persistent Frontend Connections

↓

ProxySQL

↓

Shared Backend Connection Pool

↓

MySQL
```

Each layer removes a different bottleneck.

HikariCP eliminates expensive connection creation inside each JVM.

ProxySQL prevents thousands of JVMs from overwhelming the database.

---

# Airport Analogy

Imagine an airport.

Passengers

↓

Check-in Counter

↓

Air Traffic Control

↓

Aircraft

HikariCP is the **check-in counter**.

It efficiently manages passengers waiting to board.

ProxySQL is **air traffic control**.

It decides which aircraft to use, balances traffic across runways, reroutes flights during disruptions, and optimizes airport-wide capacity.

Both are essential.

Removing either creates a bottleneck.

---

# When Should You Use Both?

For a small application with a single service talking directly to MySQL,

HikariCP is usually sufficient.

As systems grow into dozens or hundreds of services, ProxySQL becomes valuable for

- reducing database connection pressure
- centralized query routing
- read/write splitting
- failover
- traffic shaping
- organization-wide observability
- cross-service connection management

Large-scale architectures therefore use **both**.

One optimizes the application.

The other optimizes the database ecosystem.

Together they provide a scalable, resilient, and observable database access layer.
