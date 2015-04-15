..

=============================
 Supported Messaging Drivers
=============================

We need to define a policy on messaging in general.

Problem description
===================

OpenStack has gravitated toward using RabbitMQ for message passing. There
are numerous excellent alternatives available as backends, including
QPID and 0mq, but only RabbitMQ has received attention directly in the
community at large. As a result, more and more users of these backends
are switching to RabbitMQ, and this leaves QPID code lying around that
is not well tested and not well supported by the community. Said code
will continue to be a burden, and should be removed if it is doing more
harm than good.

There is also anecdotal evidence that users are using the zmq driver
to achieve high scale with fixes, but the zmq driver is not well
tested either, which may actually be worse than not having it at all,
as now anecdotes are passed from user to user but results will be very
different for users who stick to upstream code.

Proposed change
===============

RabbitMQ may not be sufficient for the entire community as the community
grows. Pluggability is still something we should maintain, but we should
have a very high standard for drivers that are shipped and documented
as being supported.

We will define a very clear policy as to the requirements for drivers
to be carried in oslo.messaging and thus supported by the OpenStack
community as a whole. We will deprecate any drivers that do not meet
the requirements, and announce said deprecations in any appropriate
channels to give users time to signal their needs. Deprecation will last
for two release cycles before removing the code. We will also review and
update documentation to annotate which drivers are supported and which
are deprecated given these policies

Policy
------

Testing
~~~~~~~

* Must have unit and/or functional test coverage of at least 60% as
  reported by coverage report. Unit tests must be run for all versions
  of python oslo.messaging currently gates on.

* Must have integration testing including at least 3 popular oslo.messaging
  dependents, preferrably at the minimum a devstack-gate job with Nova,
  Cinder, and Neutron.

* All testing above must be voting in the gate of oslo.messaging.

Documentation
~~~~~~~~~~~~~

* Must have a reasonable amount of documentation including documentation
  in the official OpenStack deployment guide.

Support
~~~~~~~

* Must have at least two individuals from the community commited to
  triaging and fixing bugs, and responding to test failures in a timely
  manner.

Prospective Drivers
~~~~~~~~~~~~~~~~~~~

* Drivers that intend to meet the requirements above, but that do not yet
  meet them will be given one full release cycle, or 6 months, whichever
  is longer, to comply before being marked for deprecation. Their use,
  however, will not be supported by the community. This will prevent a
  chicken and egg problem for new drivers.

Alternatives
------------

We could remove pluggability from oslo.messaging entirely, and just
ship RabbitMQ drivers. This option would alienate users who have private
drivers, and would also force users of the zmq driver who are trying to
fix it to abandon those efforts and try to scale with RabbitMQ.

We could also go even further, remove the pluggability, and improve the
kombu library enough to support all use cases of oslo.messaging. That is
beyond the scope of this document though.

Implementation
==============

Assignee(s)
-----------

Clint "SpamapS" Byrum

Work Items
----------

- Record policy in oslo developer documentation
- Announce policy to mailing lists
- Mark non-compliant drivers as deprecated
- Update configuration guides to note non-supported drivers
- After deprecation period, remove non-compliant drivers from code and docs

Dependencies
============

N/A

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Liberty
     - Introduced

.. note::

  This work is licensed under a Creative Commons Attribution 3.0 Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode
