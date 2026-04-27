Network Exposure and Ingress Model
==================================

Canonical OpenStack's security posture is determined by network topology more
than by any individual configuration setting. Which components can reach which
others, which boundaries are physically enforced, and how far a compromise can
propagate are all network-level decisions made at deployment time. This
document explains where the resulting boundaries are real and where they exist
only in configuration.


Network planes and trust boundaries
-------------------------------------

Canonical OpenStack organises traffic into named planes:

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Plane
     - Traffic
   * - **public**
     - User-to-service: Keystone authentication, OpenStack CLI calls, Horizon
       dashboard access.
   * - **internal**
     - Service-to-service: Nova Conductor messaging to RabbitMQ, Cinder API
       calls to the volume conductor, Neutron agent heartbeats, inter-service
       Keystone token validation.
   * - **management**
     - Operator and tooling: Juju controller agent communications, MAAS machine
       provisioning, ``clusterd`` cluster state replication.
   * - **data**
     - Hypervisor east-west: OVN/OVS encapsulated instance packets between
       compute nodes.
   * - **storage**
     - Hypervisor-to-storage: Ceph RBD block I/O from Nova Compute to the
       Ceph cluster.
   * - **storage-cluster**
     - Storage replication: Ceph OSD data migration and cluster rebalancing.
   * - *external networking*
     - North-south: traffic entering or leaving instances through floating IPs
       and OVN gateway chassis.
   * - *private networking*
     - Tenant-to-API: traffic from tenant instances to OpenStack API endpoints,
       including the Nova metadata service at ``169.254.169.254``.

What matters operationally is not the plane name but the strength of the
boundary enforcing it. Three categories apply:

**Strong boundaries** exist when a plane maps to a physically separate network
segment — a dedicated VLAN or subnet — and Juju network spaces enforce that
mapping. A packet on the public VLAN cannot reach the internal VLAN without
crossing a router the operator controls. Compromising a component on the public
plane grants no Layer 2 reachability to RabbitMQ, MySQL, or any other
internal-plane service.

**Weak boundaries** exist where scoping is logical rather than physical.
RabbitMQ and MySQL are Kubernetes ClusterIP services, reachable only within the
pod network using cluster-internal DNS names. That boundary is real but is
enforced by Kubernetes service scoping, not the physical network — any pod in
the cluster can attempt a connection.

**Absent-by-default boundaries** are those the operator must explicitly create.
Kubernetes NetworkPolicy and upstream firewall rules fall into this category.
Neither is applied automatically, and neither is present in a default Sunbeam
installation.

In a single-network manual deployment — the default for new Sunbeam
installations — all planes share a single physical network; only weak and
absent-by-default boundaries exist. The most consequential gap is between the
management plane and everything else. The Juju controller distributes
configuration secrets, including all RabbitMQ and MySQL credentials, to service
agents through its API, and ``sunbeamd`` maintains cluster membership state. In
a MAAS deployment with an isolated management VLAN, reaching either API from a
compromised tenant instance requires crossing a routed boundary the operator
controls. In a single-network deployment, those APIs share a subnet with tenant
floating IPs: an instance with an allow-all ingress security group can address
the Juju controller directly. The controller requires credentials and uses TLS,
but it is now reachable by a host the attacker controls, making
credential brute-force and targeted exploits viable attack vectors. MAAS
deployments with properly mapped Juju spaces close this gap structurally.
Manual deployments must compensate with host-based firewall rules on sunbeam
and management nodes rejecting connections to Juju and ``sunbeamd`` API ports
from instance address ranges.


The ingress boundary: MetalLB, Traefik, and Keystone
------------------------------------------------------

All OpenStack API services run as Kubernetes pods in the ``openstack``
namespace and are unreachable from outside the cluster by default. External
access flows through two layers.

**Where traffic enters.** MetalLB operates as a Layer 2 load balancer,
allocating IPs from the range configured in the deployment manifest
(``addons.metallb``, for example ``192.168.123.81-192.168.123.90``) and
announcing them via ARP on the public network's Layer 2 domain. Any host on
the same Layer 2 segment can reach an allocated IP; MetalLB performs no source
filtering. Traefik holds one of these IPs as a ``LoadBalancer``-type Kubernetes
service address and acts as the sole reverse proxy for all OpenStack endpoints,
routing requests to the correct backend by URL path prefix:
``/openstack-keystone`` to Keystone pods, ``/openstack-nova`` to Nova API pods,
and so on. Every deployed service is reachable through a Traefik path by
default; there is no network-level path to expose Keystone while blocking
Cinder or Glance from the same source.

**Where TLS terminates.** TLS terminates at Traefik. Traffic between Traefik
and backend pods travels over the cluster pod network in cleartext. Without TLS
enabled, the entire path from client to API is plain HTTP: Keystone credentials
and tokens are transmitted unencrypted. A passive observer on the public
network segment — a monitoring appliance, a workstation with a promiscuous NIC
— can capture a Keystone authentication response containing a scoped token. A
Keystone token carries the full set of authorization claims that every
downstream service accepts without re-consulting Keystone, and is valid for one
hour by default (configurable via ``[token] expiration``). The observer can
replay that token against any service endpoint through the same Traefik IP
within that window: Nova to enumerate instances, Cinder to list volumes, Glance
to download images. Keystone provides a token revocation endpoint, but
revocation requires the legitimate holder to detect the compromise and act
before expiry; without audit logging that correlates a token's source IP across
requests, the window closes undetected. TLS on Traefik is the minimum required
control for any deployment accessible from networks not exclusively under
operator control. Sunbeam supports two provisioning paths: TLS CA (Traefik
certificates signed by an external CA) and TLS Vault (HashiCorp Vault as an
intermediate CA). Both are operator-configured and not enabled by default.

Sunbeam distributes the CA certificate to service pods through Keystone's
certificate injection path. During a CA rotation, any pod still holding the
old CA while Traefik serves a certificate signed by the new one will reject
outbound TLS connections to Traefik — not with a visible handshake error, but
as Nova 503 responses or silent Cinder attachment failures. The distribution
window must be minimized.

**Where identity is enforced.** Keystone is the sole identity authority for
the entire API surface. Every token it issues is accepted by Nova, Neutron,
Cinder, Glance, and every other service until expiry or explicit revocation —
no service re-consults Keystone per request. Sunbeam serves admin, internal,
and public endpoints through the same Traefik path; policy enforcement is the
only distinction between a tenant authenticating and an operator calling an
admin-scope API. A compromised admin token reaches everything Keystone
authorizes across all services and projects. The credential bundle from
``sunbeam credentials`` must be rotated after bootstrap, stored in a secrets
management system, and restricted to a named set of operators.


Internal traffic and the data plane
-------------------------------------

Within the pod network, RabbitMQ and MySQL are reachable by any pod in the
``openstack`` namespace — a weak internal boundary enforced by ClusterIP
scoping but not by NetworkPolicy. RabbitMQ enforces virtual host access control
per service; MySQL enforces per-database grants. In larger deployments, MySQL
is provisioned as a separate instance per service, limiting a single credential
leak to one database. In smaller deployments a single MySQL instance serves all
services, so one leaked credential exposes all service databases.

A compromised pod does not need to steal credentials — they are present in the
pod's environment. RabbitMQ consumers trust queued messages without additional
authentication, so queue access bypasses Keystone authorization entirely.

Example Scenario: RCE in the Nova API pod reaches RabbitMQ
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A remote code execution vulnerability is disclosed in the Nova API server Rock.
An attacker sends a crafted request to ``/openstack-nova`` through Traefik and
gains code execution as the ``nova`` process inside the Nova API pod. Using
Nova's RabbitMQ credentials — present as environment variables in the pod — the
attacker connects directly to the RabbitMQ ClusterIP service. Nova's virtual
host carries scheduling instructions and Nova Compute callbacks; injecting
crafted messages causes Nova Compute workers on hypervisors to delete disk
images, attach network devices, or modify instance metadata outside the
Keystone authorization path. Because Neutron agents share the same cluster,
the attacker can also affect Neutron's view of network state. Without
NetworkPolicy, the compromise moves unobstructed from the API tier into the
compute and network data planes. The practical baseline is deny-all ingress to
RabbitMQ and MySQL within the ``openstack`` namespace with explicit per-service
allow rules, which limits blast radius without requiring a complete model of
every pod-to-pod path upfront.

On the physical network, the ``internal`` plane carries service-to-service
traffic between nodes. In single-network deployments it shares the physical
segment with the public plane, giving any publicly reachable node direct packet
access to internal service addresses. Isolating the ``internal`` plane onto a
dedicated VLAN requires deliberate routing to cross.

OVN provides VXLAN or Geneve encapsulation for east-west tenant traffic between
hypervisors, isolating tenant networks logically but not encrypting them. An
observer on the data network can capture inter-hypervisor tunnel traffic in
cleartext. Security groups are enforced as OVS flow rules installed by the
Neutron OVN agent on each hypervisor; the default group permits all egress and
restricts ingress to same-group traffic.

Security group enforcement depends on MicroOVN quorum. If a maintenance
operation drops the OVN southbound database below quorum, ``ovn-controller``
on each compute node retains its last-known flow tables — but security group
changes accepted into Neutron's MySQL database do not reach the OVN northbound
database. Traffic the new rules were meant to block passes freely until
MicroOVN recovers and ``ovn-northd`` reconciles. Distributing OVN chassis roles across
enough nodes to survive individual failures and monitoring MicroOVN quorum are
the controls that close this gap.

Floating IPs are implemented as OVN NAT rules that map an external address to
an instance's private port. The security group on that port governs traffic
through the mapping; the OVN gateway chassis performing NAT is not itself a
filtering boundary. An instance with a floating IP and a permissive security
group is directly exposed on the external network on all permitted ports
regardless of any upstream firewall rules. Tenant instances that contact
OpenStack APIs via private networking reach Keystone and Nova through the same
Traefik IP that operators use; traffic from instance CIDRs that should not
reach admin-scope API paths must be blocked upstream of the MetalLB IP.


Operational tradeoffs
-----------------------

Plane isolation versus deployment complexity
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A MAAS deployment with all six planes isolated requires VLANs on every switch
port in the path, Juju network spaces mapped to each subnet, and MAAS network
configurations kept in sync as nodes are added or replaced. Where tenant
instances run untrusted workloads on a network shared with operations tooling,
that overhead is not optional — plane isolation is the primary control
preventing a compromised instance from reaching the Juju controller,
``clusterd``, and MAAS APIs. The decision must be made at design time; a
default single-network deployment cannot be retroactively isolated without
redeploying with MAAS and reconfiguring network spaces.

Ingress consolidation versus selective exposure
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

One Traefik instance and one MetalLB IP means one certificate to manage, one
firewall entry, and one point of failure. It also means there is no
network-level path to permit Keystone while blocking Cinder from the same
source, or to expose Nova to tenants while restricting Glance to operators.
Per-service access control requires Traefik middleware — IP allowlists per
path prefix — that Sunbeam does not configure by default. When the MetalLB ARP
lease holder fails and a new announcement is delayed, every OpenStack API
becomes unreachable simultaneously, because Keystone's endpoint catalogue
contains only Traefik URLs and clients have no fallback path.

NetworkPolicy strictness versus maintenance burden
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Applying deny-all ingress to RabbitMQ and MySQL with explicit per-service allow
rules addresses the most consequential exposure — pod-level compromise reaching
the message bus or database tier — without requiring a complete map of every
pod-to-pod path. Extending coverage to all inter-pod communication requires
tracking every path added by each enabled service: Octavia, Barbican, Ironic,
and Manila each introduce new requirements, and a missing allow rule surfaces
as API timeouts rather than connection-refused errors.

Tunnel encapsulation versus data plane confidentiality
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

VXLAN and Geneve are connectionless, stateless, and offloaded on most modern
NICs at no per-packet CPU cost. OVN IPsec encrypts and authenticates the same
tunnels but requires per-hypervisor key management, CPU overhead where hardware
offload is unavailable, and added complexity in OVN chassis configuration and
key rotation procedures. For most deployments, physically isolating the data
network onto dedicated infrastructure is the appropriate control. For
environments with co-located tenants at distinct trust levels, regulated
workloads, or a data network that traverses shared infrastructure, encryption
is necessary — but its operational cost must be planned for explicitly.


Summary
---------

The network security of a Canonical OpenStack deployment rests on three
factors: whether network planes are physically isolated, whether TLS is
enforced at the Traefik ingress, and how securely Keystone credentials are
managed.

Plane isolation is the strongest available control but exists only in MAAS
deployments with Juju network spaces explicitly mapped to separate VLANs; it
is absent by design in single-network deployments. Traefik concentrating all
API access at a single IP and port simplifies operation but removes per-service
network controls and means a Traefik failure is a total API outage. TLS on the
Traefik ingress is the minimum required protection for any deployment accessible
from networks outside operator control. Keystone is the root of trust for the
entire API surface; a compromised admin token propagates uniformly to every
downstream service with no network boundary to limit it.

The default deployment is functional, not hardened. Boundaries that are not
physically enforced do not exist — they must be created, and the
responsibility for creating them rests with the operator.

For related configuration and implementation details, see:

* :doc:`Network traffic isolation with MAAS </explanation/network-traffic-isolation-with-maas>`
* :doc:`Identity and Access Model </explanation/security/identity-and-access-model>`
* :doc:`Secrets and Key Management </explanation/security/secrets-and-key-management>`
* :doc:`Compute Isolation </explanation/security/compute-isolation>`
* :doc:`Service endpoint encryption </explanation/service-endpoint-encryption>`
* :doc:`Design considerations </explanation/design-considerations>`
