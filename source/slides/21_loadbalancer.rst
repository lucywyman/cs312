.. _21_loadbalancer:

Load Balancers
==============

Definition
----------

* Distributes a workload across multiple computing resources
* Typically used for HTTP/HTTPS sites, but can be used for about anything
* "*Scale out*" and maximize throughput
* Offer redundancy automatically

Types
-----

**Round Robin DNS**
  Assign multiple IP addresses to the same domain name. Simple and easy to
  implement but doesn't allow for automatic fail over.

**Software TCP/UDP load balancing**
  Uses the OSI network layer to provide an entry point for traffic and then uses
  logic to pass the traffic to backend nodes. HAProxy, Nginx, Apache and Varnish
  can all act as a load balancer.

**Hardware TCP/UDP load balancing**
  Proprietary hardware that provides the same features (and usually more) than
  the software load balancing tools. They typically can provide more throughput
  and offer SSL off loading (which is critical for high traffic sites).

Round Robin DNS (RR DNS)
------------------------

* Multiple IP addresses to the same domain
* Client resolver usually picks the first IP
* Depending on the implementation of the client resolver, this can be somewhat
  random or not.
* Some resolvers always use alphabetical ordering
* DNS Caching and Time-To-Live (TTL) issues
* What happens if a site goes down? How does it affect troubleshooting?

Split-view DNS
--------------

Split-view DNS offers a simple way to split traffic up based on the IP address
of the client's source DNS server. In the case of ``ftp.osuosl.org``, we split
the DNS based on what IPs are routed via Internet2. When used in combination
with Round Robin DNS, it can be a simple and useful load balancing tool.

.. rst-class:: codeblock-sm

::

  # Querying on campus gives us one server
  $ host ftp.osuosl.org
  ftp.osuosl.org has address 140.211.166.134

  # Querying an OpenDNS server, gives us a RR DNS answer
  $ host ftp.osuosl.org 208.67.222.222
  Using domain server:
  Name: 208.67.222.222
  Address: 208.67.222.222#53
  Aliases:

  ftp.osuosl.org has address 64.50.233.100
  ftp.osuosl.org has address 64.50.236.52

Layer 4 vs Layer 7 load balancing
---------------------------------

**Linux Virtual Server (LVS)**
  Layer 4 kernel-based load balancing. Typically only works on a connection
  level but can offer fast balancing. More complicated to setup and manage.

**HAProxy**
  Layer 7 user-space load balancing. Can use both connection and inspect
  packets. Allows you to modify packets or direct traffic based on packets.
  Somewhat easier to setup and manage.

Scheduling Algorithms
---------------------

*Taken from the HAProxy documentation*

**Round Robin**
  Each server is used in turns, according to their weights.  This is the
  smoothest and fairest algorithm when the server's processing time remains
  equally distributed. This algorithm is dynamic, which means that server
  weights may be adjusted on the fly for slow starts for instance.

Scheduling Algorithms
---------------------

**Least Connection**
  The server with the lowest number of connections receives the connection.
  Round-robin is performed within groups of servers of the same load to ensure
  that all servers will be used. Use of this algorithm is recommended where very
  long sessions are expected, such as LDAP, SQL, TSE, etc... but is not very
  well suited for protocols using short sessions such as HTTP.

**First Connection**
  The first server with available connection slots receives the connection. The
  servers are chosen from the lowest numeric identifier to the highest, which
  defaults to the server's position in the farm.  Once a server reaches its
  maxconn value, the next server is used.

Scheduling Algorithms
---------------------

**Source Connection**
  The source IP address is hashed and divided by the total weight of the running
  servers to designate which server will receive the request. This ensures that
  the same client IP address will always reach the same server as long as no
  server goes down or up.

**URI**
  This algorithm hashes either the left part of the URI (before the question
  mark) or the whole URI (if the "whole" parameter is present) and divides the
  hash value by the total weight of the running servers. The result designates
  which server will receive the request.  This ensures that the same URI will
  always be directed to the same server as long as no server goes up or down.

Common Load Balancer Features
-----------------------------

.. csv-table::
  :widths: 40, 90

  Asymmetric load, Ratio to be manually assigned to a backend server
  Priority Activation, "Add or remove backend servers based on the load or other
  metrics"
  SSL Offload & Acceleration, "Specialized hardware to offload SSL CPU demand on
  high traffic sites"
  DDoS attack protection, "Mitigate DDoS attacks using SYN cookies and verifying
  a full TCP handshake before sending off to the backend server"

Common Load Balancer Features
-----------------------------

.. csv-table::
  :widths: 40, 90

  HTTP compression, "Gzip compresses the HTTP objects to reduce bandwidth but
  can increase CPU usage"
  TCP offload, "Consolidate multiple HTTP requests from multiple clients into a
  single TCP socket to the backend servers"
  Health checking, "Balance pools the backend application server to see if its
  functioning correctly"
  HTTP caching, "Balancer stores the static content in memory to serve the
  content faster"

Persistence
-----------

* HTTP sessions
* Want to keep a connection with the same backend to maintain sessions
* Using memcached for storing sessions can help this
* *What other issues?*

Software Load Balancers
-----------------------

**HAProxy**
  High performance software based load balancer that uses TCP and can be used
  for multiple protocols. Been around since 2000 and used by Github, Reddit,
  Twitter, etc. Recently added SSL support but does no caching.

**Varnish**
  HTTP accelerator and static cache server. Focuses specifically on HTTP and can
  act as a load balancer similar to HAProxy. Used by Wikipedia, Facebook and
  Twitter to name a few.

Software Load Balancers
-----------------------

**Nginx**
  Webserver that can also act as a load balancer and a caching system. It
  generally has a low memory footprint.

**Apache**
  Webserver that can act as a load balancer via the ``mod_proxy`` module.
  Provides an easy way to set up but tends to use more memory than the others.

Proprietary Load Balancers
--------------------------

* BIG-IP (F5 Networks)
* NetScaler (Citrix)

Pros/Cons
---------

**Pros**
  * Helps you scale more easily
  * Gives you more flexibility on how to route your web traffic
  * Deal with backend outages more gracefully

**Cons**
  * Makes troubleshooting more complicated
  * Potential single point of failure
  * Configuration can be more complicated

*What else?*

HAProxy
=======

HAProxy Terminology
-------------------

**frontend**
    This defines how HAProxy should forward traffic to backends

**backend**
    A set of servers that receives traffic from HAProxy

**listen**
    A simpler version of frontend and backends

HAProxy Example Configuration
-----------------------------

::

    frontend http
      maxconn 2000
      bind 0.0.0.0:80
      default_backend servers

    frontend https
      maxconn 2000
      bind 0.0.0.0:443 ssl crt /etc/pki/tls/mycert.pem
      default_backend servers

    backend servers
      server mybackendserver 10.0.0.1:80
    # server <name> <ip>:<port> [options]

Check out the docs! http://www.haproxy.org/

HAProxy Example Configuration
-----------------------------

::

    frontend http
      maxconn 2000
      bind 0.0.0.0:80
      acl cs312 hdr(host) cs312.osuosl.org
      acl osl hdr(host) osuosl.org
      use_backend cs312_servers if cs312
      use_backend osl_servers if osl
      default_backend unrecognized_site

    backend cs312_servers
      server cs312-1 10.0.0.1:80
      server cs312-2 10.0.0.2:80

    backend osl_servers
      server osl-1 10.0.1.1:80
      server osl-2 10.0.1.2:80

    backend unrecognized_site
      server others 10.0.255.1:80

Monday
======

* Bring your laptops!
* HW3 will be handed back

