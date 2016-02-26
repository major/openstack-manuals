==================================
Load Balancer as a Service (LBaaS)
==================================

The Networking service offers two load balancer implementations through the
``neutron-lbaas`` service plug-in:

* LBaaS v1: introduced in Juno (deprecated in Liberty)
* LBaaS v2: introduced in Kilo

Both implementations use agents. The agents handle the HAProxy configuration
and manage the HAProxy daemon. LBaaS v2 adds the concept of listeners to the
LBaaS v1 load balancers. LBaaS v2 allows you to configure multiple listener
ports on a single load balancer IP address.

Another LBaaS v2 implementation, `Octavia`_, has a separate API and
separate worker processes that build load balancers within virtual machines on
hypervisors that are managed by the Compute service. You do not need an agent
for Octavia.

Currently, no migration path exists between v1 and v2 load balancers. If you
choose to switch from v1 to v2, you must recreate all load balancers, pools,
and health monitors.

.. _octavia: http://docs.openstack.org/developer/octavia/

LBaaS v1
~~~~~~~~

LBaaS v1 is deprecated in the Liberty release. These links provide more
details about how LBaaS v1 works and how to configure it:

* `Load-Balancer-as-a-Service (LBaaS) overview`_
* `Basic Load-Balancer-as-a-Service operations`_

.. _Load-Balancer-as-a-Service (LBaaS) overview: http://docs.openstack.org/admin-guide-cloud/networking_introduction.html#load-balancer-as-a-service-lbaas-overview
.. _Basic Load-Balancer-as-a-Service operations: http://docs.openstack.org/admin-guide-cloud/networking_adv-features.html#basic-load-balancer-as-a-service-operations

LBaaS v2
~~~~~~~~

LBaaS v2 has several new concepts to understand:

.. image:: figures/lbaasv2-diagram.png
   :alt: LBaaS v2 layout

Load balancer
 The load balancer occupies a neutron network port and has an IP address
 assigned from a subnet.

Listener
 Load balancers can listen for requests on multiple ports. Each one of those
 ports is specified by a listener.

Pool
 A pool holds a list of members that serve content through the load balancer.

Member
 Members are servers that serve traffic behind a load balancer. Each member
 is specified by the IP address and port that it uses to serve traffic.

Health monitor
 Members may go offline from time to time and health monitors divert traffic
 away from members that are not responding properly. Health monitors are
 associated with pools.

LBaaS v2 has multiple implementations via different service plug-ins. The two
most common implementations use either an agent or the Octavia services.

Configuring LBaaS v2 with an agent
----------------------------------

#.  Add the LBaaS v2 service plug-in to the ``service_plugins`` configuration
    directive in ``/etc/neutron/neutron.conf``. The plug-in list is comma
    separated:

    .. code-block:: console

       service_plugins = [existing service plugins],neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2

#.  Add the LBaaS v2 service provider to the ``service_provider`` configuration
    directive within the ``[service_providers]`` section in
    ``/etc/neutron/neutron.conf``:

    .. code-block:: console

       service_provider = LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default

    If you have existing service providers for other networking service
    plug-ins, such as VPNaaS or FWaaS, add the ``service_provider`` line shown
    above in the ``[service_providers]`` section as a separate line. These
    configuration directives are repeatable and are not comma separated.

#.  Run the ``neutron-lbaas`` database migration:

    .. code-block:: console

       neutron-db-manage --service lbaas upgrade head

#.  If you have deployed LBaaS v1, **stop the LBaaS v1 agent now**. The v1 and
    v2 agents **cannot** run simultaneously.

#.  Start the LBaaS v2 agent:

    .. code-block:: console

       neutron-lbaasv2-agent \
       --config-file /etc/neutron/neutron.conf \
       --config-file /etc/neutron/lbaas_agent.ini

#.  Restart the Network service to activate the new configuration. You are now
    ready to create load balancers with the LBaaS v2 agent.

Configuring LBaaS v2 with Octavia
---------------------------------

Octavia provides additional capabilities for load balancers, including using a
compute driver to build instances that operate as load balancers. The `Hands
on Lab - Install and Configure OpenStack Octavia`_ session at the OpenStack
Summit in Tokyo provides an overview of Octavia.

The DevStack documentation offers a `simple method to deploy Octavia`_ and test
the service with redundant load balancer instances. If you already have Octavia
installed and configured within your environment, you can configure the Network
service to use Octavia:

#.  Add the LBaaS v2 service plug-in to the ``service_plugins`` configuration
    directive in ``/etc/neutron/neutron.conf``. The plug-in list is comma
    separated:

    .. code-block:: console

       service_plugins = [existing service plugins],neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2

#.  Add the Octavia service provider to the ``service_provider`` configuration
    directive within the ``[service_providers]`` section in
    ``/etc/neutron/neutron.conf``:

    .. code-block:: console

       service_provider = LOADBALANCERV2:Octavia:neutron_lbaas.drivers.octavia.driver.OctaviaDriver:default

    Ensure that the LBaaS v1 and v2 service providers are removed from the
    ``[service_providers]`` section. They are not used with Octavia. **Verify
    that all LBaaS agents are stopped.**

#.  Restart the Network service to activate the new configuration. You are now
    ready to create and manage load balancers with Octavia.

.. _Hands on Lab - Install and Configure OpenStack Octavia: https://www.openstack.org/summit/tokyo-2015/videos/presentation/rsvp-required-hands-on-lab-install-and-configure-openstack-octavia
.. _simple method to deploy Octavia: http://docs.openstack.org/developer/devstack/guides/devstack-with-lbaas-v2.html

LBaaS v2 Operations
~~~~~~~~~~~~~~~~~~~

The same neutron commands are used for LBaaS v2 with an agent or with Octavia.

Building an LBaaS v2 load balancer
----------------------------------

The following steps will help you build a very basic load balancer:

#.  Start by creating a load balancer on a network. In this example, the
    ``private`` network is an isolated network with two web server instances:

    .. code-block:: console

       $ neutron lbaas-loadbalancer-create --name test-lb private-subnet

#.  You can view the load balancer status and IP address with the
    ``lbaas-loadbalancer-show`` command:

    .. code-block:: console

       $ neutron lbaas-loadbalancer-show test-lb
       +---------------------+------------------------------------------------+
       | Field               | Value                                          |
       +---------------------+------------------------------------------------+
       | admin_state_up      | True                                           |
       | description         |                                                |
       | id                  | 7780f9dd-e5dd-43a9-af81-0d2d1bd9c386           |
       | listeners           | {"id": "23442d6a-4d82-40ee-8d08-243750dbc191"} |
       |                     | {"id": "7e0d084d-6d67-47e6-9f77-0115e6cf9ba8"} |
       | name                | test-lb                                        |
       | operating_status    | ONLINE                                         |
       | provider            | haproxy                                        |
       | provisioning_status | ACTIVE                                         |
       | tenant_id           | fbfce4cb346c4f9097a977c54904cafd               |
       | vip_address         | 192.168.1.22                                   |
       | vip_port_id         | 9f8f8a75-a731-4a34-b622-864907e1d556           |
       | vip_subnet_id       | f1e7827d-1bfe-40b6-b8f0-2d9fd946f59b           |
        +---------------------+------------------------------------------------+

    This load balancer is active and ready to serve traffic on ``192.168.1.22``.

#.  Verify that the load balancer is responding to pings before moving further:

    .. code-block:: console

       $ ping -c 4 192.168.1.22
       PING 192.168.1.22 (192.168.1.22) 56(84) bytes of data.
       64 bytes from 192.168.1.22: icmp_seq=1 ttl=62 time=0.410 ms
       64 bytes from 192.168.1.22: icmp_seq=2 ttl=62 time=0.407 ms
       64 bytes from 192.168.1.22: icmp_seq=3 ttl=62 time=0.396 ms
       64 bytes from 192.168.1.22: icmp_seq=4 ttl=62 time=0.397 ms

       --- 192.168.1.22 ping statistics ---
       4 packets transmitted, 4 received, 0% packet loss, time 2997ms
       rtt min/avg/max/mdev = 0.396/0.402/0.410/0.020 ms

Adding an HTTP listener
-----------------------

#.  With the load balancer online, you can add a listener for plaintext
    HTTP traffic on port 80:

    .. code-block:: console

       $ neutron lbaas-listener-create \
         --name test-lb-http \
         --loadbalancer test-lb \
         --protocol HTTP \
         --protocol-port 80

#.  You can begin building a pool and adding members to the pool to serve HTTP
    content on port 80. For this example, the web servers are ``192.168.1.16``
    and ``192.168.1.17``:

    .. code-block:: console

       $ neutron lbaas-pool-create \
         --name test-lb-pool-http \
         --lb-algorithm ROUND_ROBIN \
         --listener test-lb-http \
         --protocol HTTP
       $ neutron lbaas-member-create \
         --subnet private-subnet \
         --address 192.168.1.16 \
         --protocol-port 80 \
         test-lb-pool-http
       $ neutron lbaas-member-create \
         --subnet private-subnet \
         --address 192.168.1.17 \
         --protocol-port 80 \
         test-lb-pool-http

#.  You can use ``curl`` to verify connectivity through the load balancers to
    your web servers:

    .. code-block:: console

       $ curl 192.168.1.22
       web2
       $ curl 192.168.1.22
       web1
       $ curl 192.168.1.22
       web2
       $ curl 192.168.1.22
       web1

    In this example, the load balancer uses the round robin algorithm and the
    traffic alternates between the web servers on the backend.

#.  You can add a health monitor so that unresponsive servers are removed
    from the pool:

    .. code-block:: console

       $ neutron lbaas-healthmonitor-create \
         --delay 5 \
         --max-retries 2 \
         --timeout 10 \
         --type HTTP \
         --pool test-lb-pool-http

    In this example, the health monitor removes the server from the pool if
    it fails a health check at two five-second intervals. When the server
    recovers and begins responding to health checks again, it is added to
    the pool once again.

Adding an HTTPS listener
------------------------

You can add another listener on port 443 for HTTPS traffic. LBaaS v2 offers
SSL/TLS termination at the load balancer, but this example takes a simpler
approach and allows encrypted connections to terminate at each member server.

#.  Start by creating a listener, attaching a pool, and then adding members:

    .. code-block:: console

       $ neutron lbaas-listener-create \
         --name test-lb-https \
         --loadbalancer test-lb \
         --protocol HTTPS \
         --protocol-port 443
       $ neutron lbaas-pool-create \
         --name test-lb-pool-https \
         --lb-algorithm LEAST_CONNECTIONS \
         --listener test-lb-https \
         --protocol HTTPS
       $ neutron lbaas-member-create \
         --subnet private-subnet \
         --address 192.168.1.16 \
         --protocol-port 443 \
         test-lb-pool-https
       $ neutron lbaas-member-create \
         --subnet private-subnet \
         --address 192.168.1.17 \
         --protocol-port 443 \
         test-lb-pool-https

#.  You can add a health monitor for the HTTPS pool as well:

    .. code-block:: console

       $ neutron lbaas-healthmonitor-create \
         --delay 5 \
         --max-retries 2 \
         --timeout 10 \
         --type HTTPS \
         --pool test-lb-pool-https

    The load balancer now handles traffic on ports 80 and 443.

Associating a floating IP address
---------------------------------

Load balancers that are deployed on a public or provider network that are
accessible to external clients will not need a floating IP address assigned.
The virtual IP address (VIP) of those load balancers will be directly
accessible to external clients.

However, load balancers deployed onto private or isolated networks will need a
floating IP address assigned if they must be accessible to external clients. To
complete this step, you must have a router between the private and public
networks and an available floating IP address.

You can use the ``lbaas-loadbalancer-show`` command from the beginning of this
section to locate the ``vip_port_id``. The ``vip_port_id`` is the ID of the
network port that is assigned to the load balancer. You can associate a free
floating IP address to the load balancer using ``floatingip-associate``:

.. code-block:: console

   $ neutron floatingip-associate FLOATINGIP_ID LOAD_BALANCER_PORT_ID


