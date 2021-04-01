=====================
Load Balancer Support
=====================

:Spec Title: Load Balancer Support
:Spec Version: 1.0.0
:Author: Durran Jordan
:Advisors: Jeff Yemin, Divjot Arora, Andy Schwerin, Cory Mintz
:Status: Accepted
:Type: Standards
:Minimum Server Version: 5.0
:Last Modified: 2021-03-30

.. contents::

--------

**Abstract**
============

This specification defines driver behaviour when connected to mongodb services
through a load balancer.

**META**
========

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in `RFC 2119 <https://www.ietf.org/rfc/rfc2119.txt>`__.

**Specification**
=================


**Terms**
---------

**The PROXY Protocol**
^^^^^^^^^^^^^^^^^^^^^^

`The PROXY protocol <http://www.haproxy.org/download/1.8/doc/proxy-protocol.txt>`__
provides a convenient way to safely transport connection information such
as a client's address across multiple layers of NAT or TCP proxies.
It is designed to require little changes to existing components and to limit the
performance impact caused by the processing of the transported information.

**SDAM**
^^^^^^^^

An abbreviated form of “Server Discovery and Monitoring”, specification defined
in `this document <https://bit.ly/3fsOEmo>`__.

**Service**
^^^^^^^^^^^

Any MongoDB service that can run behind a load balancer, including but not
limited to mongos, Atlas Proxy, Elastic Front End.


**Connection String Options**
-----------------------------

To specify to the driver to operate in load balancing mode, a connection string
option of :code:`loadBalanced=true` MUST be added to the connection string. 

**URI Validation**
^^^^^^^^^^^^^^^^^^

When :code:`loadBalanced=true` is provided in the connection string, the driver
MUST throw an exception in the following cases:

- The connection string contains more than one host/port.
- The connection string contains a replicaSet option.
- The connection string contains a directConnection option.


**DNS Seedlist Discovery**
--------------------------

**Default Connection String Options**
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A default connection string option for :code:`loadBalanced=true` MUST be valid in a
TXT record, and driver validation of the TXT records MUST allow for this option to
also be a part of the connection string.

When :code:`loadBalanced=true` is an option in the TXT record, the driver MUST
validate the connection string as defined in the URI Validation section.

When :code:`loadBalanced=true` is an option in the TXT record, the driver MUST
NOT poll for changes in the SRV record.


**Command Monitoring**
----------------------

When in load balancer mode the driver MUST now include the serverId in the
:code:`CommandStartedEvent`, :code:`CommandSucceededEvent`, and
:code:`CommandFailedEvent`. The driver MAY decide how to expose this information.
Drivers that have a :code:`ConnectionId` object for example, MAY choose to provide a
:code:`serverId` in that object.


**Server Discovery and Monitoring**
-----------------------------------

**Monitoring**
^^^^^^^^^^^^^^

In the case of the driver having the :code:`loadBalanced=true` connection string option
specified, every pooled connection MUST add a loadBalanced field to the hello command
in its `handshake <https://bit.ly/3w9VGCC>`__. The value of the field MUST be true.
When the driver is in load balanced mode, the value of this parameter MUST be true.

A new :code:`TopologyType` MUST be implemented to indicate to the driver that it is
connected to a topology that is deployed behind a load balancer. The
:code:`TopologyType` MUST be named :code:`LoadBalanced`. A new :code:`ServerType` MUST
be implemented to represent a load balancer. The :code:`ServerType` MUST be named
:code:`LoadBalancer`.

When :code:`loadBalanced=true` the topology MUST start in type :code:`LoadBalanced`
and MUST remain as :code:`LoadBalanced` indefinitely. The topology MUST contain 1
:code:`ServerDescription` with a :code:`ServerType` of :code:`LoadBalancer`. The
"address" field of the :code:`ServerDescription` MUST be set to the address field
of the load balancer. All other fields in the :code:`ServerDescription` MUST remain unset.

When :code:`loadBalanced=true`, the driver MUST NOT start a monitoring connection,
however drivers MUST emit the following series of SDAM events:

- :code:`TopologyOpeningEvent` when the topology is created.
- :code:`TopologyDescriptionChangedEvent`. The :code:`previousDescription` field MUST
  have :code:`TopologyType` :code:`Unknown` and no servers. The :code:`newDescription`
  MUST have :code:`TopologyType` :code:`LoadBalanced` and one server with
  :code:`ServerType` :code:`Unknown`.
- :code:`ServerOpeningEvent` when the server representing the load balancer is created.
- :code:`ServerDescription`ChangedEvent. The :code:`previousDescription` MUST have
  :code:`ServerType` :code:`Unknown`. The :code:`newDescription` MUST have
  :code:`ServerType` :code:`LoadBalancer`.
- :code:`TopologyDescriptionChangedEvent`. The :code:`newDescription` MUST have
  :code:`TopologyType` :code:`LoadBalanced` and one server with :code:`ServerType`
  :code:`LoadBalancer`.

Drivers MUST also emit a :code:`ServerClosedEvent` and :code:`TopologyClosedEvent` when
the topology is closed and MUST NOT emit any other events when operating in this mode.

When :code:`loadBalanced=true` and the server’s hello response does not contain a
:code:`serverId` field, the driver MUST throw an exception with the message
*“Driver attempted to initialize in load balancing mode, but the server does not
support this mode.”*

When :code:`loadBalanced=false` or the option is not present, the driver MUST NOT
change any existing behaviour when connected to a non-load balanced service.
If the driver is connected to a service that is configured behind a load balancer,
and the service supports running behind a load balancer, the service MAY return an
error that the driver is not configured to use it properly.

If the :code:`loadBalanced=true` connection string option is not specified, the
driver MUST omit the option in connection handshakes.

*Example:*

Driver connection string contains :code:`loadBalanced=false` or no
:code:`loadBalanced` option:

.. code:: typescript

    { hello: 1 }

Driver connection string contains :code:`loadBalanced=true`:

.. code:: typescript

    { hello: 1, loadBalanced: 1 }

**Data-Bearing Server Type**
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A :code:`ServerType` of :code:`LoadBalancer` MUST be considered a data-bearing server.


**Driver Sessions**
-------------------

**How To Check Whether A Deployment Supports Sessions**
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Having verified in step 1 that the :code:`TopologyDescription` includes at least one
connected server a driver can now determine whether sessions are supported by inspecting
the :code:`TopologyType` and :code:`logicalSessionTimeoutMinutes` property. When the
:code:`TopologyType` is :code:`LoadBalanced`, sessions are always supported.


**Server Selection**
--------------------

Topology type: Load Balanced

A deployment of topology type Load Balanced contains one server of type :code:`LoadBalancer`.

For read and write operations, the single server in the topology MUST always be selected.

For read operations the :code:`mode`, :code:`tag_sets`, and :code:`maxStalenessSeconds`
MUST be passed through to the load balancer but do not affect selection.
See `Passing read preference to mongos <https://bit.ly/2PbyV0B>`__.


**Connection Pooling**
----------------------

When the driver is in load balancing mode and executing any cursor-initiating command, the driver
MUST NOT check the connection back into the connection pool until the cursor is closed.
The driver MUST continue to use the same connection for all getMore and killCursors commands
for a given cursor.

For drivers that implement a connection pool and when using a pinned connection, the driver MUST
emit only 1 :code:`ConnectionCheckOutStartedEvent`, and only 1 :code:`ConnectionCheckedOutEvent`
or :code:`ConnectionCheckOutFailedEvent`. Similarly, the driver MUST only publish 1
:code:`ConnectionCheckedInEvent`.

For single threaded drivers that do not use a connection pool, the driver MUST have only 1
socket connection to the load balancer in load balancing mode.

In the case of network errors for all operations that create a cursor, the driver MUST behave
as defined in the Retryable Reads specification. If a getMore fails with a network error,
drivers MUST unpin the connection that’s pinned to the cursor and return it to the pool
so it does not count against maxPoolSize. Future calls to close() on the cursor MUST NOT
attempt to execute a killCursors command on the server.

When the driver is in load balancing mode, drivers MUST NOT check the connection back into
the connection pool while executing a transaction. The driver MUST continue to use the
same connection to execute all commands for a given transaction with the exception
of ::code:`commitTransaction` and :code:`abortTransaction`, if either one of these
operations fails, the driver MUST unpin connections as follows:

*Drivers MUST unpin a :code:`ClientSession` when a command within a transaction,
including :code:`commitTransaction` and :code:`abortTransaction`, fails with a
:code:`TransientTransactionError`. Transient errors indicate that the transaction
in question has already been aborted or that the pinned mongos is down/unavailable.
Unpinning the session ensures that a subsequent :code:`abortTransaction`
(or :code:`commitTransaction`) does not block waiting on a server that is unreachable.

Additionally, drivers MUST unpin a :code:`ClientSession` when any individual
:code:`commitTransaction` command attempt fails with an :code:`UnknownTransactionCommitResult`
error label. In cases where the :code:`UnknownTransactionCommitResult` causes an automatic
retry attempt, drivers MUST unpin the :code:`ClientSession` before performing server selection
for the retry.

Starting a new transaction on a pinned :code:`ClientSession` MUST unpin the session.
Additionally, any non-transaction operation using a pinned :code:`ClientSession` MUST unpin
the session and the operation MUST perform normal server selection.*

In the case of a network errors in transactions, the driver MUST implement the following cases:

- When a network error occurs on a :code:`commitTransaction`, the connection MUST be closed
  and the operation retried once with a newly checked out connection from the pool.
- When a network error occurs on an :code:`abortTransaction`, the connection MUST be closed
  and the operation retried once with a newly checked out connection from the pool. Any
  subsequent errors from the abortTransaction command MUST be ignored.
- When a network error occurs in any other operation inside a transaction, the connection
  MUST be closed and unpinned and returned to the pool. The driver MUST add a
  :code:`"TransientTransactionError"` label to the error.

The driver connection pool MUST track the purpose for which connections are checked out
in the following 3 categories:

- Connections checked out for cursors
- Connections checked out for transactions
- Connections checked out for operations not falling under the previous 2 categories

When the connection pool’s :code:`maxPoolSize` is reached and the pool times out waiting
for a new connection the :code:`WaitQueueTimeoutError` MUST include a new detailed message,
as well as the published :code:`ConnectionCheckOutFailedEvent`: *“Timeout waiting for
connection from the connection pool. maxPoolSize: n, connections in use by cursors: n,
connections in use by transactions: n, connections in use by other operations: n”*.


**Error Handling**
------------------

When the driver is in load balanced mode and encounters any error described here as a state
change error, the driver MUST NOT make any changes to the :code:`TopologyDescription` or the
:code:`ServerDescription` of the load balancer (i.e. it MUST NOT mark the load balancer as Unknown).
If the error requires the connection pool to be cleared, the driver MUST only clear connections
with the same :code:`serverId` as the connection which errored. The connection pool MUST emit a
:code:`PoolClearedEvent` event with the :code:`serverId` of the connection that errored.
The :code:`PoolClearedEvent` MUST have a new field of :code:`serverId` with type :code:`ObjectId`.

If there is a network error or timeout on the connection before the initial handshake completes,
the driver MUST NOT make any changes to the :code:`TopologyDescription` or mark the server as Unknown.
The driver MUST also close the associated connection. If the network error occurred on the handshake’s
:code:`hello` command, the connection pool MUST NOT be cleared. If the network error occurred after
the handshake’s hello command, the connection POOL must clear close all connections with the matching
:code:`serverId`.


Change Log
==========
