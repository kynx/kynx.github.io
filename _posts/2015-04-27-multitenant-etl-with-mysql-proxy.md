---
layout: post
title: Multi-tenant ETL with MySQL Proxy
description: Techniques for creating multi-tenant ETL solutions using Pentaho PDI and MySQL Proxy
tags: etl pentaho multi-tenant mysql
comments: true
---

People can mean a number of different things when they say "multi-tenancy". Pentaho have a (rather low quality) [presentation](https://www.youtube.com/watch?v=sDaDDXcV79E) of four different
use cases. Unfortunately they don't tell you how to get any of them working.

This article looks at the basic database layout and ETL design side when implementing a multi-tenanted data warehouse.<!--more--> It may be of use if some of these ring true:

1. Your tenant source databases are identically structured
2. You need a "master" data warehouse for high-level analysis across all tenants
3. Tenants will be accessing their data warehouses using 3rd party tools via ODBC / JDBC
4. You are using MySQL for your data warehouse
5. You have a single-tenant setup already that you want to roll out to other tenants

The solution I've come up with evolved over time. I'll first discuss the challenges thrown up by multi-tenant ETL, then look at how we met them. I'm not entirely sure it
isn't a horrible hack, and there certainly are other ways of doing it. But no matter, it works.


## Separating tenant data

The key requirement in any multi-tenant reporting platform is that tenant data
is partitioned in such a way that there is no chance of one tenant accidentally (or maliciously) accessing another's data.

In practice this will mean either:

* Having a single data warehouse and implementing some form of row-level access control
* Having separate data warehouses for each tenant

MySQL has no concept of row-level access controls, but that doesn't rule out putting a `TenantID` column on each and every row. This could then be used to filter the data for reports.

However that approach has two problems: each and every report will need to include the `TenantID` in it's `WHERE` clause; and there is no way to filter the rows when the tenant is coming in over ODBC.

The `TenantID` column has its uses, which I'll come back to further down. But for access control, unless your database natively supports row-level access, it is easier and safer to have separate data warehouses for each tenant: 

* With a little bit of wiring Pentaho can switch the datasource it uses at user login: you can share reports between tenants without re-writing them
* Bog-standard `GRANT`s can control access for anyone coming in over ODBC
* Many smaller data warehouses = better query performance


## Combining tenant data

The next challenge comes when you need to report across all tenant data.

If you are only combining some of the data, or are further transforming it for the specific uses it will be put to, this shouldn't be a big deal: you simply create another ETL job that lifts and loads each tenant's data to the new structure, taking care to create new surrogate keys on the dimensions and re-link the facts to these.

We wanted all the data from all tenants. I'd already started on a single-tenant ETL for our guinea-pig client and didn't fancy writing a whole new ETL process to populate the master data warehouse. I also wanted to be able to take a report written for one tenant and run it against all tenants with no modifications.

If the existing ETL process could _simultaneously_ populate both the tenant and master data warehouses I thought development would be faster and maintenance simpler.


## The problem of surrogacy

I looked at a number of ways to achieve this simultaneous population of two data warehouses on the DB side. These included:

1. Tenant data warehouses composed only of views on the master that filtered by `TenantID`. The ETL would then simply run against the master
2. Using MySQL's [MERGE](https://dev.mysql.com/doc/refman/5.5/en/merge-storage-engine.html) storage engine to combine all identical tenant tables into a single master table
3. Hacking the PDI steps so I could specify a primary and secondary connection they would write to

The first works, but the views require maintenance and lose the performance benefits of multiple small(ish) data warehouses. A look at the amount of code I'd be modifying ruled out the third.

The second had merit, but unfortunately you cannot specify which of the underlying tables in the `MERGE` gets the update. However, it does illustrate the key problem I kept encountering. 

Let me explain. For my scheme to work, surrogate keys would have to be unique across all tenant databases.

![Technical key](/public/images/2015-04-27/technical_key.png){: .right-image .keyline }
PDI's [Dimension Lookup / Update](http://infocenter.pentaho.com/help/index.jsp?topic=%2Fpdi_user_guide%2Fconcept_pdi_usr_dimension_lookup_update.html) step gets the next surrogate key by doing a `SELECT MAX(id)` on the same table you're inserting into. There isn't an option to get it from somewhere else, like a central repository of tables and latest values (although this was suggested in [a JIRA](http://jira.pentaho.com/browse/PDI-7017) dating back to 2011).

I briefly contemplated introducing a step that inserted a dummy row in the dimension with the next surrogate key before the transform started then another step that deleted it later, but that seemed ugly and brittle.

What I needed was a way to intercept the queries on the wire. Then I could:

* Redirect all `SELECT MAX(SurrogateKey) FROM dimension` queries to the master
* Run all `INSERT`, `UPDATE` and `DELETE` queries on both tenant and master
* Leave any other `SELECT`s to run on just the tenant


## Enter MySQL Proxy

![Dolphin at the switchboard](/public/images/2015-04-27/mysql_proxy_switchboard.jpg){: .left-image}
It turns out MySQL has a tool that is designed for precisely this kind of foolishness: [MySQL Proxy](http://dev.mysql.com/doc/mysql-proxy/en/).

It sits between the client application (PDI) and the MySQL server listening on port 4040. It intercepts the queries and feeds them to a custom [Lua](http://www.lua.org) script, which can re-write queries, send them to different servers and manipulate the results. Power indeed.

Now I'd never run into Lua before, but it's used in stuff from digital TVs to World of Warcraft and Angry Birds. And it turns out Lua is pretty easy to learn. The [MySQL wire protocol](https://dev.mysql.com/doc/internals/en/client-server-protocol.html) isn't (what wire protocol is?), but the documentation is straightforward.

After a day scratching my head, here's what I came up with:

{% gist kynx/37429efa154531c67cc5 mt-proxy.lua %}

OK, that's quite a chunk of code. I'm not going to go through it line-by-line, but hopefully the comments give some idea what's going on. If not, feel free to pester me in the feedback section below. The key parts are:

* **get_mt_db()** This returns the name of the master database based on the name of the tenant. Adapt to your needs.
* **read_query()** This is where we intercept the incoming query and decide what to do with it, depending on the type of command. Whenever we append a query we pass an ID as the first parameter. This is used when MySQL passes the results back to...
* **read_query_results()** This uses the ID to determine where the query was run and takes the appropriate action with the results and errors it finds

## Implementation

### Database connections

The first thing to do (if you haven't already) is [parameterize your database connections](http://wiki.pentaho.com/display/EAI/Named+Parameters) in PDI so that the port number can be changed on the fly. This will enable you to switch from sending queries through the proxy to querying the database directly.

In practice you'll probably want two separate connections: one for your normal ETL work - which uses the proxy - and another for one-off jobs that are designed to work on one database at a time, such as DDL queries.

### TenantID

I mentioned adding a `TenantID` column to all tables in your data warehouses while discussing row-level access controls. You will want to do this anyway so you can identify where data came from.

There are two approaches. The first is to use a surrogate key and place it on every single table. However I'm a little more semantic, and put a `SourceDB` varchar column on just the dimensions so I can see where the data came just by browsing the table. The downside of that approach is that to identify the provenance of a particular fact it needs to be joined on a dimension.

In either case you will need a centralised database of tenants. In our setup this includes details such as which repository they use (ie "production" or "staging"), timezones (for scheduling ETL runs out of working hours) and for driving the ETL process itself.

### Creating new tenants

Hopefully you're going to be adding new tenants all the time! If so, you'll want to automate the process. To tackle this I created a new Kettle job that:

* Creates an empty tenant database
* Copies the structure of the tables / views / etc from the master
* Populates tenant dimensions with "special" rows (ie "Unknown", "Not found")
* Populates any common tables, like Date dimensions

This is likely to be pretty specific to your data warehouse structure, but if there's any interest I'll post some examples.

### Maintenance

Our setup requires that all tenant databases and the master database share exactly the same structure. Any changes to the schema need to be performed across all databases. Again, a centralised database of tenants will help automate this process.

Any DBA grey beard will tell you that when store the same piece of information in two places, sooner or later one of them will be wrong. The data-warehouse theocracy merrily cock a snoot at such normalised orthodoxy, but the grey beards have got a point.

At some point something _will_ go wrong, and your master data warehouse will get out of sync with the tenants. This will happen whatever method you use to populate the master. You will need some tools to identify any differences and in the worse case rebuild the master from all the tenants. One advantage of having unique surrogate keys is that the latter can be as simple as `INSERT INTO table SELECT * FROM tenantdw.table`.


## Pros and Cons

If I haven't scared you off by now, here's what I've learned from pushing this puppy out.

### Good bits

* Separates multi-tenancy from the ETL process entirely: simply change the port number for your connection and you're back to a single-tenant ETL process
* No need for additional ETL jobs to populate master data warehouse
* Lua script could be easily adapted so master data warehouse resides on another server from the tenant, again with no change to the ETL


### Not so good bits

* MySQL Proxy is officially "alpha" software. In practice this has not been an issue for us, and it is in use on some fairly large sites for jobs such as read/write splitting
* Introduces some latency (~400 microseconds) to every request
* Introduces another layer in the process: one more thing to go wrong, extra knowledge required before you can hack
* The ETL runs for each tenant cannot run in parallel or you will get duplicate IDs

For us, the last point is probably the biggest drawback. It's not a killer yet, but as we scale I anticipate it becoming more of an issue. Over the last year I've learned a few ETL tricks that would make populating the master in a more conventional, PDI-only way less of a maintenance burden. I'll try and write those up in a future post.
