Redis FDW for PostgreSQL 9.1+
==============================

This PostgreSQL extension implements a Foreign Data Wrapper (FDW) for
the Redis key/value database: http://redis.io/

This code was originally experimental, and largely intended as a pet project 
for Dave to experiment with and learn about FDWs in PostgreSQL. It has now been
extended for closer to production use by Andrew.

By all means use it, but do so entirely at your own risk! You have been
warned!

Building
--------

To build the code, you need the hiredis C interface to Redis installed 
on your system. You can checkout the hiredis from
[https://github.com/redis/hiredis](Github), 
or it might be available for your OS as it is for Fedora, for example.

Once that's done, the extension can be built with:

    PATH=/usr/local/pgsql91/bin/:$PATH make USE_PGXS=1
    sudo PATH=/usr/local/pgsql91/bin/:$PATH make USE_PGXS=1 install

(assuming you have PostgreSQL 9.1 in /usr/local/pgsql91).

Make necessary changes for 9.2 and later.

You will need to have the right branch checked out to match the PostgreSQL
release you are buiding against, as the FDW API has changed from release 
to release.

Dave has tested the original on Mac OS X 10.6 only, and Andrew on Fedora and
Suse. Other *nix's should also work.
Neither of us have tested on Windows, but the code should be good on MinGW.

Limitations
-----------

- There's no such thing as a cursor in Redis, or MVCC, which leaves us
  with no way to atomically query the database for the available keys
  and then fetch each value. So, we get a list of keys to begin with,
  and then fetch whatever records still exist as we build the tuples.

- We can only push down a single qual to Redis, which must use the 
  TEXTEQ operator, and must be on the 'key' column.

- There is no support for non-scalar datatypes in Redis
  such as lists, for PostgreSQL 9.1. There is such support for later releases.

Usage
-----

The following parameters can be set on a Redis foreign server:

address:	The address or hostname of the Redis server.
	 	Default: 127.0.0.1

port:		The port number on which the Redis server is listening.
     		Default: 6379

The following parameters can be set on a Redis foreign table:

database:	The numeric ID of the Redis database to query.
	  	Default: 0

(9.2 and later) tabletype: can be 'hash', 'list', 'set' or 'zset'
	    Default: none, meaning only look at scalar values.

(9.2 and later) tablekeyprefix: only get items whose names start with the prefix
        Default: none

(9.2 and later) tablekeyset: fetch item names from the named set
        Default: none

You can only have one of tablekeyset and tablekeyprefix.

Structured items are returned as array text, or, if the value column is a
text array as an array of values. In the case of hash objects this array is
an array of key, value, key, value ...

The following parameter can be set on a user mapping for a Redis
foreign server:

password:	The password to authenticate to the Redis server with. 
     Default: <none>

Example
-------

	CREATE EXTENSION redis_fdw;

	CREATE SERVER redis_server 
		FOREIGN DATA WRAPPER redis_fdw 
		OPTIONS (address '127.0.0.1', port '6379');

	CREATE FOREIGN TABLE redis_db0 (key text, value text) 
		SERVER redis_server
		OPTIONS (database '0');

	CREATE USER MAPPING FOR PUBLIC
		SERVER redis_server
		OPTIONS (password 'secret');

	CREATE FOREIGN TABLE myredishash (key text, value text[])
		SERVER redis_server
		OPTIONS (database '0', tabletype 'hash, tablekeyprefix 'mytable:');
	 
Testing
-------

The tests for 9.2 and later assume that you have access to a redis server
on the localmachine with no password, and uses database 15, which must be empty,
and that the redis-cli program is in the PATH when it is run.
The test script checks that the database is empty before it tries to
populate it, and it cleans up afterwards.



Authors
------- 

Dave Page
dpage@pgadmin.org

Andrew Dunstan
andrew@dunslane.net
