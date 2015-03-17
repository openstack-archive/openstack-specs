========================================
 No More Downward SQL Schema Migrations
========================================

SQL Migrations are the accepted method for OpenStack projects to ensure
that a given schema is consistent across deployments. Often these
migrations include updating of the underlying data as well as
modifications to the schema. It is inadvisable to perform a downwards
migration in any environment.


Problem description
===================

Best practices state: when reverting post schema migration the
correct course is to restore to a known-good state of the database.

Many migrations in OpenStack include data-manipulation to ensure the data
conforms to the new schema; often these data-migrations are difficult or
impossible to reverse without significant overhead. Performing a downgrade
of the schema with such data manipulation can lead to inconsistent or
broken state. The possibility of bad-states, relatively minimal testing,
and no demand for support renders a downgrade of the schema an unsafe
action.

Proposed change
===============

The proposed change is as follows:

* Eliminate the downgrade option(s) from the CLI tools utilized for
  schema migrations. This is done by rejecting any attempt to migrate
  that would result in a lower schema version number than current

* Document best practices on restoring to a previous schema version
  including the steps to take prior to migrating the schema

* Stop support of migrations with downgrade functions across all
  projects

* Do not add future migrations with downgrade functions


Alternatives
------------

Downgrade paths can continue to be supported.

Implementation
==============

Assignee(s)
-----------

Morgan Fainberg (irc: morganfainberg)
Matt Fischer (irc: mfisch)

Work Items
----------

* Document best practices for restoring to a previous schema version

* Update oslo.db to return appropriate errors when trying to perform
  a schema downgrade

* (all core teams) Do not accept new migrations that include downgrade
  functions

* (all core teams) Projects may drop downgrade functions in all
  current migration scripts

Dependencies
============

No external dependencies.

References
==========

1. `openstack-dev mailing list topic <http://lists.openstack.org/pipermail/
   openstack-dev/2015-January/055586.html>`_

2. `openstack-operators mailing list topic <http://lists.openstack.org/
   pipermail/openstack-operators/2015-January/006082.html>`_

3. `Current migration/rollback documentation <http://docs.openstack.org/
   openstack-ops/content/ops_upgrades-roll-back.html>`_

.. note::

  This work is licensed under a Creative Commons Attribution 3.0 Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode
