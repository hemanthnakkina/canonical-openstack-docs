Ports and protocols
===================

All ports and protocols used by a Canonical OpenStack (Sunbeam) deployment.
Entries are grouped by traffic category. The **Encryption** column uses
normative terms; see the key below.

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Term
     - Meaning
   * - Required
     - TLS is always active on this path; no operator configuration needed.
   * - Required (when TLS enabled)
     - TLS is active on this path once the operator configures TLS CA or Vault.
       Must be configured before the deployment is reachable from untrusted networks.
   * - Optional
     - TLS is supported but not required on this path.
   * - Not used
     - No encryption on this path. Compensating network controls are required.
       See section Notes.
   * - ClusterIP
     - Reachable only within the Kubernetes pod network.
   * - MetalLB / NodePort
     - Reachable on physical node or MetalLB-allocated IP.


External API endpoints
-----------------------

All OpenStack API services are accessible externally through a single Traefik
ingress. MetalLB allocates the ingress IP from the configured address pool.
Individual service ports below reflect pod-internal defaults; all external
traffic reaches services via Traefik on port 80 or 443.

.. list-table::
   :header-rows: 1
   :widths: 22 8 10 18 42

   * - Service
     - Port
     - Protocol
     - Encryption
     - Notes
   * - Traefik ingress (HTTP)
     - 80
     - TCP
     - Not used
     - MetalLB IP. Redirects to 443 when TLS is configured.
   * - Traefik ingress (HTTPS)
     - 443
     - TCP
     - Required (when TLS enabled)
     - Not active by default. Must be configured before external use.
   * - Horizon (dashboard)
     - 80
     - TCP
     - Required (when TLS enabled)
     - Web UI for OpenStack. Pod-internal port 80; served at the
       ``/openstack-horizon`` path by Traefik. Authenticates users directly
       against Keystone. Must not be accessible without TLS when reachable
       from untrusted networks.
   * - Keystone
     - 5000
     - TCP
     - TLS (required)
     - Admin, internal, and public endpoints all served through Traefik.
   * - Nova API
     - 8774
     - TCP
     - TLS (required)
     - Compute API v2.1.
   * - Nova metadata
     - 8775
     - TCP
     - TLS (required)
     - Served via Traefik; accessed by instances through the metadata path.
   * - Placement
     - 8778
     - TCP
     - TLS (required)
     - Resource provider and allocation API.
   * - Neutron
     - 9696
     - TCP
     - TLS (required)
     - Network API.
   * - Cinder
     - 8776
     - TCP
     - TLS (required)
     - Block storage API. Pod-internal port is 8090; Traefik rewrites to 8776.
   * - Glance
     - 9292
     - TCP
     - TLS (required)
     - Image API.
   * - Heat API
     - 8004
     - TCP
     - TLS (required)
     - Orchestration API.
   * - Heat CloudFormation API
     - 8000
     - TCP
     - TLS (required)
     - CFN-compatible endpoint.
   * - Barbican
     - 9311
     - TCP
     - TLS (required)
     - Key management API. Required for TLS-enabled deployments using Vault.
   * - Octavia
     - 9876
     - TCP
     - TLS (required)
     - Load balancer API.
   * - Designate
     - 9001
     - TCP
     - TLS (required)
     - DNS API.
   * - Magnum
     - 9511
     - TCP
     - TLS (required)
     - Container infrastructure management API.
   * - Manila
     - 8786
     - TCP
     - TLS (required)
     - Shared filesystem API.
   * - Aodh
     - 8042
     - TCP
     - TLS (required)
     - Alarming API.
   * - Gnocchi
     - 8041
     - TCP
     - TLS (required)
     - Metric storage API.
   * - CloudKitty
     - 8889
     - TCP
     - TLS (required)
     - Rating API.
   * - Watcher
     - 9322
     - TCP
     - TLS (required)
     - Resource optimization API.
   * - Ironic
     - 6385
     - TCP
     - TLS (required)
     - Bare metal provisioning API. Only present in baremetal deployments.
   * - Nova noVNC proxy
     - 6080
     - TCP
     - Optional
     - WebSocket-based VNC console access for instances. Access controlled
       by a short-lived Nova console token embedded in the console URL.
       Must be restricted to authorized operator networks by firewall rule.
   * - Nova SPICE proxy
     - 6082
     - TCP
     - Optional
     - SPICE console access for instances. Access controlled by a short-lived
       Nova console token. Must be restricted to authorized operator
       networks by firewall rule.


Internal service communication
--------------------------------

These services are ClusterIP-scoped within the ``openstack`` Kubernetes
namespace. No NetworkPolicy is applied by default; any pod in the cluster can
attempt connections.

.. list-table::
   :header-rows: 1
   :widths: 22 8 10 18 42

   * - Service
     - Port
     - Protocol
     - Encryption
     - Notes
   * - RabbitMQ AMQP
     - 5672
     - TCP
     - Not used
     - Message bus for all OpenStack services. Must not be externally exposed.
       NetworkPolicy restriction to known consumer pods is required.
   * - RabbitMQ management
     - 15672
     - TCP
     - Not used
     - HTTP management API. Must not be externally exposed.
   * - MySQL / Galera
     - 3306
     - TCP
     - Not used
     - Service databases. Must not be externally exposed. NetworkPolicy
       restriction to known consumer pods is required.
   * - Octavia health manager
     - 5555
     - UDP
     - Not used
     - Heartbeats between Octavia and amphora instances. Internal only.
   * - OpenStack Exporter (metrics)
     - 9180
     - TCP
     - Not used
     - Prometheus scrape endpoint. ClusterIP only; restricted to COS stack.


Control plane services
-----------------------

These services manage cluster membership, Kubernetes state, and Juju
orchestration. They run on physical or virtual nodes, outside Kubernetes pods.

.. list-table::
   :header-rows: 1
   :widths: 22 8 10 18 42

   * - Service
     - Port
     - Protocol
     - Encryption
     - Notes
   * - MicroK8s / k8s API server
     - 16443
     - TCP
     - Required
     - Kubernetes API. Juju and sunbeam CLI connect here. Standard upstream k8s
       uses 6443; MicroK8s defaults to 16443.
   * - etcd (k8s cluster state)
     - 2379
     - TCP
     - Required
     - Client-to-etcd. Internal to Kubernetes control plane.
   * - etcd (peer replication)
     - 2380
     - TCP
     - Required
     - etcd raft peer communication.
   * - Juju controller API
     - 17070
     - TCP
     - Required
     - Juju agent and CLI connections. Must be restricted from tenant address
       ranges. Must not be reachable from instance floating IP ranges.
   * - sunbeamd / clusterd
     - 7000
     - TCP
     - Required
     - MicroCluster membership API. Must be restricted to sunbeam node
       addresses only.
   * - MAAS API (HTTP)
     - 5240
     - TCP
     - Not used
     - MAAS deployments only. Used by the MAAS region controller.
   * - MAAS API (HTTPS)
     - 5249
     - TCP
     - Required
     - MAAS deployments only. TLS listener for the MAAS region controller.


OVN and data plane traffic
---------------------------

OVN manages east-west tenant traffic and north-south NAT. Tunnel traffic is
unencrypted by default; IPsec is available but not configured by default.

.. list-table::
   :header-rows: 1
   :widths: 22 8 10 18 42

   * - Service
     - Port
     - Protocol
     - Encryption
     - Notes
   * - Geneve tunnel (east-west)
     - 6081
     - UDP
     - Not used; Optional (IPsec)
     - Default OVN encapsulation between compute nodes. Carries inter-instance
       traffic in cleartext by default. OVN IPsec is available for environments
       where data plane confidentiality is required.
   * - VXLAN tunnel (alternative)
     - 4789
     - UDP
     - Not used
     - Used if VXLAN encapsulation is selected instead of Geneve. No
       encryption option available.
   * - OVN Northbound DB
     - 6641
     - TCP
     - Required
     - Accessed by Neutron OVN driver and ovn-northd. Cluster-internal.
   * - OVN Southbound DB
     - 6642
     - TCP
     - Required
     - Accessed by ovn-controller on each compute node.
   * - OVN Southbound DB (admin)
     - 16642
     - TCP
     - Required
     - Administrative access to the southbound database.
   * - OVN NB cluster (raft)
     - 6643
     - TCP
     - Required
     - OVN Northbound DB raft peer replication.
   * - OVN SB cluster (raft)
     - 6644
     - TCP
     - Required
     - OVN Southbound DB raft peer replication.


Storage communication
----------------------

Ceph is the default storage backend. All traffic below is between compute
nodes, the Cinder volume service, and Ceph OSD/MON nodes.

.. list-table::
   :header-rows: 1
   :widths: 22 8 10 18 42

   * - Service
     - Port
     - Protocol
     - Encryption
     - Notes
   * - Ceph Monitor (v2 msgr)
     - 3300
     - TCP
     - Not used
     - Preferred protocol for Ceph Nautilus and later. Used by clients and OSDs
       to reach monitors.
   * - Ceph Monitor (v1 msgr / legacy)
     - 6789
     - TCP
     - Not used
     - Legacy monitor port. Retained for compatibility; v2 is preferred.
   * - Ceph OSD data / replication
     - 6800–7300
     - TCP
     - Not used
     - OSD peer-to-peer replication and client I/O. Range is per-OSD; exact
       assignments vary by node.
   * - Ceph Dashboard
     - 8443
     - TCP
     - Required
     - Administrative web UI. Not used in normal Sunbeam operations. Must be
       restricted to management plane addresses.


Observability stack
--------------------

These ports are used by COS (Canonical Observability Stack) components when
deployed alongside Sunbeam. All are internal to the observability cluster
unless explicitly exposed.

.. list-table::
   :header-rows: 1
   :widths: 22 8 10 18 42

   * - Service
     - Port
     - Protocol
     - Encryption
     - Notes
   * - Prometheus
     - 9090
     - TCP
     - Not used
     - Metrics collection. Scraped by Grafana.
   * - Grafana
     - 3000
     - TCP
     - Optional
     - Dashboard UI. Must be restricted to operator networks.
   * - Loki
     - 3100
     - TCP
     - Not used
     - Log aggregation. Agents push via Promtail or Grafana Agent.
   * - Promtail / Grafana Agent (HTTP)
     - 9080
     - TCP
     - Not used
     - Per-pod agent HTTP listener.
   * - Promtail / Grafana Agent (gRPC)
     - 9095
     - TCP
     - Not used
     - Per-pod agent gRPC listener.


Operational access
-------------------

.. list-table::
   :header-rows: 1
   :widths: 22 8 10 18 42

   * - Service
     - Port
     - Protocol
     - Encryption
     - Notes
   * - SSH
     - 22
     - TCP
     - Required
     - Required on all nodes for Juju agent access and operator shell access.
       Must be restricted to management plane address ranges.
   * - DNS (Designate / BIND)
     - 53
     - UDP / TCP
     - Not used
     - External DNS resolution. BIND pods expose both UDP and TCP.
   * - RNDC (BIND control)
     - 953
     - TCP
     - Not used
     - Internal only. Must not be externally exposed.
   * - Ironic conductor HTTP
     - 8080
     - TCP
     - Not used
     - HTTP file server for PXE/iPXE image delivery in baremetal deployments.
   * - TFTP (Ironic)
     - 69
     - UDP
     - Not used
     - PXE boot in baremetal deployments. Requires L2 reachability from
       target nodes.


Notes and operator responsibilities
-------------------------------------

**Encryption defaults**
   TLS is not active on the Traefik ingress by default. Any deployment
   reachable from untrusted networks must have TLS configured before use.
   See :doc:`Service endpoint encryption </explanation/service-endpoint-encryption>`.

**Internal cluster boundaries**
   RabbitMQ (5672) and MySQL (3306) are ClusterIP-scoped but have no
   NetworkPolicy applied by default. NetworkPolicy deny-all ingress with
   explicit per-service allow rules is the required baseline for both.

**Juju controller (17070)**
   In single-network deployments, port 17070 shares a subnet with tenant
   floating IPs. Host-based firewall rules must restrict access from instance
   address ranges on all sunbeam and management nodes.

**OVN tunnel traffic**
   Geneve and VXLAN tunnels are unencrypted. An observer on the data network
   can capture inter-hypervisor traffic. OVN IPsec is available for environments
   where data plane confidentiality is required; it is not configured by default.

**Port ranges**
   Ceph OSD ports (6800–7300) vary by deployment. The range accommodates
   multiple OSDs per node; exact assignments are managed by Ceph automatically.

**Deployment variation**
   Baremetal deployments (MAAS-backed) add TFTP (69), Ironic HTTP (8080), and
   MAAS API (5240/5249) to the surface area. MAAS ports must be reachable from
   all managed bare metal nodes.

For related reference material, see:

* :doc:`Certificates and TLS </reference/security/certificates-and-tls>`
* :doc:`Encryption and Data Protection </reference/security/encryption-and-data-protection>`
* :doc:`API auditing </reference/api-auditing>`
* :doc:`Network Exposure and Ingress Model </explanation/security/network-exposure-and-ingress-model>`
* :doc:`Service endpoint encryption </explanation/service-endpoint-encryption>`
* :doc:`Network traffic isolation with MAAS </explanation/network-traffic-isolation-with-maas>`
