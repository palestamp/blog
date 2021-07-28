---
title: "Transaction Isolation Levels"
date: 2021-07-12T13:02:40+02:00
draft: true
tags: [postgres, database]
---

1. ACID
2. I stands for Isolation
3. Isolation vs Consistency
4. Why different isolation levels? Why trade isolation for performance?
5. History-based walkthrough
    1. Snapshot Isolation vs Serializable
    2. Serializable Snapshot Isolation

# ACID

In database systems ACID refers to a set of guarantees about transactions behavior. In the essence, transaction defined by those guarantees. Let's try to see what ACID is and why it is useful. 

Let's say there is a John, and John is in a urgent need of a new computer monitor. Let's also imagine that we live in a times when monitor manufacturers decided that selling monitors and monitor power adaptors separately is a very good and eco-friendly idea. John browses reseller's web-shop and finds a perfect monitor for himself, after a bit of time he also finds a suitable power adaptor. He adds both, monitor and power adaptor into the web-shop basket. John clicks "Order" and our transactional journey begins.

To process John's order request web-shop needs to save it into the database. Web-shop makes use of a common SQL database and stores each order item in a separate row, in this sense "monitor" and "power adaptor" will be stored as two separate rows.

For the sake of example, let's imagine that two insert statements will be performed:

```sql
BEGIN;
INSERT INTO order_items (order_id, item, amount) VALUES (1, "monitor", 1);
INSERT INTO order_items (order_id, item, amount) VALUES (1, "power adaptor", 1);
COMMIT;
```

To store those rows correctly database needs to ensure ACID guarantees.

## Atomicity

We should save both order items together, monitor is useless without power adaptor, and most likely the same is true for the opposite situation. This is where atomicity helps:

> ... each transaction is treated as a single "unit", which either succeeds completely, or fails completely: if any of the statements constituting a transaction fails to complete, the entire transaction fails and the database is left unchanged.

## Consistency (Correctness)

Stock of order items is finite, and web-shop can not sell what it does not have at hand. So we need to ensure that we only can sell an order item if it is in the stock. On the database level it can (and should) be ensured by constraints - stock for an order item can never go below 0. Consistency guarantee is responsible for that:

> Consistency ensures that a transaction can only bring the database from one valid state to another, maintaining database invariants: any data written to the database must be valid according to all defined rules.

In the world of data storages "Consistency" has two different connotations. First exists in ACID and second comes from the CAP theorem. This creates a bit of confusion when we talk about distributed (hence, CAP theorem is applicable) ACID-compliant database. To disambiguate this we can also name C in ACID as "Correctness".

## Isolation

It turns out that John is not the only one who wants to buy this monitor. More to say, this particular model of monitor is very popular and several people trying to buy it at the same moment of time. Isolation guarantee will ensure that the same monitor item will not be sold to two different persons, leaving one of the customers unhappy and web-shop management enraged.

> Isolation ensures that concurrent execution of transactions leaves the database in the same state that would have been obtained if the transactions were executed sequentially.

Quote above describes perfect situation, in reality it is quite hard and expensive to ensure "virtually sequential" execution of concurrent transactions. We will talk more about it below.

## Durability

After John pressed an "Order" button he receives message on the screen saying that order was registered successfully. It would be a shame if on delivery date it will turn out that just after John made an order web-shops database has a brief power outage and order was never persisted. Durability protects us from such cases - if John saw that order was successfully registered - power outage will not destroy this data.

> Durability guarantees that once a transaction has been committed, it will remain committed even in the case of a system failure (e.g., power outage or crash).

---
---

<p style="text-align: center;"> NOTES </p>

---
---


# History

## Sources

* [A History of Transaction Histories](https://ristret.com/s/f643zk/history_transaction_histories)
* [Hermitage: Testing the “I” in ACID](https://martin.kleppmann.com/2014/11/25/hermitage-testing-the-i-in-acid.html)

* https://github.com/ept/hermitage/blob/master/postgres.md - <mark>(See also for anomalies list)</mark>

---

> The safety guarantees provided by transactions are often described by the well-known acronym ACID, which stands for Atomicity, Consistency, Isolation, and Durability. It was coined in 1983 by Theo Härder and Andreas Reuter [7] in an effort to establish precise terminology for fault-tolerance mechanisms in databases.
> *Designing Data-Intensive Systems, ch 7, also see [Principles of Transaction-Oriented Database Recovery](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.87.2812&rep=rep1&type=pdf)*

ACID is there to simplify application developer reasoning model and offer predictability while working with database.

---

<mark>Can non-repeatable read happens if only one transaction uses `REPEATABLE READ` isolation level and other uses `READ COMMITED`?</mark>

---

## Milestones

1. 1975 - [*Granularity of Locks and Degrees of Consistency in a Shared Data Base*](http://citeseer.ist.psu.edu/viewdoc/download?doi=10.1.1.92.8248&rep=rep1&type=pdf) The team working on the first SQL database **System R** already realized this in 1975, and decided to offer weaker isolation levels than serializability

1. 1992 - [*ANSI SQL-92*](http://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt) first attempt to make a uniform definition of transaction isolation levels.

2. 1995 - [*A Critique of ANSI SQL Isolation Levels*](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf) which points out that the ANSI SQL definition of serializable is not serializable. Also points out that defining isolation in terms of “the absence of specific anomalous phenomena” is bad, and advises to approach the question in terms of "transaction histories". New anomalies defined.

3. 1999 - [*Weak Consistency: A Generalized Theory and Optimistic Implementations for Distributed Transactions*](http://publications.csail.mit.edu/lcs/pubs/pdf/MIT-LCS-TR-786.pdf) introduces concept of  Direct Serialization Graph. Then he maps each badness back to the ANSI definitions <mark>(because ANSI terms are still understandable by broad public?)</mark>. New anomalies defined. It is a first time someone coherently gave mathematical definitions of isolation levels in terms of properties one can observe in a transaction history graph.

4. 2005 - [*Making Snapshot Isolation Serializable*](https://dsf.berkeley.edu/cs286/papers/ssi-tods2005.pdf) It essentially takes a vanilla snapshot isolation database, and layers on some checks in the SQL statements, to ensure that you have serializability. <mark>(because snapshot isolation was in most of the dbs those times?) and (what is TPC-C's "One Weird Trick"?)</mark> 

5. 2008 - [*Serializable Isolation for Snapshot Databases*](https://ses.library.usyd.edu.au/bitstream/handle/2123/5353/michael-cahill-2009-thesis.pdf;jsessionid=B2622D4FDBA315E83216801C1504D12F?sequence=1) This paper basically runs with that idea, except instead of having those checks be written in application code, they do the checks at the database level. The technique is called **SSI** (Serializable Snapshot Isolation)

6. 2021 - Postgres implements SSI in version 9.1, from now on `REPEATABLE READ` corresponds to SI and `SERIALIZABLE` to SSI.

# Vocabulary

> Isolation levels define the degree to which a transaction must be isolated from the data modifications made by any other transaction in the database system. A transaction isolation level is defined by the following phenomena.

* **Dirty Read** – A Dirty Read is the situation when a transaction reads a data that has not yet been committed. For example, Let’s say transaction 1 updates a row and leaves it uncommitted, meanwhile, Transaction 2 reads the updated row. If transaction 1 rolls back the change, transaction 2 will have read data that is considered never to have existed.

* **Non Repeatable read** – Non Repeatable Read occurs when a transaction reads same row twice, and get a different value each time. For example, suppose transaction T1 reads data. Due to concurrency, another transaction T2 updates the same data and commit, Now if transaction T1 rereads the same data, it will retrieve a different value.

* **Phantom Read** – Phantom Read occurs when two same queries are executed, but the rows retrieved by the two, are different. For example, suppose transaction T1 retrieves a set of rows that satisfy some search criteria. Now, Transaction T2 generates some new rows that match the search criteria for transaction T1. If transaction T1 re-executes the statement that reads the rows, it gets a different set of rows this time.

> What is a difference between Non Repeatable Read and Phantom Read?


```sql
SELECT * FROM pages WHERE id = 1;
```

You can think about isolation anomalies as
* Dirty Read - when one transaction reads the data that was not committed (by another transaction) yet or will not be ever committed.
* Non-repeatable read - when within one transaction same query returns row with different column values. 
* Phantom read - when within one transaction same query can return different number of rows [^1]

# Problems with SQL Standard definition

SQL standard that fails to accurately define database isolation levels. [^6][^7]



[^2]: https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf

[^9]: https://www.interdb.jp/pg/pgsql05.html - Postgres MVCC - postgres concurrency control mechanism.

[^10]: https://www.interdb.jp/pg/pgsql06.html - describes how VACUUM procedure works in postgres - in high load cases - this one is essential knowledge (maybe not to the depth it described though)

[^3]: https://blog.sentry.io/2015/07/23/transaction-id-wraparound-in-postgres - real-life story on managing transaction id wraparounds.

[^4]: https://engineering.nordeus.com/postgres-locking-revealed/ Locking mechanisms overview

[^5]: https://www.postgresql.org/docs/13/transaction-iso.html - postgres transaction isolation levels. Those levels are a bit different from other dbs (they are generally slightly different in different dbs).

[^6]: https://fauna.com/blog/introduction-to-transaction-isolation-levels#what-is-an-isolation-level

[^7]: https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf

[^8]: https://ristret.com/s/f643zk/history_transaction_histories


# Reading list

* ACIDRain: http://www.bailis.org/papers/acidrain-sigmod2017.pdf
* Generalized isolation level definitions: https://15721.courses.cs.cmu.edu/spring2018/papers/04-occ/adya-icde2000.pdf
* When is "ACID" ACID? Rarely: http://www.bailis.org/blog/when-is-acid-acid-rarely/
* Designing Data-Intensive Applications: look for isolation chapter
* HAT, not CAP: Towards Highly Available Transactions: http://www.bailis.org/papers/hat-hotos2013.pdf
* Highly Available Transactions: Virtues and Limitations (Extended Version): https://arxiv.org/pdf/1302.0309.pdf

# Other fun things

* http://go-database-sql.org/errors.html - good for understanding common pitfalls while working with db from Go.
* https://medium.com/avitotech/how-to-work-with-postgres-in-go-bad2dabd13e4 - Go and PgBouncer

## Case for MySQL and Postgres being "easy reasonable"

```sql
CREATE TABLE pages (id int, amount int check(amount > 0));
INSERT INTO pages VALUES (1, 30), (2, 47);
```

T1
```sql
UPDATE pages SET amount = -1 WHERE id = 2;
SELECT * FROM pages;
```

T2
```sql
UPDATE pages SET amount = 100 WHERE id = 2;
```

MySQL's result set for T1's select will be (1, 30), (2, 47);

Postgres will abort transaction after check constraint violation;