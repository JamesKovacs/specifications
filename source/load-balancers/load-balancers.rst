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


Change Log
==========
