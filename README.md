repmgr: Replication Manager for PostgreSQL clusters
===================================================

`repmgr` is an open-source tool suite for mananaging replication and failover
among multiple PostgreSQL server nodes. It enhances PostgreSQL's built-in
hot-standby capabilities with a set of administration tools for monitoring
replication, setting up standby servers and performing failover/switchover
operations.

This document assumes you are familiar with PostgreSQL replication setup and
Linux/UNIX system administration.

Repmgr 3 Features
-----------------

`repmgr 3` takes advantage of features introduced in PostgreSQL 9.3 and
later, including timeline following and replication slots, to make setting
up and managing replication smoother and easier. For earlier PostgreSQL
versions please continue use the 2.x branch.

New features in `repmgr 3` include:

* uses pg_basebackup to clone servers
* Supports timeline following, meaning a standby does not have to be
  restarted after being promoted to master
* supports cascading replication
* supports tablespace remapping (in PostgreSQL 9.3 via rsync only)
* enables use of replication slots (PostgreSQL 9.4 and later)
* usability improvements, including better logging and error reporting

Conceptual Overview
-------------------

`repmgr` provides two binaries:

 - `repmgr`: a command-line client to manage replication and `repmgr`
   configuration
 - `repmgrd`: an optional daemon process which runs on standby nodes to monitor
   replication and node status

Each PostgreSQL node requires a `repmgr.conf` configuration file; additionally
it must be "registered" with the current master node using the `repmgr`
command-line client. `repmgr` stores information about managed nodes in a
custom schema on the node's current master database; see below for details.

Supported Releases
------------------

`repmgr` works with PostgreSQL 9.3 and later. All server nodes must be running
the same PostgreSQL major version, and preferably should be running the same
minor version.

PostgreSQL versions 9.0 ~ 9.2 are supporeted by `repmgr` version 2.

Requirements
------------

`repmgr` will work on any Linux or UNIX-like environment capable of running
PostgreSQL. `rsync` and password-less SSH connections between servers are
only required if `rsync` is to be used to clone standby servers. Also,
if `repmgr` is meant to copy PostgreSQL configuration files located outside
of the main data directory, pg_basebackup will not be able to copy these,
and `rsync` will be used.

Installation
------------

`repmgr` must be installed on each PostgreSQL server node.

* Packages
 - RPM packages for RedHat-based distributions are available from PGDG
 - Debian/Ubuntu provide .deb packages.

  It is also possible to build .deb packages directly from the `repmgr` source;
  see README.rst for further details.

* Source installation
 - `repmgr` source code is hosted at github (https://github.com/2ndQuadrant/repmgr);
   tar.gz files can be downloaded from https://github.com/2ndQuadrant/repmgr/releases .

   `repmgr` can be built easily using PGXS:

       sudo make USE_PGXS=1 install

Configuration
-------------

### Server configuration

By default, `repmgr` uses PostgreSQL's build-in replication protocol
for communicating with remote servers, e.g. when cloning a primary.
However, password-less SSH logins may need to be enable for the database
system user (typically `postgres`) between all server nodes if you
wish `repmgr` to copy configuration files not located in

### PostgreSQL configuration

PostgreSQL node needs to be configured for replication with the following
settings:

    wal_level = 'hot_standby'      # minimal, archive, hot_standby, or logical
    archive_mode = on              # allows archiving to be done
    archive_command = 'cd .'       # command to use to archive a logfile segment
    max_wal_senders = 10           # max number of walsender processes
    wal_keep_segments = 5000       # in logfile segments, 16MB each; 0 disables
    hot_standby = on               # "on" allows queries during recovery

Note that `repmgr` expects a default of 5000 wal_keep_segments, although this
value can be overridden when executing the `repmgr` client.

From PostgreSQL 9.4, replication slots are available, which remove the
requirement to retain a fixed number of WAL logfile segments. See
'repmgr configuration' for details.

Additionally, `repmgr` requires a dedicated PostgreSQL superuser account
and a database in which to store monitoring and replication data. The `repmgr`
user account will also be used for replication connections from the standby,
so a separate replication user with the `REPLICATION` privilege is not required.
The database can in principle be any database, including the default `postgres`
one, however it's probably advisable to create a dedicated database for `repmgr`
usage. Both user and database will be created by `repmgr`.


### repmgr configuration

Each PostgreSQL node requires a `repmgr.conf` configuration file containing
identification and database connection information. This is a sample
minimal configuration:

    cluster=test
    node=1
    node_name=node1
    conninfo='host=repmgr_node1 user=repmgr_usr dbname=repmgr_db'
    pg_bindir=/path/to/postgres/bin

* `cluster`: common name for the replication cluster; this must be the same on all nodes
* `node`: a unique, abitrary integer identifier
* `name`: a unique, human-readable name
* `conninfo`: a standard conninfo string enabling repmgr to connect to the
   control database; user and name must be the same on all nodes, while other
   parameters such as port may differ. The `host` parameter *must* be a hostname
   resolvable by all nodes on the cluster.
* `pg_bindir`: (optional) location of PostgreSQL binaries, if not in the default $PATH

Note that the configuration file should *not* be stored inside the PostgreSQL
data directory. The configuration file can be specified with the
`-f, --config-file=PATH` option and can have any arbitrary name.`repmgr`
will fail with an error if it does not find the specified file. If no
configuration file is specified, `repmgr` will search for `repmgr.conf`
in the current working directory; if no file is found it will continue
with default values.

The master node must be registered first using `repmgr master register`,
and each standby needs to be registered using `repmgr standby register`
tool; this inserts details about each node into the control database.

Example replication setup
-------------------------

See the QUICKSTART.md file for an annotated example setup.


Failover
--------

To promote a standby to master, on the standby execute e.g.:

    repmgr -f $HOME/repmgr/repmgr.conf --verbose standby promote

`repmgr` will attempt to connect to the current master to verify that it
is not available (if it is, `repmgr` will not promote the standby).

Other standby servers need to be told to follow the new master with e.g.:

    repmgr -f $HOME/repmgr/repmgr.conf --verbose standby follow

See file `autofailover_quick_setup.rst` for details on setting up
automated failover.


Converting a failed master to a standby
---------------------------------------

Often it's desirable to bring a failed master back into replication
as a standby. First, ensure that the master's PostgreSQL server is
no longer running; then use `repmgr standby clone` to re-sync its
data directory with the current master, e.g.:

    repmgr -f $HOME/repmgr/repmgr.conf \
      --force --rsync-only \
      -h node2 -d repmgr_db -U repmgr_usr --verbose \
      standby clone

Here it's essential to use the command line options `--force`, to
ensure `repmgr` will re-use the existing data directory, and
`--rsync-only`, which causes `repmgr` to use `rsync` rather than
`pg_basebackup`, as the latter can only be used to clone a fresh
standby.

The node can then be restarted.

The node will then need to be re-registered with `repmgr`; again
the `--force` option is required to update the existing record:

     repmgr -f $HOME/repmgr/repmgr.conf
       --force \
       standby register



Replication management with repmgrd
-----------------------------------

`repmgrd` is a management and monitoring daemon which runs on standby nodes
and which can automate actions such as failover and updating standbys to
follow the new master.`repmgrd`   can be started simply with e.g.:

    repmgrd -f $HOME/repmgr/repmgr.conf --verbose > $HOME/repmgr/repmgr.log 2>&1

or alternatively:

    repmgrd -f $HOME/repmgr/repmgr.conf --verbose --monitoring-history > $HOME/repmgr/repmgrd.log 2>&1

which will track replication advance or lag on all registerd standbys.

For permanent operation, we recommend using the options `-d/--daemonize` to
detach the `repmgrd` process, and `-p/--pid-file` to write the process PID
to a file.

Example log output (at default log level):

    [2015-03-11 13:15:40] [INFO] checking cluster configuration with schema 'repmgr_test'
    [2015-03-11 13:15:40] [INFO] checking node 2 in cluster 'test'
    [2015-03-11 13:15:40] [INFO] reloading configuration file and updating repmgr tables
    [2015-03-11 13:15:40] [INFO] starting continuous standby node monitoring


Monitoring
----------



Cascading standbys
------------------


Replication slots
-----------------



Reference
---------

### repmgr commands


### repmgr database schema

`repmgr` creates a small schema for its own use in the database specified in
each node's conninfo configuration parameter. This database can in principle
be any database. The schema name is the global `cluster` name prefixed
with `repmgr_`, so for the example setup above the schema name is
`repmgr_test`.

The schema contains two tables:

* `repl_nodes`
  stores information about all registered servers in the cluster
* `repl_monitor`
  stores monitoring information about each node (generated by `repmgrd`

and one view, `repl_status`, which summarizes the latest monitoring information
for each node.

### Error codes

`repmgr` or `repmgrd` will return one of the following error codes on program
exit:

* SUCCESS (0)             Program ran successfully.
* ERR_BAD_CONFIG (1)      Configuration file could not be parsed or was invalid
* ERR_BAD_RSYNC (2)       An rsync call made by the program returned an error
* ERR_NO_RESTART (4)      An attempt to restart a PostgreSQL instance failed
* ERR_DB_CON (6)          Error when trying to connect to a database
* ERR_DB_QUERY (7)        Error while executing a database query
* ERR_PROMOTED (8)        Exiting program because the node has been promoted to master
* ERR_BAD_PASSWORD (9)    Password used to connect to a database was rejected
* ERR_STR_OVERFLOW (10)   String overflow error
* ERR_FAILOVER_FAIL (11)  Error encountered during failover (repmgrd only)
* ERR_BAD_SSH (12)        Error when connecting to remote host via SSH
* ERR_SYS_FAILURE (13)    Error when forking (repmgrd only)
* ERR_BAD_BASEBACKUP (14) Error when executing pg_basebackup


Support and Assistance
----------------------

2ndQuadrant provides 24x7 production support for repmgr, including
configuration assistance, installation verification and training for
running a robust replication cluster.

There is a mailing list/forum to discuss contributions or issues
http://groups.google.com/group/repmgr

#repmgr is registered in freenode IRC

Further information is available at http://www.repmgr.org/

We'd love to hear from you about how you use repmgr. Case studies and
news are always welcome. Send us an email at info@2ndQuadrant.com, or
send a postcard to

repmgr
c/o 2ndQuadrant
7200 The Quorum
Oxford Business Park North
Oxford
OX4 2JZ
United Kingdom

Thanks from the repmgr core team.

Ian Barwick
Jaime Casanova
Abhijit Menon-Sen
Simon Riggs
Cedric Villemain



Further reading
---------------

* http://blog.2ndquadrant.com/announcing-repmgr-2-0/
* http://blog.2ndquadrant.com/managing-useful-clusters-repmgr/
* http://blog.2ndquadrant.com/easier_postgresql_90_clusters/