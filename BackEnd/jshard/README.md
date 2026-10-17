# jshard

*J2OS Platform 2.4*

A table that keeps growing eventually outgrows one database server: it no longer fits on the disk, the queries slow down, or one machine cannot take the load. The usual answer is *sharding*: the rows are split over several databases, called *shards*, and each row lives in exactly one of them. The application still wants to write `INSERT` and `SELECT` as if there were a single table, so something has to decide which database each statement goes to.

**jshard** (`org.j2os.platform.jshard`) does that. It is built on *Apache ShardingSphere* and the *HikariCP* connection pool, and it turns a list of databases and a list of tables into one ordinary `javax.sql.DataSource`. Your code keeps using JDBC, `JdbcTemplate` or JPA. jshard looks at the key in each statement and sends the statement to the right database.

```java
JShardDataSource dataSource = JShardDataSourceProvider.builder()
        .shard("shard-a", databaseA)
        .shard("shard-b", databaseB)
        .table("customer", "customer_id")
        .build();

// from here on it is an ordinary javax.sql.DataSource
```

The module has seven public types:

- `JShardConnectionConfig` describes how to connect to one database.
- `JShardTableConfig` names a table to split and the column that decides where a row goes.
- `JShardDataSourceProvider` collects the shards and tables and builds the data source.
- `JShardDataSource` is the finished `DataSource`. It is also `Closeable`.
- `JShardRouter` answers "which shard owns this key?" without touching any database.
- `JShardHashAlgorithm` is the rule that maps a key to a shard. ShardingSphere calls it for every statement, and you rarely call it yourself.
- `JShardDataSourceRegistry` owns the connection pools. The builder uses it, and you rarely need it directly.

This guide goes step by step: each chapter has a short program and the output it prints. The appendices are for quick reference.

## Contents

1. [How rows find their shard](#1-how-rows-find-their-shard)
2. [Describing a database](#2-describing-a-database)
3. [Describing a table](#3-describing-a-table)
4. [Where a key goes](#4-where-a-key-goes)
5. [The hash ring](#5-the-hash-ring)
6. [When the set of shards changes](#6-when-the-set-of-shards-changes)
7. [Building the data source](#7-building-the-data-source)
8. [Using the data source](#8-using-the-data-source)
9. [Replicas](#9-replicas)
10. [Checking the databases, and closing](#10-checking-the-databases-and-closing)
11. [In a Spring application](#11-in-a-spring-application)
12. [Putting it together: a customer directory that grows](#12-putting-it-together-a-customer-directory-that-grows)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The module needs ShardingSphere, HikariCP, Guava and SLF4J on the classpath, plus the JDBC driver of your database. The programs in chapters 2 to 6 and chapter 12 use only the configuration classes and the router, so they run without any database.

---

## 1. How rows find their shard

Every sharded table has a *sharding column*, for example `customer_id`. When a row is written, the value of that column is turned into a number by a *hash function*, a fixed recipe that always gives the same number for the same text and spreads different texts evenly. The number decides the shard. When the same value is read later, the same recipe leads to the same shard, so the row is found without asking the other databases.

Three consequences shape the rest of this guide.

**The key decides everything.** A statement that names the key, such as `WHERE customer_id = ?`, goes to one shard. A statement that does not, such as a count of all rows, has to ask every shard (chapter 8).

**The key is text.** The hash works on `String` values, so the sharding column should be a text column, for example `VARCHAR`. An identifier that is a number can be stored as text.

**The set of shards matters.** The shard for a key depends on the names of all the shards. If the set changes, some keys change shard, and the rows that are already stored do not move by themselves (chapter 6).

The tables must already exist, with the same name, in every database. jshard does not create or change tables.

## 2. Describing a database

A `JShardConnectionConfig` holds what is needed to connect to one database. It is immutable, so it can be shared freely.

```java
import org.j2os.platform.jshard.config.JShardConnectionConfig;

public class ConnectionConfig {

    public static void main(String[] args) {
        JShardConnectionConfig simple = JShardConnectionConfig.of(
                "org.postgresql.Driver", "jdbc:postgresql://db1.example.com:5432/shop?password=secret",
                "admin", "s3cret", 10);
        System.out.println(simple);

        JShardConnectionConfig tuned = JShardConnectionConfig.builder()
                .driverClassName("org.postgresql.Driver")
                .jdbcUrl("jdbc:postgresql://db2.example.com:5432/shop")
                .username("app")
                .password("s3cret")
                .poolSize(10)
                .connectionTimeoutMs(5_000)
                .maxLifetimeMs(1_800_000)
                .build();
        System.out.println(tuned);
        System.out.println("connection timeout: " + tuned.getConnectionTimeoutMs());
        System.out.println("idle timeout:       " + tuned.getIdleTimeoutMs());

        try {
            JShardConnectionConfig.of("org.postgresql.Driver", "jdbc:postgresql://db3/shop", "app", "s3cret", 0);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
        try {
            JShardConnectionConfig.of("org.postgresql.Driver", " ", "app", "s3cret", 5);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
ShardConfig{driver='org.postgresql.Driver', host=jdbc:postgresql://db1.example.com:5432/shop, user=a***n, poolSize=10}
ShardConfig{driver='org.postgresql.Driver', host=jdbc:postgresql://db2.example.com:5432/shop, user=a***p, poolSize=10}
connection timeout: 5000
idle timeout:       null
poolSize must be greater than zero
ShardConfig.jdbcUrl must not be null/blank
```

- `JShardConnectionConfig.of(driver, url, user, password, poolSize)` covers the five required values. The `poolSize` is the largest number of connections that may be open to this database at the same time.
- The builder adds optional pool settings. A setting that is not set stays `null` and the pool uses its own default, as the last two getters show.
- `toString()` is safe to write to a log. It never prints the password, it keeps only the first and last letter of the user name, and it cuts everything after `?` from the URL, because a URL sometimes carries a secret as a parameter. The `?password=secret` in the first URL does not appear in the output.
- A missing or blank driver, URL, user name or password, and a pool size of zero or less, are rejected at once with `IllegalArgumentException`. The message names the field.

The optional settings are those of HikariCP, in milliseconds unless stated otherwise:

| Builder method | Meaning |
|----------------|---------|
| `connectionTimeoutMs` | how long a caller waits for a free connection before it gets an error |
| `idleTimeoutMs` | how long an unused connection stays open before it is closed |
| `maxLifetimeMs` | the age at which a connection is closed and replaced |
| `minimumIdle` | how many unused connections the pool keeps ready (a count) |
| `validationTimeoutMs` | how long to wait while a connection is checked for health |
| `keepaliveTimeMs` | how often an unused connection is tested to keep it alive |

## 3. Describing a table

A `JShardTableConfig` holds two names: the table, and the column whose value decides the shard.

```java
import org.j2os.platform.jshard.config.JShardTableConfig;

public class TableConfig {

    public static void main(String[] args) {
        JShardTableConfig customer = new JShardTableConfig("customer", "customer_id");
        System.out.println(customer);
        System.out.println(customer.name() + " is split by " + customer.shardingColumn());

        String[][] bad = {{"shop.customer", "id"}, {"customer", "customer id"}, {"customer", ""}, {" customer", "id"}};
        for (String[] pair : bad) {
            try {
                new JShardTableConfig(pair[0], pair[1]);
            } catch (IllegalArgumentException e) {
                System.out.println(e.getMessage());
            }
        }
    }
}
```

```text
ShardingTableConfig{name='customer', shardingColumn='customer_id'}
customer is split by customer_id
ShardingTableConfig.name must not contain '.', ',', or a quote character: 'shop.customer'
ShardingTableConfig.shardingColumn must not contain whitespace or control characters: 'customer id'
ShardingTableConfig.shardingColumn must not be null/blank
ShardingTableConfig.name must not have leading/trailing whitespace: ' customer'
```

- `name()` and `shardingColumn()` return the two names. The class is a Java *record*, a small immutable class with `equals` and `hashCode` built in.
- Both names are checked when the object is created. They must not be blank, must not start or end with a space, and must not contain a dot, a comma, a quote, a space or a control character. These characters would break the expression that ShardingSphere builds from the names, so jshard refuses them early.

Because the table name is also the name of the table in every database, a schema prefix such as `shop.customer` is not accepted. Use the plain table name.

## 4. Where a key goes

`JShardRouter` predicts the shard of a key. It needs no database: only the names of the shards, the number of virtual nodes (chapter 5) and a seed number.

```java
import org.j2os.platform.jshard.router.JShardRouter;

import java.util.Set;

public class WhereIsIt {

    public static void main(String[] args) {
        JShardRouter router = new JShardRouter(Set.of("shard-a", "shard-b", "shard-c"), 160, 0);

        for (String id : new String[]{"c-1001", "c-1002", "c-1003", "c-1004", "c-1001"}) {
            System.out.println(id + " -> " + router.getShardKey(id));
        }

        JShardRouter sameNames = new JShardRouter(Set.of("shard-c", "shard-a", "shard-b"), 160, 0);
        System.out.println("another router: c-1001 -> " + sameNames.getShardKey("c-1001"));

        try {
            router.getShardKey(" ");
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
        try {
            new JShardRouter(Set.of(), 160, 0);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
c-1001 -> shard-a
c-1002 -> shard-b
c-1003 -> shard-b
c-1004 -> shard-c
c-1001 -> shard-a
another router: c-1001 -> shard-a
The sharding column value for table 'n/a' must not be null/empty
shardNames must not be null/empty
```

- `getShardKey(key)` returns the name of the shard that owns the key.
- The same key always gives the same shard. A second router built with the same names gives the same answer, even if the names are listed in a different order.
- The rule uses the shard **names** and the two numbers, nothing else. The host, the user and the password of the databases play no part.
- A blank key is rejected. The message says `table 'n/a'` because no table is involved when the router is used on its own.
- A router needs at least one shard name.

The router is the tool for questions that come before the database: where will this customer be stored, which database should I look at when I debug a problem, how many rows would move if I add a shard. The data source you build in chapter 7 owns a router of its own, available with `getRouter()`, and it gives the same answers as the real statements.

## 5. The hash ring

jshard places the shards on a *hash ring*, an imaginary circle of numbers. Each shard gets many points on the circle, called *virtual nodes*. A key is hashed to a number, and the first virtual node at or after that number, going around the circle, names the shard.

How many virtual nodes each shard gets decides how evenly the keys are shared.

```java
import org.j2os.platform.jshard.router.JShardRouter;

import java.util.Set;
import java.util.TreeMap;

public class RingBalance {

    public static void main(String[] args) {
        Set<String> shards = Set.of("shard-a", "shard-b", "shard-c");

        for (int virtualNodes : new int[]{1, 10, 160, 2000}) {
            JShardRouter router = new JShardRouter(shards, virtualNodes, 0);
            TreeMap<String, Integer> count = new TreeMap<>();
            for (int i = 0; i < 30_000; i++) {
                count.merge(router.getShardKey("c-" + i), 1, Integer::sum);
            }
            System.out.println(virtualNodes + " per shard: " + count);
        }
    }
}
```

```text
1 per shard: {shard-a=2717, shard-b=25732, shard-c=1551}
10 per shard: {shard-a=3633, shard-b=13250, shard-c=13117}
160 per shard: {shard-a=11305, shard-b=8768, shard-c=9927}
2000 per shard: {shard-a=10518, shard-b=9654, shard-c=9828}
```

- With one virtual node per shard the split is very uneven: `shard-b` receives most of the keys.
- With 160, the default, the shares are close but not equal.
- With 2000 the shares are within about five percent of each other.

A larger number gives a more even split but costs a little memory and time when the ring is built, in proportion to the number of shards times the number of virtual nodes. The default of 160 suits most cases. The largest accepted value is 100000.

The ring is built from the shard names, the virtual-node count and the seed, and it is kept in a cache that is shared by the whole JVM. Every router and every data source with the same three values uses the same ring, so they always agree.

`JShardHashAlgorithm` is the class behind all this. ShardingSphere creates it and calls it for each statement. It can also be used directly:

```java
import org.j2os.platform.jshard.algorithm.JShardHashAlgorithm;

import java.util.List;
import java.util.Properties;

public class AlgorithmDirect {

    public static void main(String[] args) {
        JShardHashAlgorithm algorithm = new JShardHashAlgorithm();
        System.out.println("type: " + algorithm.getType());
        System.out.println("defaults: " + JShardHashAlgorithm.DEFAULT_VIRTUAL_NODES_PER_SHARD
                + " virtual nodes, seed " + JShardHashAlgorithm.DEFAULT_HASH_SEED);

        Properties props = new Properties();
        props.setProperty("sharding-count", "3");
        algorithm.init(props);

        System.out.println(algorithm.predictShard(List.of("a", "b", "c"), "c-1001"));
        try {
            algorithm.predictShard(List.of("a", "b"), "c-1001");
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage());
        }
        try {
            algorithm.predictShard(List.of(), "c-1001");
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage());
        }

        props.setProperty("virtual-nodes-per-shard", "0");
        try {
            algorithm.init(props);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
type: JSHARD_CONSISTENT_HASH
defaults: 160 virtual nodes, seed 0
a
The number of available shards (2) does not match the configured sharding-count (3)
No shards are available for table 'n/a'
virtual-nodes-per-shard must be positive
```

- `getType()` is the name under which the algorithm is known to ShardingSphere.
- `predictShard(names, key)` is what the router calls.
- When `sharding-count` is set, the algorithm checks that the number of shards matches it. The data source sets it for you.
- A property that is out of range is rejected when the algorithm is initialised.

## 6. When the set of shards changes

Sooner or later a fourth database is needed. The ring is designed for this: adding a shard takes keys from the others, but it does not reshuffle everything.

```java
import org.j2os.platform.jshard.router.JShardRouter;

import java.util.Set;

public class AddShard {

    public static void main(String[] args) {
        JShardRouter before = new JShardRouter(Set.of("shard-a", "shard-b", "shard-c"), 160, 0);
        JShardRouter after = new JShardRouter(Set.of("shard-a", "shard-b", "shard-c", "shard-d"), 160, 0);

        int total = 30_000;
        int moved = 0;
        int movedToOldShard = 0;
        int movedByModulo = 0;
        for (int i = 0; i < total; i++) {
            String id = "c-" + i;
            if (!before.getShardKey(id).equals(after.getShardKey(id))) {
                moved++;
                if (!after.getShardKey(id).equals("shard-d")) {
                    movedToOldShard++;
                }
            }
            if (Math.floorMod(id.hashCode(), 3) != Math.floorMod(id.hashCode(), 4)) {
                movedByModulo++;
            }
        }
        System.out.println("keys that change shard when a fourth shard is added:");
        System.out.println("  hash ring:      " + moved + " of " + total);
        System.out.println("  hash % shards:  " + movedByModulo + " of " + total);
        System.out.println("moved keys that went to an old shard: " + movedToOldShard);
    }
}
```

```text
keys that change shard when a fourth shard is added:
  hash ring:      6740 of 30000
  hash % shards:  22499 of 30000
moved keys that went to an old shard: 0
```

- On the ring, a little over a fifth of the keys move when the fourth shard is added, which is roughly the share that the new shard should own, and every key that moves goes to the new shard. No key moves between the old shards.
- With the simple recipe `hash % number of shards`, three out of four keys change shard.

Two other changes are far more destructive:

```java
import org.j2os.platform.jshard.router.JShardRouter;

import java.util.Set;

public class SeedAndNames {

    public static void main(String[] args) {
        JShardRouter base = new JShardRouter(Set.of("shard-a", "shard-b", "shard-c"), 160, 0);
        JShardRouter otherSeed = new JShardRouter(Set.of("shard-a", "shard-b", "shard-c"), 160, 7);
        JShardRouter renamed = new JShardRouter(Set.of("shard-a", "shard-b", "shard-x"), 160, 0);

        int seedChanged = 0;
        int renameChanged = 0;
        for (int i = 0; i < 10_000; i++) {
            String id = "c-" + i;
            if (!base.getShardKey(id).equals(otherSeed.getShardKey(id))) {
                seedChanged++;
            }
            if (!base.getShardKey(id).equals(renamed.getShardKey(id))) {
                renameChanged++;
            }
        }
        System.out.println("changing the seed moves:       " + seedChanged + " of 10000 keys");
        System.out.println("renaming shard-c to shard-x moves: " + renameChanged + " of 10000 keys");
    }
}
```

```text
changing the seed moves:       6659 of 10000 keys
renaming shard-c to shard-x moves: 5071 of 10000 keys
```

- A different seed places everything again from scratch, so most keys move.
- Renaming a shard counts as removing one shard and adding another. About half of the keys move.

The lesson is the same in each case: **the shard names and the seed are part of your data's address.** Choose them once and do not change them. jshard checks only that they are valid, so a changed name is accepted silently and rows are looked up in the wrong place.

jshard also does not move rows. After you add a shard, the keys that now belong to the new shard are still stored in the old ones, and a lookup by key goes to the new, empty shard and finds nothing. Copy those rows first (chapter 12 shows how to list them), and only then switch the application to the new configuration.

## 7. Building the data source

The `JShardDataSourceProvider.builder()` collects the description and `build()` creates the data source.

```java
import org.j2os.platform.jshard.config.JShardConnectionConfig;
import org.j2os.platform.jshard.datasource.JShardDataSource;
import org.j2os.platform.jshard.datasource.JShardDataSourceProvider;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

public class ShopData {

    public static void main(String[] args) throws Exception {
        String password = System.getenv("DB_PASSWORD");

        JShardConnectionConfig dbA = JShardConnectionConfig.of(
                "org.postgresql.Driver", "jdbc:postgresql://db-a.example.com:5432/shop", "app", password, 10);
        JShardConnectionConfig dbB = JShardConnectionConfig.of(
                "org.postgresql.Driver", "jdbc:postgresql://db-b.example.com:5432/shop", "app", password, 10);
        JShardConnectionConfig dbC = JShardConnectionConfig.of(
                "org.postgresql.Driver", "jdbc:postgresql://db-c.example.com:5432/shop", "app", password, 10);

        try (JShardDataSource dataSource = JShardDataSourceProvider.builder()
                .shard("shard-a", dbA)
                .shard("shard-b", dbB)
                .shard("shard-c", dbC)
                .table("customer", "customer_id")
                .table("customer_order", "customer_id")
                .build();
             Connection connection = dataSource.getConnection()) {

            try (PreparedStatement insert = connection.prepareStatement(
                    "INSERT INTO customer (customer_id, name) VALUES (?, ?)")) {
                insert.setString(1, "c-1001");
                insert.setString(2, "Sara");
                insert.executeUpdate();                  // stored on the shard that owns "c-1001"
            }

            try (PreparedStatement select = connection.prepareStatement(
                    "SELECT name FROM customer WHERE customer_id = ?")) {
                select.setString(1, "c-1001");           // only that shard is asked
                try (ResultSet row = select.executeQuery()) {
                    row.next();
                    System.out.println(row.getString("name"));
                }
            }

            try (PreparedStatement count = connection.prepareStatement(
                    "SELECT COUNT(*) FROM customer")) {  // no customer_id: every shard is asked
                try (ResultSet row = count.executeQuery()) {
                    row.next();
                    System.out.println(row.getLong(1));
                }
            }
        }
    }
}
```

- `shard(name, primary, replicas...)` adds a database under a shard name. Chapter 9 explains the replicas.
- `table(name, shardingColumn)` adds a sharded table. A second table can use the same sharding column. Here `customer_order` is split by `customer_id` too, so the orders of a customer are stored on the same shard as the customer.
- `build()` creates a connection pool for each database and assembles the ShardingSphere rules. It declares `throws Exception` because ShardingSphere may fail while it builds. If anything fails after some pools have started, they are all closed before the failure is thrown.
- The result is closed by `try-with-resources`, which ends all the pools.

Other builder methods:

| Method | Meaning |
|--------|---------|
| `shards(map)` | adds several shards at once; the first element of each list is the primary |
| `tables(list)` | adds several `JShardTableConfig` objects at once |
| `virtualNodesPerShard(n)` | the ring density of chapter 5; the default is 160 |
| `hashSeed(n)` | the seed of chapter 6; the default is 0 |
| `showSql(true)` | makes ShardingSphere log the logical SQL and the SQL sent to each shard |

`showSql(true)` is the best tool for learning where statements go. Turn it off in production, where it makes a lot of noise.

The builder checks the description and explains each mistake:

```java
import org.j2os.platform.jshard.config.JShardConnectionConfig;
import org.j2os.platform.jshard.datasource.JShardDataSourceProvider;

import java.util.concurrent.Callable;

public class BuilderRules {

    public static void main(String[] args) {
        JShardConnectionConfig db = JShardConnectionConfig.of(
                "org.postgresql.Driver", "jdbc:postgresql://localhost/shop", "app", "secret", 5);

        Callable<?>[] attempts = {
                () -> JShardDataSourceProvider.builder().table("customer", "customer_id").build(),
                () -> JShardDataSourceProvider.builder().shard("shard-a", db).build(),
                () -> JShardDataSourceProvider.builder().shard("shard.a", db).table("customer", "customer_id").build(),
                () -> JShardDataSourceProvider.builder().shard("shard-a", db)
                        .table("customer", "customer_id").table("customer", "customer_id").build(),
                () -> JShardDataSourceProvider.builder().shard("shard-a", db)
                        .table("customer", "customer_id").virtualNodesPerShard(0).build(),
                () -> JShardDataSourceProvider.builder().shard("shard-a", db)
                        .table("customer", "customer_id").virtualNodesPerShard(200_000).build(),
                () -> JShardDataSourceProvider.builder().shard("shard-a", db).shard("shard-a", db),
        };
        for (Callable<?> attempt : attempts) {
            try {
                attempt.call();
            } catch (Exception e) {
                System.out.println(e.getMessage());
            }
        }
    }
}
```

```text
At least one shard must be added via .shard(...)
At least one table must be added via .table(...)
A shard name must not contain '.', ',', or a quote character (ShardingSphere syntax restriction): 'shard.a'
Duplicate table name in the tables list: 'customer'
virtualNodesPerShard must be greater than zero
virtualNodesPerShard (200000) exceeds the allowed maximum (100000)
Shard 'shard-a' has already been added
```

- At least one shard and one table are required.
- Shard names follow the same rules as table names in chapter 3. A name with a dot is refused.
- The same table twice is refused, and so is the same shard name twice. The shard name is checked when `shard(...)` is called, the other rules when `build()` is called.
- The number of virtual nodes must be between 1 and 100000.

## 8. Using the data source

`JShardDataSource` is a `javax.sql.DataSource`. Every JDBC API, `JdbcTemplate` or JPA that accepts a `DataSource` works with it.

ShardingSphere reads each statement and decides where to send it:

```java
// the key is in the condition: one shard
"SELECT * FROM customer WHERE customer_id = ?"
"UPDATE customer SET name = ? WHERE customer_id = ?"
"DELETE FROM customer WHERE customer_id = ?"

// no key, or a range of keys: every shard
"SELECT * FROM customer WHERE name = ?"
"SELECT * FROM customer WHERE customer_id > ?"
"SELECT COUNT(*) FROM customer"
```

- A statement with `customer_id = ?` in its condition goes to the one shard that owns the value. Inserts do the same: the value of `customer_id` in the row decides.
- A statement without the sharding column, or with a range such as `customer_id > ?`, goes to every shard. A range cannot be mapped to shards by a hash, so jshard answers with all of them to make sure that no row is missed. The results are merged, so a `COUNT(*)` gives the total over all shards.

So a lookup by key costs one database, and a search by any other column costs all of them. Design the tables so that the common queries carry the key.

The sharding column value must not be `null` or blank when a statement needs it for routing. The statement then fails with an error that carries the message `The sharding column value for table '<table>' must not be null/empty`.

Some things deserve care:

- **The key does not change.** Treat the sharding column of a row as fixed. A different value would belong on a different shard.
- **Transactions are per database.** jshard does not set up distributed transactions. A change that touches several shards is not guaranteed to be all-or-nothing.
- **Joins.** The module does not declare that two tables are related. Test any statement that joins sharded tables with `showSql(true)` and check where it goes.

## 9. Replicas

A shard can have copies, called *replicas*. They are databases that hold the same data as the main one, the *primary*, and are kept up to date by the database's own replication. Reads can be spread over the replicas and the primary is left for writes.

```java
JShardConnectionConfig primary  = JShardConnectionConfig.of(driver, "jdbc:postgresql://a-primary/shop", "app", password, 10);
JShardConnectionConfig replica1 = JShardConnectionConfig.of(driver, "jdbc:postgresql://a-replica1/shop", "app", password, 10);
JShardConnectionConfig replica2 = JShardConnectionConfig.of(driver, "jdbc:postgresql://a-replica2/shop", "app", password, 10);

JShardDataSource dataSource = JShardDataSourceProvider.builder()
        .shard("shard-a", primary, replica1, replica2)      // the first is the primary
        .shard("shard-b", primaryB)                          // no replicas
        .table("customer", "customer_id")
        .build();
```

- The first configuration of a shard is the primary and the others are replicas.
- ShardingSphere's read/write splitting sends writes to the primary and reads to the replicas.
- A shard without replicas simply uses its primary for everything.

jshard does not set up replication. The databases must already replicate to each other. A replica can be a little behind the primary, so a read straight after a write can miss the new row. Code that must see its own write should be arranged accordingly.

## 10. Checking the databases, and closing

A wrong password or a database that is down normally shows up in the middle of a query. `assertAllReachable` checks all of them before you build, and names the one that failed.

```java
Map<String, List<JShardConnectionConfig>> shards = new LinkedHashMap<>();
shards.put("shard-a", List.of(primaryA, replicaA1));        // first element: primary
shards.put("shard-b", List.of(primaryB));

List<JShardTableConfig> tables = List.of(
        new JShardTableConfig("customer", "customer_id"),
        new JShardTableConfig("customer_order", "customer_id"));

JShardDataSourceProvider.assertAllReachable(shards);         // fails fast, before the build

JShardDataSource dataSource = JShardDataSourceProvider.builder()
        .shards(shards)
        .tables(tables)
        .virtualNodesPerShard(160)
        .hashSeed(0)
        .showSql(false)
        .build();
```

`assertAllReachable` tries each primary and replica for three seconds. It throws an exception that names the shard and the role, for example:

```java
try {
    JShardDataSourceProvider.assertAllReachable(shards);
} catch (IllegalStateException e) {
    System.out.println(e.getMessage());     // Shard 'shard-a' (replica1) is not reachable.
}
```

`IllegalArgumentException` is thrown instead if a shard has no configuration at all. The check opens a short-lived connection to each database and closes it again, so it is cheap enough to run at startup.

`JShardDataSource.close()` closes ShardingSphere and every connection pool of the data source.

```java
JShardDataSource dataSource = JShardDataSourceProvider.builder()
        .shard("shard-a", dbA)
        .table("customer", "customer_id")
        .build();

dataSource.close();
dataSource.close();                          // harmless
dataSource.getConnection();                  // SQLException: This JShardDataSource has already been closed
```

- `close()` can be called more than once.
- After it, `getConnection` fails with `SQLException`.
- `getRouter()` keeps working, because it needs no database.

Several data sources can live in one program. Each has its own pools, so closing one does not touch the others. Because the router depends only on the names, the number of virtual nodes and the seed, two data sources with the same values always route a key to the same shard name.

## 11. In a Spring application

In a Spring application the data source is a *bean*, an object that Spring creates once and shares. The bean lives as long as the application and releases its pools when it ends.

```java
import org.j2os.platform.jshard.config.JShardConnectionConfig;
import org.j2os.platform.jshard.datasource.JShardDataSource;
import org.j2os.platform.jshard.datasource.JShardDataSourceProvider;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ShardConfig {

    @Bean(destroyMethod = "close")                      // pools are released when the application stops
    public JShardDataSource dataSource(@Value("${shop.db.password}") String password) throws Exception {
        JShardConnectionConfig dbA = JShardConnectionConfig.of(
                "org.postgresql.Driver", "jdbc:postgresql://db-a.example.com:5432/shop", "app", password, 10);
        JShardConnectionConfig dbB = JShardConnectionConfig.of(
                "org.postgresql.Driver", "jdbc:postgresql://db-b.example.com:5432/shop", "app", password, 10);

        return JShardDataSourceProvider.builder()
                .shard("shard-a", dbA)
                .shard("shard-b", dbB)
                .table("customer", "customer_id")
                .build();
    }
}
```

- The password comes from a property, not from the code.
- `destroyMethod = "close"` makes Spring call `close()` when the application stops.
- `build()` throws `Exception`, so the method declares it.

Everything else in the application uses the bean like any `DataSource`:

```java
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

import javax.sql.DataSource;

@Repository
public class CustomerRepository {

    private final JdbcTemplate jdbc;

    public CustomerRepository(DataSource dataSource) {          // the JShardDataSource bean
        this.jdbc = new JdbcTemplate(dataSource);
    }

    public void save(String id, String name) {
        jdbc.update("INSERT INTO customer (customer_id, name) VALUES (?, ?)", id, name);
    }

    public String findName(String id) {
        return jdbc.queryForObject("SELECT name FROM customer WHERE customer_id = ?", String.class, id);
    }
}
```

`JdbcTemplate` neither knows nor cares that there are several databases behind the data source. `findName` goes to one shard and returns, `save` stores the row on the shard that owns the key.

## 12. Putting it together: a customer directory that grows

A shop stores its customers in three databases and is about to add a fourth. The team needs to know before the change which customers will move, because those rows must be copied to the new database before the application is switched over. The router gives the answer without touching any database.

```java
import org.j2os.platform.jshard.router.JShardRouter;

import java.util.Set;

public class GrowthPlan {

    public static void main(String[] args) {
        JShardRouter today = new JShardRouter(Set.of("shard-a", "shard-b", "shard-c"), 160, 0);
        JShardRouter planned = new JShardRouter(Set.of("shard-a", "shard-b", "shard-c", "shard-d"), 160, 0);

        int total = 40;
        int toMove = 0;
        for (int i = 1; i <= total; i++) {
            String id = "c-" + (1000 + i);
            String from = today.getShardKey(id);
            String to = planned.getShardKey(id);
            if (!from.equals(to)) {
                toMove++;
                System.out.println(id + ": copy from " + from + " to " + to);
            }
        }
        System.out.println(toMove + " of " + total + " customers move; the other " + (total - toMove) + " stay.");
    }
}
```

```text
c-1013: copy from shard-b to shard-d
c-1026: copy from shard-b to shard-d
c-1029: copy from shard-c to shard-d
c-1030: copy from shard-a to shard-d
c-1031: copy from shard-a to shard-d
c-1034: copy from shard-c to shard-d
c-1040: copy from shard-a to shard-d
7 of 40 customers move; the other 33 stay.
```

The plan is simple and uses most of the module:

1. The three shards have the names `shard-a`, `shard-b` and `shard-c`, and the plan keeps the virtual-node count (160) and the seed (0) that the application uses. They must match the live configuration exactly, or the plan describes a different system.
2. `today` is the router for the current set of shards and `planned` is the router for the set with `shard-d`.
3. For each customer the two routers are compared. A customer whose shard changes is listed, with the shard it is stored on now and the shard that will own it.
4. In the output seven of the forty customers move, and all of them go to `shard-d`, as chapter 6 predicts. The other thirty-three stay where they are.

In a real migration the migration code would read the customer ids from the databases instead of building them from numbers, copy each listed row to the new shard, and delete the old copy once the application runs on the new configuration. The application then gets the fourth shard by adding one line to the builder from chapter 7:

```java
.shard("shard-d", dbD)
```

---

## Appendix A. Quick reference

### A.1 JShardConnectionConfig

| Member | Description |
|--------|-------------|
| `of(driver, url, user, password, poolSize)` | creates a configuration from the five required values |
| `builder()` | starts a builder with optional pool settings |
| `getJdbcUrl()`, `getUsername()`, `getPassword()`, `getDriverClassName()`, `getPoolSize()` | the required values |
| `getConnectionTimeoutMs()`, `getIdleTimeoutMs()`, `getMaxLifetimeMs()`, `getValidationTimeoutMs()`, `getKeepaliveTimeMs()` | optional values, or `null` when not set |
| `getMinimumIdle()` | optional value, or `null` when not set |
| `equals`, `hashCode` | compare all fields, including the password |
| `toString()` | log-safe: no password, masked user name, no query string |

### A.2 JShardTableConfig

| Member | Description |
|--------|-------------|
| `new JShardTableConfig(name, shardingColumn)` | validates both names |
| `name()` | the table name |
| `shardingColumn()` | the column that decides the shard |

### A.3 JShardDataSourceProvider

| Member | Description |
|--------|-------------|
| `builder()` | starts a builder |
| `assertAllReachable(map)` | checks every primary and replica; throws if one is not reachable |
| `Builder.shard(name, primary, replicas...)` | adds a shard |
| `Builder.shards(map)` | adds several shards; the first element of each list is the primary |
| `Builder.table(name, column)` | adds a sharded table |
| `Builder.tables(list)` | adds several tables |
| `Builder.virtualNodesPerShard(n)` | virtual nodes per shard, 1 to 100000; default 160 |
| `Builder.hashSeed(n)` | the hash seed; default 0 |
| `Builder.showSql(boolean)` | logs the SQL; default off |
| `Builder.build()` | creates the pools and the data source; declares `throws Exception` |

### A.4 JShardDataSource

| Member | Description |
|--------|-------------|
| `getConnection()` | a connection to the sharded data source; fails after `close()` |
| `getRouter()` | the router for this data source |
| `close()` | closes ShardingSphere and all pools; safe to repeat |
| `unwrap`, `isWrapperFor` | answer for `JShardDataSource` and `DataSource` themselves, otherwise ask the inner data source |

### A.5 JShardRouter

| Member | Description |
|--------|-------------|
| `new JShardRouter(names, virtualNodes, seed)` | a router for a fixed set of shard names |
| `getShardKey(key)` | the name of the shard that owns the key |

### A.6 JShardHashAlgorithm

| Member | Description |
|--------|-------------|
| `DEFAULT_VIRTUAL_NODES_PER_SHARD` | 160 |
| `DEFAULT_HASH_SEED` | 0 |
| `MAX_VIRTUAL_NODES_PER_SHARD` | 100000 |
| `init(props)` | reads `sharding-count`, `virtual-nodes-per-shard` and `hash-seed` |
| `getType()` | `JSHARD_CONSISTENT_HASH` |
| `predictShard(names, key)` | the shard for a key |

### A.7 JShardDataSourceRegistry

| Member | Description |
|--------|-------------|
| `create(key, config)` | creates and registers a connection pool under a key |
| `closeAll()` | closes every pool of the registry |
| `isReachable(config, seconds)` | static; `true` if a connection can be made and is valid within the time (at least one second) |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| a blank driver, URL, user name or password | `IllegalArgumentException` |
| a pool size of zero or less | `IllegalArgumentException` |
| a table or column name with a dot, comma, quote, space or control character | `IllegalArgumentException` |
| `build()` without a shard or without a table | `IllegalArgumentException` |
| an invalid shard name | `IllegalArgumentException` at `build()` |
| the same table or the same shard name twice | `IllegalArgumentException` |
| `shard(null, ...)` or a `null` primary | `NullPointerException` |
| a shard with an empty configuration list | `IllegalArgumentException` |
| virtual nodes below 1 or above 100000 | `IllegalArgumentException` |
| a blank key for the router | `IllegalArgumentException` |
| a router without shard names | `IllegalArgumentException` |
| an algorithm used with a shard count that differs from `sharding-count` | `IllegalStateException` |
| `assertAllReachable` finds an unreachable database | `IllegalStateException` naming the shard and role |
| `build()` fails after some pools were created | the pools are closed, then the failure is thrown |
| `getConnection()` after `close()` | `SQLException` |
| `close()` twice | no effect |
| a statement that carries a range or no key | sent to every shard |
| a key that is `null` or blank in a statement | the statement fails; see Appendix D |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Shard names and the seed are part of the address.** Changing either moves keys. Choose them once.
- **jshard does not move rows.** After a shard is added or removed, copy the affected rows yourself before switching.
- **Tables must exist in every database.** jshard does not create them, and all shards need the same table name.
- **Queries without the key ask every shard.** One slow shard slows the whole query. Keep common queries keyed.
- **Do not change the key of a row.** It would belong on another shard.
- **The key is text.** Store numeric identifiers as text columns.
- **A shard that is down fails the queries that need it.** There is no automatic failover.
- **Writes across shards are not atomic.** Design for it, or keep related rows on the same shard by using the same key.
- **A router is only as good as its inputs.** Plans made with different names, virtual nodes or seed than the live system describe another system.
- **Do not log passwords.** Log the configuration with `toString()`, which is safe.

### C.2 Behaviour worth knowing

- The shard of a key depends only on the shard names, the virtual-node count and the seed.
- The hash is MurmurHash3, placed on a ring of 64-bit numbers. Shards are processed in alphabetical order when the ring is built, so a rare collision between two virtual nodes is resolved the same way on every start.
- Rings are cached for the whole JVM, up to 1000 rings, and a ring that is not used for six hours is dropped and rebuilt on the next use.
- A `JShardRouter` is built once and does not change. It does not follow later changes to the cluster.
- A range condition on the sharding column is sent to every shard.
- Each `build()` creates its own pools. Data sources do not share connections.
- Optional pool settings that are not set use the HikariCP defaults.
- The `sharding-count` check makes the algorithm refuse to run if ShardingSphere offers a different number of shards than the data source was built with.

### C.3 Limits

- **At most 100000 virtual nodes per shard**, 160 by default.
- **Text keys only.** The algorithm hashes `String` values.
- **One sharding column per table**, and one strategy: the hash ring. There is no table-level sharding inside a database, no range sharding and no custom algorithm.
- **Names without special characters.** Shard, table and column names must not contain dots, commas, quotes, spaces or control characters.
- **A password is required.** A configuration with a blank password is refused.
- **No schema management, no data migration, no distributed transactions.**

### C.4 Tuning

- Leave the virtual nodes at 160 unless the measured balance is poor. Raise it to a few thousand for a more even split.
- Size each `poolSize` for the load of that database and keep the sum of all pools below what the databases accept. A larger pool does not make slow queries faster.
- Set `connectionTimeoutMs` low enough that a stuck database fails requests quickly instead of piling them up.
- Run `assertAllReachable` at startup so that a mistake in a password or a host shows up at once and with a clear message.
- Use `showSql(true)` while developing to check where each statement goes, and turn it off afterwards.
- Plan growth with a `JShardRouter` before you touch the databases.
- Keep the shard names, the seed and the virtual-node count in one place in the configuration, and never change them casually.
- Choose a key that your common queries already have, and use the same key for related tables.

## Appendix D. Message reference

**IllegalArgumentException**

| Message | Meaning |
|---------|---------|
| `ShardConfig.<field> must not be null/blank` | a required connection value is missing; the field is `jdbcUrl`, `username`, `password` or `driverClassName` |
| `poolSize must be greater than zero` | the pool size is zero or negative |
| `ShardingTableConfig.<field> must not be null/blank` | the table or column name is empty; the field is `name` or `shardingColumn` |
| `ShardingTableConfig.<field> must not have leading/trailing whitespace: '<value>'` | the name starts or ends with a space |
| `ShardingTableConfig.<field> must not contain '.', ',', or a quote character: '<value>'` | the name contains a dot, a comma or a quote |
| `ShardingTableConfig.<field> must not contain whitespace or control characters: '<value>'` | the name contains a space or a control character |
| `At least one shard must be added via .shard(...)` | `build()` without shards |
| `At least one table must be added via .table(...)` | `build()` without tables |
| `Shard '<name>' has already been added` | the same shard name twice |
| `Duplicate table name in the tables list: '<name>'` | the same table twice |
| `Shard '<name>' requires at least one config (primary)` | `shards(map)` with an empty list for a shard |
| `Shard '<name>' requires at least one ShardConfig (primary)` | the same problem found by `assertAllReachable` |
| `A shard name must not be null/blank` | the shard name is empty |
| `A shard name must not have leading/trailing whitespace: '<name>'` | the shard name starts or ends with a space |
| `A shard name must not contain '.', ',', or a quote character (ShardingSphere syntax restriction): '<name>'` | the shard name contains a dot, a comma or a quote |
| `A shard name must not contain whitespace or control characters: '<name>'` | the shard name contains a space or a control character |
| `virtualNodesPerShard must be greater than zero` | zero or negative |
| `virtualNodesPerShard (<n>) exceeds the allowed maximum (100000)` | too many virtual nodes |
| `virtual-nodes-per-shard must be positive` | the same rule in the algorithm properties |
| `virtual-nodes-per-shard (<n>) exceeds the allowed maximum (100000)` | the same limit in the algorithm properties |
| `sharding-count must be positive` | the algorithm property is zero or negative |
| `shardNames must not be null/empty` | a router without shard names |
| `The sharding column value for table '<table>' must not be null/empty` | the key is `null` or blank; the table is `n/a` when the router is used alone |

**NullPointerException**

| Message | Meaning |
|---------|---------|
| `The shard name must not be null` | `shard(null, ...)` |
| `The primary config for shard '<name>' must not be null` | a `null` primary |
| `The shardConfigs map must not be null` | `shards(null)` |
| `The tableConfigs list must not be null` | `tables(null)` |

**IllegalStateException**

| Message | Meaning |
|---------|---------|
| `Shard '<name>' (primary) is not reachable.` | `assertAllReachable` cannot connect to the primary |
| `Shard '<name>' (replica<n>) is not reachable.` | the same for a replica; the first replica is `replica1` |
| `The number of available shards (<n>) does not match the configured sharding-count (<m>)` | the algorithm is given another number of shards than it expects |
| `No shards are available for table '<table>'` | the algorithm is given no shards |
| `Could not resolve a hash ring collision after <n> attempts` | the ring cannot place a virtual node; practically never seen |
| `A DataSource is already registered under key '<key>' in this registry` | `create` is called twice with the same key |

**SQLException**

| Message | Meaning |
|---------|---------|
| `This JShardDataSource has already been closed` | `getConnection` after `close()` |
