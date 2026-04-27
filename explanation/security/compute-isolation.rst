Compute Isolation
=================

Compute isolation in Canonical OpenStack is the set of boundaries that prevent
one workload from controlling another workload or the control plane through the
compute path. In Sunbeam, those boundaries span Nova scheduling decisions,
hypervisor-level enforcement, Neutron and OVN network controls, storage
attachment rules, image provenance, and identity checks attached to API calls.
No single component provides isolation alone. Isolation quality is determined
by how these controls combine under failure.


How compute isolation operates in Sunbeam
-----------------------------------------

The first boundary is placement. Nova schedules instances onto compute nodes,
and the hypervisor constrains each instance to its assigned virtual CPUs,
memory, and attached devices. That boundary is strong for accidental
interference between tenants, but it is not a cryptographic boundary. A host
compromise or hypervisor escape bypasses tenant separation immediately.

The second boundary is network mediation. Instance traffic reaches other
instances and external networks through Neutron policy translated into OVN and
OVS flow rules on compute and gateway nodes. Security groups are the primary
tenant-facing control. They can isolate workloads effectively when rules are
minimal and maintained, but they are only as strict as their broadest allow.

The third boundary is storage attachment. Cinder and Nova coordinate which
volumes can attach to which instances, and Ceph enforces access through client
credentials on the storage path. This is a meaningful boundary between tenants,
but operator mistakes in attachment workflows or over-permissive service
credentials can still expose data across projects.

The fourth boundary is image trust. Glance controls what boot artifacts are
available. Compute isolation assumes the image itself is trustworthy enough to
enter the hypervisor boundary. If malicious images are admitted, isolation is
reduced to damage containment rather than prevention.

Identity threads through all of these. Nova, Neutron, Cinder, and Glance rely
on Keystone-issued tokens and service credentials to authorize cross-service
calls. A compute request is never compute-only in practice; it is a chain of
identity-backed calls across multiple services.


Where compute isolation breaks down in real deployments
-------------------------------------------------------

Compute isolation fails most often at the seams between services, where each
component is behaving correctly in isolation but the end-to-end boundary is
weaker than operators assume.

The common starting point is a compromised workload in a tenant network. On
its own, that should be containable by security groups and project scoping.
Containment erodes when management and tenant-reachable paths are not strictly
separated. In a single-network deployment, a workload with outbound access can
target API ingress directly and probe for weak credentials, stale tokens, or
misconfigured policy. Keystone still enforces authentication, but the attack
surface is now broad enough that identity failures and network exposure become
part of compute isolation.

The higher-impact path starts from a compute-host or service compromise. Code
execution in a Nova service pod or on a compute node provides entry into the
cross-service propagation pattern described in
:doc:`Identity and Access Model </explanation/security/identity-and-access-model>`.
The compute-specific consequence is what an attacker can steer with that access:
volumes are detached from legitimate instances and reattached to
attacker-controlled ones, east-west security group rules are opened through
Neutron port operations, and boot images are replaced so subsequent instance
launches start compromised. The compute domain is both the probable entry point
and the surface where credential abuse produces its most visible operational
impact.

A second breakdown pattern is asymmetric policy enforcement between services.
Because each service holds its own oslo.policy configuration, the policies can
diverge across Sunbeam releases or between differently deployed service
versions. The result is that a request denied by one service may be accepted by
another in the same workflow. From the tenant perspective this appears as
inconsistent enforcement; from an attacker perspective it reveals the weakest
service as the pivot point.

A third pattern is delayed control-plane convergence under failure. OVN quorum
loss, message-bus disruption, or partial service restart can leave old flow
rules and authorization assumptions active longer than intended. During that
window, operators may believe a restrictive change is active while compute
nodes still enforce the previous state. Isolation exists in configuration but
not yet in dataplane behavior.

The practical model is that compute isolation is a moving boundary maintained
by continuous consistency across identity, network, storage, and control-plane
state. It does not fail only when a hypervisor is broken. It fails when service
interactions remain trusted after one participant is compromised.


Trust boundaries and cross-domain interactions
----------------------------------------------

Some boundaries in compute isolation are strong, some are conditional, and some
are operator-created.

Hypervisor enforcement is strong against ordinary tenant behavior but weak
against host compromise. Network segmentation can be strong when planes are
physically separated with Juju spaces and upstream controls; in flat networks
it is mostly logical and easier to bypass through misrouting or permissive
security groups. Service authentication with Keystone is cryptographically
strong for token validation but cannot distinguish a legitimate service account
from a stolen one.

Compute isolation therefore depends on adjacent domains:

* Identity and access: token lifetime, role scope, and service credential
  handling determine whether compromise remains local or becomes cloud-wide.
* Network exposure and ingress: if tenant paths can reach broad API surfaces,
  compute-originated attacks gain direct control-plane reach.
* Image and artifact trust: untrusted images import attacker logic directly
  into compute nodes and instance workflows.
* Storage protection: incorrect attachment controls or broad service grants
  convert a compute incident into cross-tenant data exposure.
* Host and substrate hardening: kernel, container runtime, and node baseline
  controls define whether a guest compromise can become a host compromise.

Isolation posture is only as strong as the weakest domain participating in
instance lifecycle operations.


Operational tradeoffs that shape isolation
------------------------------------------

Stricter isolation usually means less operational flexibility. Enforcing
tighter security groups reduces lateral movement but breaks ad hoc debugging
patterns where operators SSH broadly across tenant networks. Restricting volume
attachment paths and requiring explicit approvals for cross-project operations
improves tenant separation but slows incident response and migration workflows.

Short token lifetimes reduce the usefulness of stolen credentials in
cross-service compute workflows, but they require automation that refreshes
tokens reliably during long-running operations such as bulk instance builds.
Operators who harden token expiry without validating automation typically
discover failures as intermittent 401 responses in Nova or Cinder operations.

Physical plane isolation in MAAS-backed deployments materially improves compute
containment by separating management and tenant-reachable paths, but it adds
network design and lifecycle overhead that smaller teams may struggle to
maintain. Flat-network deployments reduce complexity at deployment time while
increasing the blast radius of later compromise.


Summary
-------

Compute isolation in Sunbeam is a system property, not a Nova-only feature. It
depends on synchronized enforcement across hypervisor controls, Neutron and OVN
network policy, Cinder attachment behavior, Glance image trust, and
Keystone-backed service authorization.

The key failure mode is not a single broken control. It is authenticated
cross-service propagation after one trusted component is compromised. Real
hardening work therefore focuses on limiting trust expansion: narrow service
credentials, tight network boundaries, controlled image sources, and operating
procedures that keep policy and runtime state aligned.

For related details, see:

* :doc:`Identity and Access Model </explanation/security/identity-and-access-model>`
* :doc:`Network Exposure and Ingress Model </explanation/security/network-exposure-and-ingress-model>`
* :doc:`Control Plane Integrity </explanation/security/control-plane-integrity>`
* :doc:`Storage Protection </explanation/security/storage-protection>`
* :doc:`Observability and Audit </explanation/security/observability-and-audit>`
* :doc:`Design considerations </explanation/design-considerations>`