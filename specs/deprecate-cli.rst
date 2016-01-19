=========================
Deprecate Individual CLIs
=========================

https://blueprints.launchpad.net/+spec/deprecate-clis

Historically, each service has offered a CLI application that is included with
the python-\*client that provides administrative and user control over the
service. With the popularity of OpenStack Client and the majority of functions
having been implemented there we should look to officially deprecate the
individual CLI applications.

This does not imply that the entire python-\*client will be deprecated, just
the CLI portion of the library. The python bindings are expected to continue to
work.

Problem description
===================

There is currently no standard workflow for interacting with OpenStack services
on the command line. In the beginning it made sense that there was a nova CLI
for working with nova. As keystone, glance and cinder split out they cloned
novaclient and adapted it to their needs. By the time neutron and the deluge of
big tent services came along there was a clear pattern that each service would
provide a CLI along with their library.

Given the common base and some strong persuasion there is at least a common
subset of parameters and environment variables that are accepted by all CLI
applications. However as new features come up such as YAML based configuration,
keystone's v3 authentication or SSL handling issues these must be addressed in
each project individually and the way these parameters are handled have
drifted or are supported to various levels.

This also creates a horrible user experience for those trying to interact with
the CLI as you have to continually switch between different formatting, command
structures, capabilities and requires a deep knowledge of which service is
responsible for different tasks - a pattern we have been trying to break in
favour of a unified OpenStack interface.

To deal with this the OpenStack client project has now been around for nearly 2
years. It provides a pluggable way to register CLI tasks and a common place to
fix security and usability issues.

Proposed change
===============

Whilst there has been general support for the OpenStack Client project and
support from individual services (it is the primary/supported CLI for keystone
and several newer services) there is no clear direction on whether our users
should use it or continue using the project specific CLIs. Similarly there is
no clear direction on whether developers should contribute new features in
services to the service specific CLI or to OpenStack client.

This blueprint proposes that as a community we ratify that OpenStack Client is
to be the default supported CLI application going forward. This will give
services the direction to deprecate the project CLIs and start pushing their
new features to OpenStack Client. It will give our documentation teams the
direction to start using OpenStack Client as the command for setting up
functionality.

Given that various projects currently have different needs from their CLI I do
not expect that we will be immediately able to deprecate all CLIs. There may be
certain tasks for which there will always need to be a project specific CLI.
The intent of this blueprint initially is not to provide a timeline or force
projects away from their own CLIs. Instead to provide direction to start
deprecating the CLIs for which OpenStack Client already has functionality
compatibility and properly start the process.

Alternatives
------------

We could look at an oslo project that handles the common components of CLI
generation such that we could standardize parameters and handle client creation
in a cross service way. There may be an advantage to doing this anyway as there
will likely always be tools that want to provide a CLI interface to an
OpenStack API that do not belong in OpenStack Client and these should remain
consistent.

Doing nothing is always an option. OpenStack client is steadily gaining
adoption naturally because it can quickly provide new features across a range
of services and so CLI deprecation may happen naturally over time. However
until then we must duplicate the effort of supporting features in multiple
places.

Implementation
==============

As with all OpenStack applications there will have to be a 2 cycle deprecation
period for all these tools.

There are multiple components to this spec and much of the work required will
have to be performed individually in each of the services and documentation
projects. The intention of this spec is to indicate to projects that this is
the direction of the community so we can figure out the project specific
requirements in those groups.


Assignee(s)
-----------

Primary assignee:
  jamielennox
  dtroyer

Work Items
----------

- Add a deprecation warning to clients that have OpenStack Client equivalent
  functionality.
- Update documentation to use OpenStack Client commands instead of project
  specific CLIs (see Documentation Impact).
- Remove CLI components from CLIs after deprecation period complete.

Service Impact
--------------

For most CLI applications we must first start emitting deprecation warnings for
the CLI tools that ship with the deployment libraries.

For core services most functionality is already present and maintained in the
OpenStack Client repository so they would need to ensure feature parity however
they would typically not require any additional code.

As part of core functionality OSC currently supports:

- Nova
- Glance
- Cinder
- Swift
- Neutron
- Keystone

A number of additional projects have already implemented their CLI as an
OpenStack Client plugin. These projects will not be affected. Projects that
have not created plugins would need to implement a plugin that handles the
features they wish to expose via CLI.

Services that currently include an OpenStack Client plugin in their repository
include (but not limited to):

- Zaqar
- Sahara
- Designate

Documentation impact
--------------------

This will be a fairly fundamental change in the way we have communicated with
users to consume openstack and so will require significant documentation
changes.

This will include (but not limited to):

- Install Guides
- Admin Guide
- Ops Guide
- CLI Reference can be deprecated or redirected to OpenStack Client
  documentation.

The OpenStack Client is already in use and deployed as a part of most
installations (it is required for keystone). Therefore changes to documentation
would not be dependant on any work happening in the services. The spec attempts
to ratify that this is the correct approach.

Dependencies
============

There have been many required steps to this goal such as os-client-config,
keystoneauth, cliff, stevedore and the work that has already gone into
OpenStack client. We are now at the point where we can move forward with the
change.

The OpenStack SDK is not listed as a dependency here because it is not
currently a dependency of OpenStack Client. It is intended that when OpenStack
SDK is released it will be consumed by OpenStack Client however that can be
considered an implementation detail.

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Mitaka
     - Introduced

.. note::

  This work is licensed under a Creative Commons Attribution 3.0 Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode
