..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
BitTorrent-based image provisioning
===================================

https://bugs.launchpad.net/ironic/+bug/1576661

The proposed feature enables the provisioning of instances using the
BitTorrent protocol. Peer-to-peer sharing can reduce load on image
storage and network when simultaneously provisioning multiple instances
with the same image.

Problem description
===================

Ironic allows the deployment of images by downloading them from
the centralized image store. This approach becomes a scalability
bottleneck when provisioning large numbers of instances simultaneously:

* In research computing, an established use case in infrastructure
  management is the simultaneous deployment of large numbers of servers with
  a single system image. The Rocks Cluster Installer [2]_ uses BitTorrent in
  this use case to good effect. The advantage of BitTorrent is that the
  capacity to serve image data grows with the demand for that data.

* Provisioning of an undercloud as part of an OpenStack deployment.
  TripleO uses Nova and Ironic to provision the undercloud and Kolla can use
  Bifrost and Ironic to bootstrap nodes.

As a result, Ironic's current methods for deployment do not scale well
to support large-scale concurrent provisioning. Substantial control plane
infrastructure is required to support concurrent provisioning at scale -
and the requirement grows linearly with the size of the deployment.

Proposed change
===============

Simultaneously provisioning the same image to multiple instances can be more
efficient by sharing the image data between deploy agents.

This change proposes to add support for the BitTorrent protocol.
The proposed design illustrated as a sequence diagram::

 Ironic           Ironic        Glance    Glare     Ironic     Torrent  Ironic
  API           Conductor    (image     (artifact    node1     Tracker   node2
   |                |          storage)     store)    |          |           |
   |---do_deploy--->|              |        |         |          |           |
   |                |     ...      |        |         |          |           |
   |                |--get image-->|        |         |          |           |
   |                |   info       |        |         |          |           |
   |                |--get torrent--------->|         |          |           |
   |                |    file      |        |         |          |           |
   |add webseed url |              |        |         |          |           |
   | to torrent  ---|              |        |         |          |           |
   |  metadata  |   |              |        |         |          |           |
   |             -->|              |        |         |          |           |
   |      ...       |     ...      |        |         |          |           |
   |                |  continue deploy, pass image    |          |           |
   |                |  info with torrent metadata     |          |           |
   |                |-------------------------------->|          |           |
   |                |------------------------------------------------------->|
   |                |              |        |         |    client announce   |
   |                |              |        |         |<-------->|<--------->|
   |                |              |        |         |          |           |
   |                |              |--download image->|          |           |
   |                |              |---------------download image----------->|
   |                |              |                  |<-share image pieces->|

This change proposes to use BitTorrent webseed_ feature to reduce Ironic
Conductor load. This simplifies the role of the conductor, avoiding
the need to seed the images. The images can be seeded directly from
object storage via HTTP. The HTTP server must support the HTTP/1.1
Accept-Ranges extension in order to perform this role (Swift_ supports this).

The implementation of BitTorrent provisioning could be considered as
3 independent tasks:

* A BitTorrent client to download and share images.

  A BitTorrent client should be installed on the agent ramdisk (e.g Aria2_,
  it is a lightweight multi-protocol utility, which supports BitTorrent
  and Metalink).

  Every BitTorrent client, the node being deployed, will announce to the
  tracker a number times during the course of a download to update its peers
  information.

* A mechanism for discovering peers (tracker, DHT_, LPD_).

  In current spec it's proposed to use the BitTorrent tracker. For example
  qbittorrent (qbittorent-nox_).
  It's possible to avoid tracker requirement by using BitTorrent extensions
  like LPD_ and DHT_. But this approach has some drawbacks.

  LPD_ can suffer from race conditions, when peers start sharing at similar
  times. The peers - the nodes being deployed - have a short lifetime,
  limited to the duration of deployment. Due to the relatively large discovery
  interval (5 minutes) recommended by the `LPD specification`_, this issue may
  persist for the entire duration of deployment. So BitTorrent clients don't
  allow to configure discovery interval due `LPD specification`_
  recommendation. Also sending LSD announce very often on large networks may
  cause multicast storms.
  It's also possible that multicast could be disabled on deployment network
  due security reasons.

  DHT_ requires at least one node (a node is a client/server listening on
  a UDP port implementing the distributed hash table protocol, and keep DHT
  table) to be permanently available. It could be running on the conductor
  host. The protocol is based on Kademila_ and is implemented over UDP. So
  this makes it no easier to manage than a tracker.

  A BitTorrent tracker is a simple web service which responds to requests
  from BitTorrent clients. The requests include metrics from clients that
  help the tracker keep overall statistics about the torrent. The response
  includes a peer list that helps the client participate in the torrent swarm.

  Multiple BitTorrent trackers are required to support Ironic deployment
  with High Availability configuration.

* A mechanism for generating and managing torrent files.

  Torrent files contain the URL of the torrent tracker, the block size of
  the file(s) and the SHA-1 checksums for each block of the file(s) being
  transferred. Linux tools for creating torrent files include ctorrent_,
  transmission_, and mktorrent_.

  So the storage for torrent files is required, Artifact Repository (Glare_)
  would be best choice here. Operators will create torrent file using standard
  tools and upload it to Glare_ and provide link to it via image property
  ``torrent_file=<glare id>'`` (it's possible to support glance UUIDs and http
  URLs as well, URI schemes could be used to distinguish torrent file source,
  like ``glare://uuid``, ``glance://uuid``, ``http://path/``). Later on the
  Ironic Conductor fetches image torrent metadata and orders the node being
  deployed to download the image over the BitTorrent protocol.
  In case standalone mode, torrent file location could be provided via
  ``instance_info/image_torrent_file`` in addition to image source
  ``instance_info/image_source``.

Based on the torrent file, the tracker and webseed_ info (image location url
either Swift temp url or http(s) link), provided by the Ironic Conductor,
the nodes being deployed generate torrent metadata. When another node boots,
it uses this metadata to retrieve the torrent file for the BitTorrent client.
The deploying node client in turn announces [1]_ itself back to the tracker
for other nodes to reach out.

The deploying node torrent client requires an endpoint URL of the tracker.
This information will be provided via a new configuration option
``torrent_trackers``. This option contains multiple trackers. Multiple URLs
can be tracked to achieve redundancy of the torrent swarm::

    torrent_trackers = http://<host1>:<port1>/announce,
                       http://<host2>:<port1>/announce

The redundancy requirement narrows the client choice down to those supporting
the multi-tracker_ metadata extension.

To use BitTorrent as the default provisioning mechanism, the Glance image
property ``torrent_file`` should be specified, and Ironic configuration
option ``[deploy]enable_torrent_provisioning`` should be enabled.
If ``[deploy]enable_torrent_provisioning`` set to ``False``,
``torrent_file`` will be ignored, and image will be downloaded via HTTP.
In case node's driver doesn't support BitTorrent provisioning, BitTorrent
options will be ignored as well.

One of the shortcomings of the proposed approach is the low peer share ratio
due to node (peer) short lifespan. Having finished downloading an image, the
node doesn't continue seeding the image. Therefore the sharing ratio in a
swarm cannot be enforced without delaying a client BM instance boot and has
a random distribution over time.
But this doesn't have big impact on the overall deployment performance,
the difference between image downloading time and API request delay is
insignificant.

The spec doesn't consider PEX_ extension, as it could not be used on its own
to introduce a new peer to a swarm. PEX_ allows peers in a swarm to exchange
information about the swarm directly without asking the tracker or the DHT_.
So it could be used for future enhancements.

The final workflow for operators looks like:

* configure Ironic to use BitTorrent provisioning
* create the deployment image and its torrent file
* upload the torrent file to Glare (Glance or HTTP server)
* upload the image to Glance and set the ``torrent_file`` property
* deploy instances via BitTorrent protocol

Alternatives
------------

* Introduce seeding functionality in Ironic Conductor. Downloading would
  be a little faster in the beginning, if the image is already cached: in
  this case all conductors will seed the image immediately and clients
  will not need to download it via HTTP. But the overall deployment
  performance enhancement is insignificant, because all load would be
  moved to conductor from storage, and at some point it could become a
  performance issue.
  The disadvantage of this approach is that the conductor does the bulk work
  of seeding the images, which can be avoided, as the image can be seeded
  directly from the object store via a temporary URL. Also the conductors
  would need some image retention policy.

* Using DHT_ and LPD_ instead of tracker, as mentioned in proposed section.

* Create a new service which will perform torrent provisioning, like Glance
  in previous suggestion.

* Using magnet_ links. But we still need to get image metadata somewhere, and
  at start of deploying, we don't have seeds at all, as we are using web seed,
  so there is no benefit of it.

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

"ironic" CLI
~~~~~~~~~~~~

None

"openstack baremetal" CLI
~~~~~~~~~~~~~~~~~~~~~~~~~

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

The agent should be able to get the torrent metadata and provide
it to the BitTorrent client.

Also some BitTorrent client should be installed on the ramdisk.

Security impact
---------------

There will be no changes in image fetching mechanism, only one more
protocol will be used - BitTorrent.

Other end user impact
---------------------

The user has to specify which images require torrent provisioning. It
could be done by providing an additional image property:
``torrent_file= <scheme://path>``, this parameter will indicate that torrent
provisioning should be used for this image. If ``torrent_file``
is not provided, the default provisioning method will be used.

Scalability impact
------------------

Using BitTorrent for image data distribution should reduce load on the
OpenStack control plane, storage and networking. The load will move to
the BMs being deployed.

Using BitTorrent image distribution for provisioning allows to offload
storage cluster and fully utilize concurrent network bandwidth.
The BitTorrent provisioning reduces overall time required to deploy an
image to a number of nodes [2]_.


Performance Impact
------------------

* Additional CPU resources required for generating torrent metadata
  on the BMs.

* Torrent tracker will additionally utilize CPU and networking during
  node provisioning.

* BM nodes will do additional network calls to torrent tracker and exchange
  meta information with peers.

* All above impacts are insignificant in comparison with the benefit of the
  shorter deployment time and of the control plane resources spared.

Other deployer impact
---------------------

* Torrent tracker service should be deployed for keeping information
  about peers. There are no special requirements for the  torrent tracker,
  it would be used only for peer announcement. It could be installed on
  conductor nodes.

* Glare service to store torrent files (or use glance or separate http server
  as alternative).

* The bootstrap image will require a torrent client.

* Additional configuration for Ironic (configuration option
  ``[deploy]enable_torrent_provisioning`` is enabled ``True`` and
  ``[deploy]torrent_trackers`` is provided).

Developer impact
----------------

Each deploy interface might implement BitTorrent provisioning to support
the new feature.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  * ashestakov
  * aarefiev

Work Items
----------

* Implement BitTorrent provisioning for agent driver;

* Implement new functionality on agent ramdisk side;

* Implement devstack plugin part;

* Add new gate job.

Dependencies
============

* python bcoding_ library is required;

* BitTorrent client on deploy image.

Testing
=======

* Unit tests coverage;

* A new gate job will be created to ensure BitTorrent provisioning works.

Upgrades and Backwards Compatibility
====================================

This change is backward compatible. Ironic continues to use HTTP for node
provisioning by default.
Also configuration option ``[deploy]enable_torrent_provisioning`` will be
turned off by default.

Old version of IPA will ignore provided BitTorrent metadata.

Documentation Impact
====================

* Document BitTorrent provisioning approach and benefits.

* Document process on how to switch on new provisioning.


References
==========

.. [1] The BitTorrent Protocol Specification:
       http://www.bittorrent.org/beps/bep_0003.html

.. [2] The Rocks Linux Cluster Project:
       http://www.rocksclusters.org/

.. [3] The Accept-Ranges HTTP/1.1 Byte-Serving extension:
       https://en.wikipedia.org/wiki/Byte_serving

.. _webseed: http://www.bittorrent.org/beps/bep_0019.html

.. _Aria2: https://aria2.github.io

.. _qbittorent-nox: https://github.com/qbittorrent/qBittorrent/wiki/Running-qBittorrent-without-X-server

.. _Glare: https://github.com/openstack/glare

.. _bcoding: https://pypi.python.org/pypi/bcoding/1.5

.. _DHT: http://www.bittorrent.org/beps/bep_0005.html

.. _LPD: http://www.bittorrent.org/beps/bep_0014.html

.. _PEX: http://www.bittorrent.org/beps/bep_0011.html

.. _multi-tracker: http://www.bittorrent.org/beps/bep_0012.html

.. _ctorrent: http://ctorrent.sourceforge.net

.. _transmission: https://transmissionbt.com

.. _mktorrent: http://mktorrent.sourceforge.net/

.. _Swift: http://developer.openstack.org/api-ref/object-storage/?expanded=get-object-content-and-metadata-detail#

.. _magnet: https://en.wikipedia.org/wiki/Magnet_URI_scheme

.. _LPD specification: http://www.bittorrent.org/beps/bep_0014.html

.. _Kademila: http://pdos.csail.mit.edu/~petar/papers/maymounkov-kademlia-lncs.pdf
