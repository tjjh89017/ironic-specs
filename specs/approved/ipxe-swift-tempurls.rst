..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
iPXE to use Swift Temporary URLs
================================

https://bugs.launchpad.net/ironic/+bug/1526404

This adds support for generating Swift temporary URLs for the
deploy ramdisk and kernel when booting with iPXE.

Problem description
===================

Currently the iPXE driver requires an external HTTP server to serve
the deploy ramdisk and kernel. When used with Glance, the
``ironic-conductor`` fetches the images from it and place them under the
HTTP root directory, and if a rebalance happens in the hash right the
new ``ironic-conductor`` taking over the node have to do the same thing,
fetch the images and cache it locally to be able to manage that node.

Having an external HTTP server should not be required when Glance is used
with a Swift backend, with Swift we can generate temporary URLs that can
be passed to iPXE to download the images without requiring credentials.

Proposed change
===============

The proposed implementation consists in having the iPXE driver to create
a Swift tempurl for the deploy ramdisk and kernel that the
node will boot as part of the config generation.

This also proposes adding a boolean configuration option under
the ``pxe`` group called ``ipxe_use_swift``. If True this will tell iPXE to
not cache the images in the disk and generate the Swift tempurl for the
ramdisk and kernel, if False, iPXE will continue to cache the images
under the HTTP root directory. Defaults to False.

Note that in order to keep compatibility with Nova behavior,
kernel/ramdisk of the user image still have to be cached in case
``netboot`` is required. Doing otherwise will make it impossible for user
to reboot the instance from within when tempurls have expired or the image
is deleted from Glance altogether.

Alternatives
------------

Continue to use an external HTTP server and caching the images on
the disk.

Data model impact
-----------------

None

State Machine Impact
--------------------

None

REST API impact
---------------

None

Client (CLI) impact
-------------------

None

RPC API impact
--------------

None

Driver API impact
-----------------

None

Nova driver impact
------------------

None

Ramdisk impact
--------------

N/A

.. NOTE: This section was not present at the time this spec was approved.

Security impact
---------------

There's a positive security impact because the Swift temporary URLs does
have an expiration time and the images in the external HTTP server will
be available until the instance is destroyed.

Other end user impact
---------------------

None

Scalability impact
------------------

There is a scaling benefit to download directly from Swift since a Swift
cluster can be scaled horizontally by adding new nodes.

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  lucasagomes <lucasagomes@gmail.com>

Other contributors:
  pshchelo <shchelokovskyy@gmail.com>

Work Items
----------

* Add the new ``ipxe_use_swift`` configuration option under the ``pxe`` group.

* Get the PXE driver to generate the Swift temporary URLs as part of
  the configuration generation when ``ipxe_use_swift`` is True.

* Skip caching the image on the disk when ``ipxe_use_swift`` is True.

Dependencies
============

None

Testing
=======

Unittests will be added.

Upgrades and Backwards Compatibility
====================================

None

Documentation Impact
====================

The iPXE documentation will be updated to reflect the changes made by
this spec.

References
==========

.. [#] http://docs.openstack.org/kilo/config-reference/content/object-storage-tempurl.html
