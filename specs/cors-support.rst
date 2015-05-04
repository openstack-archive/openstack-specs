============
CORS Support
============

The W3C has released a Technical Recommendation (TR) via which an API may
permit a user agent - usually a web browser - to selectively break the
`same-origin policy`_. This permits javascript running in the user agent to
access the API from domains, protocols, and ports that do not match the API
itself. This TR is called Cross Origin Resource Sharing (CORS_).

This specification details how CORS_ is implemented and supported across
OpenStack's services.

Problem description
===================

User Agents (browsers), in order to limit Cross-Site Scripting exploits, do
not permit access to an API that does not match the hostname, protocol, and
port from which the javascript itself is hosted. For example, if a user
agent's javascript is hosted at `https://example.com:443/`, and tries to access
openstack ironic at `https://example.com:6354/`, it would not be permitted to
do so because the ports do not match. This is called the `same-origin policy`_.

The `default ports`_ for most openstack services (excluding horizon) are not
the ports commonly used by user agents to access websites (80, 443). As such,
even if the services were hosted on the same domain and protocol, it would be
impossible for any user agent's application to access these services
directly, as any request would violate the above policy.

The current method of addressing this is to provide an API proxy, currently
part of the horizon project, which is accessible from the same location as
any javascript that might wish to access it. This additional code requires
additional maintenance for both upstream and downstream teams, and is largely
unnecessary.

This specification does *not* presume to require an additional configuration
step for operators for a 'default' install of OpenStack and its user
interface. Horizon currently maintains, and shall continue to maintain, its
own installation requirements.

This specification does *not* presume to set front-end application design
standards- rather it exists to expand the options that front-end teams have,
and allow them to make whatever choice makes the most sense for them.

This specification *does* provide a method by which teams, whether upstream or
downstream, can choose to implement additional user interfaces of their own. An
example use case may be Ironic, which may wish to ship an interface that can
live independently of horizon, for such users who do not want to install
additional components.

Proposed change
===============

All OpenStack API's should implement a common middleware that implements CORS
in a reusable, optional fashion. This middleware must be well documented,
with security concerns highlighted, in order to properly educate the operator
community on their choices.

`CORS Middleware`_ is available in oslo_middleware version 0.3.0. Additional
work would be required to add this middleware to the appropriate services,
and to add the necessary documentation to the docs repository.

Note that improperly implemented CORS_ support is a security concern, and
this should be highlighted in the documentation.

Alternatives
------------

One alternative is to provide a proxy, much like horizon's implementation, or
a well configured Apache mod_proxy. It would require additional documentation
that teaches UI development teams on how to implement and build on it. These
options are already available and well documented, however they do not truly
address the problem of services such as Ironic, which represents its resource
links in a strictly RESTful fashion. In that case, the proxy would have to read
every request and response, and replace all link references to Ironic with
references to itself.

Implementation
==============

Assignee
--------

Primary assignee:
  Michael Krotscheck (krotscheck)

Work Items
----------

- Update Global Requirements to use oslo_middleware version 1.2.0 (complete)
- Propose `CORS Middleware`_ to OpenStack API's that do not already support it.
  This includes, but is not restricted to: Nova, Glance, Neutron, Cinder,
  Keystone, Ceilometer, Heat, Trove, Sahara, and Ironic.
- Propose refactor to use `CORS Middleware`_ to OpenStack API's that already
  support it via other means. This includes, but is not restricted to: Swift.
- Write documentation for CORS configuration.
  - The authoritative content will live in the Cloud Admin Guide.
  - The Security Guide will contain a comment and link to the Cloud Admin Guide.

Dependencies
============

- Depends on oslo_middleware version 1.2.0 (already in Global Requirements)

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

.. _CORS: http://www.w3.org/TR/cors/
.. _`default ports`: http://docs.openstack.org/juno/config-reference/content/firewalls-default-ports.html
.. _`Same-origin Policy`: http://en.wikipedia.org/wiki/Same-origin_policy
.. _`CORS Middleware`: http://docs.openstack.org/developer/oslo.middleware/cors.html
