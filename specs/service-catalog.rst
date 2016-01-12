===============================
Service Catalog Standardization
===============================

https://blueprints.launchpad.net/keystone/+spec/service-catalog-standards

The service catalog offers cloud consumers an initial view to all the available
services in an OpenStack cloud along with additional information about
regions, API versions, and projects available. This catalog is to help make it
easy to efficiently find information about the services, such as how to
configure communications between services. But the catalog is also terribly
underused, under documented, and inconsistently configured among most services.
By standardizing the service catalog we can provide a better user experience
with OpenStack.

Problem description
===================

The service catalog might be the first interaction a user has with an OpenStack
cloud to understand what the cloud offers in services and resources. That
interaction can be confusing, inconsistent between cloud providers, and contain
names and numbers that are mysterious and need decoding.

Providers making a service catalog might not think about consumers who see
multiple service catalogs in a single week.

The API Working Group did some initial fact finding about the varieties of
service catalogs available, and discovered just how varied the catalog can be.
See
https://wiki.openstack.org/wiki/API_Working_Group/Current_Design/Service_Catalog.

As an example of the inconsistency, cloud providers have filled in the
"name" object as all three: "nova", "cloudServersOpenStack" and "Compute".

Such a diverse service catalog means that services don't depend on it
being consistent, SDK devs don't completely understand it, and it
requires applications to encode cloud-specific behavior.

Here are some concrete examples of information that must be encoded because it
cannot be determined from the service catalog.

For example, a `nova.conf` file has to indicate exact URLs for many API
endpoints::

   [glance]
   api_servers = http://127.0.0.1:9292
   [neutron]
   url = http://127.0.0.1:9696

Ideally, rather than hardcoding these URL/port values in configuration
files, the service catalog could provide discoverability for those.

Another example, the ``catalog_info = volume:cinder:publicURL`` in
`nova.conf` is a configuration setting to set the info to match when
looking for cinder in the service catalog. Format is separated values
of the form::

    <service_type>:<service_name>:<endpoint_type>.

There's also an ``endpoint_template`` `nova.conf` variable that
overrides the service catalog lookup with template for cinder
endpoint, such as ``http://localhost:8776/v1/%(project_id)s``.

While we are working through many of these issues, we are ensuring that
projects understand that user experience includes consistency, discoverability,
and simplicity as design tenets for service catalog incremental improvements.

A Vision of the Ideal Service Catalog
=====================================

The following is a vision of where we want to get to with the service
catalog in OpenStack.

1. Querying type='volume' in any service catalog on any cloud returns
   an unversioned URL for that service. This is a contract we can
   depend on.

2. All OpenStack server components find the operational urls for other
   OpenStack services with the catalog.

3. The service types used in the catalog, such as ``volume`` or ``compute``,
   defined in server components. This definition ensures that standardization
   propagates, as clouds will not work if their service catalog is not
   defined correctly.

4. There are Tempest tests for standard service catalog, and it's a
   DefCore requirement that standard service catalog entries are
   defined.


Proposed change
===============

What we want to solve for:

- Standard required naming for endpoints (versioned vs. unversioned,
  contains project ID vs. no project ID).

    * We want unversioned endpoints so that the user can get
      information about multiple available versions in a given cloud.
    * We do not want project ID, account ID, or tenant ID as part of
      the resource URI for an OpenStack API endpoint.
    * Standard naming means all consumers, including other OpenStack
      services, can trust what the value of type='volume' will be.

- List of changes needed in existing clouds/products to comply with this.

    * We want DevStack to follow these standards as the best practice example.
    * We want to use JSON Schema to define the API for the service catalog
      to ensure understanding and compliance.
    * JSON Schema must allow for "extra" data so that we can continue with
      name, and vendor-specific "Extra" things during the transition(s).
    * Known types such as `service_type` can be documented in `projects.yaml`
      in the `openstack/governance` git repository.

- List of changes in OpenStack projects that would rely on this standard, thus
  making sure we've got it right.

- Published guidelines we recommend that DefCore requires of cloud provider's
  service catalogs going forward. These guidelines can be created in the API
  Working Group set of guidelines.

- Documentation for all new projects to comply with the service catalog
  standards defined by the guidelines.

Top difficulties with the service catalog for SDK devs are currently:

- Name and type are meaningless, misunderstood, and poorly documented.

- Regions are not consistently named or used. The way regions are structured
  are a pain for SDKs, because it requires a lot of traversing. Encourage
  clear provider documentation and guidance for this naming.

- The versions in URLs are inconsistent (see what we want to solve for above).

- The tying between auth and getting a service catalog seems unnecessary. See
  roles example above. A user should be able to get a list of all the services
  and endpoints in a single, preferably unauthenticated, call.

Documentation can improve some of the difficulties. Standards and guidelines
should be published from within the Cloud Admin Guide, the Installation Guides,
and the Identity service (keystone) developer documentation.

The list of changes is gathered here:

- Ensure each service's API has a version request (current standard is a GET
  call to /). However, keystoneauths's session can't use that to discover
  versions because the URL returned by the Identity service for another
  configured service is the versioned endpoint. The version is embedded in the
  URL. We should have the Identity service discover version number with each
  services' API itself.

- Remove ``project_id`` template from endpoints, acknowledging that future clients
  will have to account for this change.

- Ensure DevStack examples are consistent and can be used as an exemplary
  best practice.

- Ensure Tempest works with new catalog.

- Write a tempest test that uses JSON Schema for the service catalog.

- Provide the standard project and service names in the governance repository
  through the `projects.yaml` file. However, enable flexibility in the "name"
  for providers to offer multiple services.

- Cause project's interactions with the service catalog to be standard so that
  for example, the nova project does not need three configuration variables to
  specify how nova can interact with the cinder service catalog entries.

- Ensure that the publicURL, adminURL, and internalURL have known use cases.
  Work with the operator community to understand whether those can be
  consolidated when presenting the catalog to an end user.

Alternatives
------------

What happens currently is DevStack's configuration becomes a de facto standard
for endpoint URL naming, which then indicates both the name and type standard.

Implementation
==============


Assignee(s)
-----------

  Anne Gentle annegentle
  Augustina Ragwitz
  Sean Dague sdague
  Dolph Matthews dolphm

Work Items
----------

Create a guideline in the API Working Group repository for service
types, names, endpoint URLs, and configuration for cloud providers
creating service catalog entries.

Create a JSON Schema for the service catalog, to be stored as a
tempest test, so that the refstack repo can make use of it. Tempest
tests can check for valid entries. So the Identity project won't
enforce the list, rather a test in Tempest can enforce for
interoperability. The test will check each entry based on JSON Schema,
such as:

- existence of service_type: required
- value type of service_type: string (reference for value from
  governance/projects.yaml file)
- extra data: acceptable because of the need for transition for providers

DevStack should be the reference implementation for best practices in
service catalog entries.

Create a conceptual topic about the service catalog using
http://dolphm.com/openstack-keystone-service-catalog/ as a starting
point.



Dependencies
============

.. Summit session:
   https://libertydesignsummit.sched.org/event/194b2589eca19956cb88ada45e985e29

Additional Reading
==================

http://docs.openstack.org/developer/keystone/configuration.html?highlight=catalog#service-catalog

http://docs.openstack.org/juno/config-reference/content/list-of-compute-config-options.html

http://dolphm.com/openstack-keystone-service-catalog/

https://etherpad.openstack.org/p/service-catalog-cross-project-vancouver

https://wiki.openstack.org/wiki/API_Working_Group/Current_Design/Entry_Points

https://wiki.openstack.org/wiki/API_Working_Group/Current_Design/Service_Catalog

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
