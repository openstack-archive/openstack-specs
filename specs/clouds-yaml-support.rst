================================
 clouds.yaml support in clients
================================

`clouds.yaml` is a config file for the facilitation of consuming multiple
OpenStack clouds in a reasonable way. It is processed by the `os-client-config`
library, and is currently supported by `python-openstackclient`, `shade`,
`nodepool` and Ansible 2.0.

It should be supported across the board in our client utilities.

Problem description
===================

One of the goals of our efforts in OpenStack is interoperability between
clouds. Although there are several reasons that this is important, one of
them is to allow consumers to spread their workloads across multiple clouds.

Once a user has more than one cloud, dealing with credentials for the tasks of
selecting a specific cloud to operate on, or of performing actions across all
available clouds, becomes important.

Because the only auth information mechanism the OpenStack project has provided
so far, `openrc`, is targetted towards a single cloud, projects have have
attempted to deal with the problem in a myriad of different ways that do not
carry over to each other.

Although `python-openstackclient` supports `clouds.yaml` cloud definitions,
there are still some functions not yet exposed in `python-openstackclient` and
cloud users sometimes have to fall back to the legacy client utilities. That
means that even though `python-openstackclient` allows the user to manage
their clouds simply, the problem of dealing with piles of `openrc` files
remains, making it a net loss complexity-wise.

Proposed change
===============

Each of the python client utilities that exist should use `os-client-config` to
process their input parameters. New projects that do not yet have a CLI
utility should use `python-openstackclient` instead, and should not write new
CLI utilities.

An example of migrating an existing utility to `os-client-config` can be seen
in https://review.openstack.org/#/c/236325/ which adds the support to
`python-neutronclient`. Since all of those need to migrate to `keystoneauth1`
anyway, and since `os-client-config` is well integrated with `keystoneauth1`
it makes sense to do it as a single change.

This change will also add `OS_CLOUD` and `--os-cloud` as options supported
everywhere for selecting a named cloud from a collection of configured
cloud configurations.

Horizon should add a 'Download clouds.yaml' link where the 'Download openrc'
link is.

Reach out to the ecosystem of client utilities and libraries to suggest adding
support for consuming `clouds.yaml` files.
`gophercloud` https://github.com/rackspace/gophercloud/issues/487 has been
contacted already, but at least `libcloud`, `fog`, `jclouds` - or any other
framework that is in the Getting Started guide should at least be contacted
about adding support.

It should be pointed out that `os-client-config` does not require the use of
or existence of `clouds.yaml` and the traditional `openrc` environment
variables will continue to work as always.

http://inaugust.com/posts/multi-cloud-with-python-openstackclient.html is
a walkthrough on what life looks like in a world of `os-client-config` and
`python-openstackclient`.

Alternatives
------------

Using `envdir` has been suggested and is a good fit for direct user
consumption. However, some calling environments like `tox` and `ansible` make
communicating information from the calling context to the execution context
via environment variables clunkier than one would like. `python-neutronclient`
for instance has a `functional-creds.conf` that it writes out to avoid the
problems with environment variables and `tox`.

Just focus on `python-openstackclient`. While this is a wonderful future, it's
still the future. Adding `clouds.yaml` support to the existing clients gets us
a stronger bridge to the future state of everyone using
`python-openstackclient` for everything.

Use `oslo.config` as the basis of credentials configuration instead of yaml.
This was originally considered when `os-client-config` was being written, but
due to the advent of keystone auth plugins, it becomes important for some
use cases to have nested data structures, which is not particularly clean
to express in ini format.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  mordred

mordred is happy to do all of the work - but is also not territorial and if
elements of the work magically get done by happy friends, the world would be
a lovely place.

Work Items
----------

Not exhaustive, but should be close. Many projects provide openstackclient
extensions rather than their own client, so are covered already.

* Add support to python-barbicanclient
* Add support to python-ceilometerclient
* Add support to python-cinderclient
* Add support to python-designateclient
* Add support to python-glanceclient
* Add support to python-heatclient
* Add support to python-ironicclient
* Add support to python-keystoneclient
* Add support to python-magnumclient
* Add support to python-manilaclient
* Add support to python-neutronclient
* Add support to python-novaclient
* Add support to python-saharaclient
* Add support to python-swiftclient
* Add download link to horizon

Dependencies
============

None

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
