..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Logging Guidelines
==========================================

https://blueprints.launchpad.net/nova/+spec/log-guidelines

Problem description
===================

The current state of logging both within and between OpenStack
components is inconsistent to the point of being somewhat harmful by
obscuring the current state, function, and real cause of errors in an
OpenStack cloud. A consistent, unified logging format will better
enable cloud administrators to monitor and maintain their
environments.

Before we can address this in OpenStack, we first need to come up with
a set of guidelines that we can get broad agreement on. This is
expected to happen in waves, and this is the first iteration to gather
agreement on.

Proposed change
===============

Definition of Log Levels
------------------------

http://stackoverflow.com/a/2031209
This is a nice writeup about when to use each log level. Here is a
brief description:

- Debug: Shows everything and is likely not suitable for normal
  production operation due to the sheer size of logs generated
- Info: Usually indicates successful service start/stop, versions and
  such non-error related data. This should include largely positive
  units of work that are accomplished (such as starting a compute,
  creating a user, deleting a volume, etc.)
- Audit: REMOVE - (all previous Audit messages should be put as INFO)
- Warning: Indicates that there might be a systemic issue; potential
  predictive failure notice
- Error: An error has occurred and an administrator should research
  the event
- Critical: An error has occurred and the system might be unstable;
  immediately get administrator assistance

We can think of this from an operator perspective the following ways
(Note: we are not specifying operator policy here, just trying to set
tone for developers that aren't familiar with how these messages will
be interpreted):

- Critical : ZOMG! Cluster on FIRE! Call all pagers, wake up
  everyone. This is an unrecoverable error with a service that has or
  probably will lead to service death or massive degredation.
- Error: Serious issue with cloud, administrator should be notified
  immediately via email/pager. On call people expected to respond.
- Warning: Something is not right, should get looked into during the
  next work week. Administrators should be working through eliminating
  warnings as part of normal work.
- Info: normal status messages showing measureable units of positive
  work passing through under normal functioning of the system. Should
  not be so verbose as to overwhelm real signal with noise. Should not
  be continuous "I'm alive!" messages.
- Debug: developer logging level, only enable if you are interested in
  reading through a ton of additional information about what is going on.

Proposed Changes From Status Quo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Deprecate and remove AUDIT level

Rationale, AUDIT is confusing, and people use it for entirely the
wrong purposes. The origin of AUDIT was a NASA specific requirement
which is not longer really relevant to the current code.

Information that was previously being emitted at AUDIT should instead
be sent as notifications to a notification queue. *Note: Notification formats
and frequency are beyond the scope of this spec.*

Overall Logging Rules
---------------------
The following principles should apply to all messages

Log messages at Info and above should be a "unit of work"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The Info log level is defined as: "normal status messages showing
measureable units of positive work passing through under normal
functioning of the system."

A measurable unit of work should be describable by a short sentence
fragment, in the past tense with a noun and a verb of something
significant.

Examples::

  Instance spawned

  Instance destroyed

  Volume attached

  Image failed to copy

Words like "started", "finished", or any verb ending in "ing" are
flags for non unit of work messages.

Debugging start / end messages
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

At the Debug log level it is often extremely important to flag the
beginning and ending of actions to track the progression of flows
(which might error out before the unit of work is completed).

This should be made clear by there being a "starting" message with
some indication of completion for that starting point.

In a real OpenStack environment lots of things are happening in
parallel. There are multiple workers per services, multiple instances
of services in the cloud.

Examples of Good and Bad uses of Info
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Below are some examples of good and bad users of info. In the Good
examples we can see the 'noun / verb' fragment for a unit of work
(successfully is probably superfluous and could be removed).

In the bad examples we see trace level thinking put into INFO and
above messages.

**Good**

::
   2014-01-26 15:36:10.597 28297 INFO nova.virt.libvirt.driver [-]
   [instance: b1b8e5c7-12f0-4092-84f6-297fe7642070] Instance spawned
   successfully.

   2014-01-26 15:36:14.307 28297 INFO nova.virt.libvirt.driver [-]
   [instance: b1b8e5c7-12f0-4092-84f6-297fe7642070] Instance destroyed
   successfully.

**Bad**

::
   2014-01-26 15:36:11.198 INFO nova.virt.libvirt.driver
   [req-ded67509-1e5d-4fb2-a0e2-92932bba9271
   FixedIPsNegativeTestXml-1426989627 FixedIPsNegativeTestXml-38506689]
   [instance: fd027464-6e15-4f5d-8b1f-c389bdb8772a] Creating image

   2014-01-26 15:36:11.525 INFO nova.virt.libvirt.driver
   [req-ded67509-1e5d-4fb2-a0e2-92932bba9271
   FixedIPsNegativeTestXml-1426989627 FixedIPsNegativeTestXml-38506689]
   [instance: fd027464-6e15-4f5d-8b1f-c389bdb8772a] Using config drive

   2014-01-26 15:36:12.326 AUDIT nova.compute.manager
   [req-714315e2-6318-4005-8f8f-05d7796ff45d FixedIPsTestXml-911165017
   FixedIPsTestXml-1315774890] [instance:
   b1b8e5c7-12f0-4092-84f6-297fe7642070] Terminating instance

   2014-01-26 15:36:12.570 INFO nova.virt.libvirt.driver
   [req-ded67509-1e5d-4fb2-a0e2-92932bba9271
   FixedIPsNegativeTestXml-1426989627 FixedIPsNegativeTestXml-38506689]
   [instance: fd027464-6e15-4f5d-8b1f-c389bdb8772a] Creating config
   drive at
   /opt/stack/data/nova/instances/fd027464-6e15-4f5d-8b1f
   -c389bdb8772a/disk.config

This is mostly an overshare issue. At Info these are stages that don't
really need to be fully communicated.

Messages shouldn't need a secret decoder ring
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Bad**

::
   2014-01-26 15:36:14.256 28297 INFO nova.compute.manager [-]
   Lifecycle event 1 on VM b1b8e5c7-12f0-4092-84f6-297fe7642070

General rule, when using constants or enums ensure they are translated
back to user strings prior to being sent to the user.

Specific Event Types
--------------------

In addition to the above guidelines very specific additional
requirements exist.

WSGI requests
~~~~~~~~~~~~~

Should be:

- Logged at **INFO** level
- Logged exactly once per request
- Include enough information to know what the request was

The last point is notable, because some POST API requests don't
include enough information in the URL alone to determine what the
API did. For instance, Nova Server Actions (where POST includes a
method name).

Rationale: Operators should be able to easily see what API requests
their users are making in their cloud to understand the usage patterns
of their users with their cloud.

Operator Deprecation Warnings
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Should be:

- Logged at **WARN** level
- Logged exactly once per service start (not on every request through
  code)
- Include directions on what to do to migrate from the deprecated
  state

Rationale: Operators need to know that some aspect of their cloud
configuration is now deprecated, and will require changes in the
future. And they need enough of a bread crumb trail to figure out how
to do that.

REST API Deprecation Warnings
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Should be:

- **Not** logged any higher than DEBUG (these are not operator facing
  messages)
- Logged no more than once per REST API usage / tenant. Definitely
  not on *every* REST API call.

Rationale: The users of the REST API don't have access to the system
logs. Therefore logging at a WARNING level is telling the wrong people
about the fact that they are using a deprecated API.

Deprecation of User facing API should be communicated via User facing
mechanisms, being API change notes associated with new API versions.

Stacktraces in Logs
~~~~~~~~~~~~~~~~~~~

Should be:

- **exceptional** events, for unforeseeable circumstance that is not
  yet recoverable by the system.
- Logged at ERROR level
- Considered high priority bugs to be addressed by the development
  team.

Rationale: The current behavior of OpenStack is extremely stack trace
happy. Many existing stack traces in the logs are considered
*normal*. This dramatically increases the time to find the root cause
of real issues in OpenStack.


Logging by non-OpenStack Components
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

OpenStack uses a ton of libraries, which have their own definitions of
logging. This causes a lot of extraneous information in normal logs by
wildly different definitions of those libraries.

As such, all 3rd party libraries should have their logging levels
adjusted so only real errors are logged.

Currently proposed settings for 3rd party libraries:

- amqp=WARN
- boto=WARN
- qpid=WARN
- sqlalchemy=WARN
- suds=INFO
- iso8601=WARN
- requests.packages.urllib3.connectionpool=WARN
- urllib3.connectionpool=WARN



Alternatives
------------

Continue to have terribly confusing logs

Data model impact
-----------------

NA

REST API impact
---------------

NA

Security impact
---------------

NA

Notifications impact
--------------------

NA

Other end user impact
---------------------

NA

Performance Impact
------------------

NA

Other deployer impact
---------------------

Should provide a much more standard way to determine what's going on
in the system.

Developer impact
----------------

Developers will need to be cognizant of these guidelines in creating
new code or reviewing code.

Implementation
==============

Assignee(s)
-----------

Assignee is for moving these guidelines through the review process to
something that we all agree on. The expectation is that these become
review criteria that we can reference and are implemented by a large
number of people. Once approved, will also drive collecting volunteers
to help fix in multiple projects.

Primary assignee:
  Sean Dague <sean@dague.net>

Work Items
----------
Using this section to highlight things we need to decide that aren't
settled as of yet.

Proposed changes with general consensus

- Drop AUDIT log level, move all AUDIT message to either an INFO log
  message or a ``notification``.
- Begin adjusting log levels within projects to match the severity
  guidelines.


Dependencies
============

NA

Testing
=======

See tests provided by
https://blueprints.launchpad.net/nova/+spec/clean-logs

Documentation Impact
====================

Once agreed upon this should form a more permanent document on logging
specifications.

References
==========

- Security Log Guidelines -
  https://wiki.openstack.org/wiki/Security/Guidelines/logging_guidelines
- Wiki page for basic logging standards proposal developed early in
  Icehouse - https://wiki.openstack.org/wiki/LoggingStandards
