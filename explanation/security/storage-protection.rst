Storage protection
==================

Storage protection in Canonical OpenStack is about controlling who can read,
write, attach, copy, snapshot, and recover data over time. In Sunbeam, this is
not only a Ceph question. The effective boundary is created jointly by Cinder,
Nova, Glance, Keystone, and the storage and management networks that carry
control and data traffic.

Operators usually experience storage as an API workflow: create volume, attach
volume, snapshot, back up, restore. Underneath, each step crosses service
boundaries. Cinder authorizes and orchestrates operations, Nova consumes those
operations during instance lifecycle changes, and Ceph enforces access on the
backend. Protection quality depends on whether those services stay consistent
under normal load and during failures.


How storage protection operates in Sunbeam
------------------------------------------

Sunbeam uses Cinder as the control plane for block storage and Ceph as the
default backend in storage-enabled deployments. Cinder API calls are
authenticated with Keystone tokens and translated into backend actions through
Cinder services and drivers. Nova requests attachment and detachment through
Cinder when instances start, stop, migrate, or recover. The design intent is
clear separation: tenants control only resources in their project scope, while
service accounts perform privileged cross-service work on their behalf.

Three different boundaries are active at once.

The first is identity scope. Keystone roles and project scoping decide whether
an operator or workload can issue a Cinder action. This boundary is strong when
policies remain aligned with secure RBAC defaults and service credentials are
well controlled.

The second is control-plane isolation. Cinder, Nova, and messaging/database
services communicate over internal paths inside the cluster and management
fabric. This boundary is only as strong as network segmentation and pod-level
reachability controls.

The third is backend data-path authorization. Ceph client credentials determine
which pools and objects services can reach. Even when an API request is valid,
backend credentials decide whether data can actually be read or modified.

Storage protection is therefore a chained property: API authorization, service
orchestration, and backend authorization must all agree. If one link is weak,
the whole chain is weak.


Where storage protection breaks down in real deployments
--------------------------------------------------------

Storage protection fails in production less often through cryptographic breakage
and more often through trusted automation running with broader authority than
operators intended.

A common starting point is service credential exposure after a pod compromise.
The credential extraction and cross-service propagation mechanics are described
in :doc:`Identity and Access Model </explanation/security/identity-and-access-model>`.
What is specific to storage is the impact on data: volumes are detached from
legitimate instances and reattached elsewhere, snapshots are exported outside
the project that owns them, and backup artifacts persist in locations that
outlive the original resource controls. Ceph backend operations proceed normally
because every request arrives authenticated. No component in the chain produces
an error.

The impact becomes concrete when attachment control and project boundaries drift
out of alignment. Operators relying on broad admin tokens for operational
tooling can allow volume reattachment or snapshot export paths that were assumed
to be restricted. The compromise then shifts from
service compromise to tenant data exposure: data appears in the wrong instance,
in the wrong project, or in exported artifacts that outlive the original
resource controls.

A second failure path appears during recovery operations. Under node failure or
maintenance pressure, teams prioritize service restoration speed and temporarily
widen permissions for migration, rebuild, or forced detach actions. Those
temporary exceptions are rarely tracked with the same rigor as production
policy. If they remain in place, the steady-state storage boundary silently
changes. Later compromises do not need a new vulnerability; they exploit the
already-expanded operating model.

A third breakdown is control-plane inconsistency. Messaging disruption,
partial service restarts, or backend degradation can leave operator intent and
runtime state out of sync. Cinder and Nova may disagree briefly about attachment
state, while Ceph still serves the last committed backend mapping. In that
window, emergency operational actions taken to "unstick" workloads can bypass
normal review paths and create data handling outcomes that violate tenant
separation, even though no service is visibly down.

The center of gravity for storage protection is therefore not only encryption at
rest or token validation in isolation. It is whether cross-service storage
workflows keep tenant intent intact when services are degraded, credentials are
stolen, or operators are under time pressure.


Trust boundaries across identity, network, and storage services
---------------------------------------------------------------

Some storage boundaries are strongly enforced, some are conditional, and some
exist only if operators create them.

Keystone token validation is a strong cryptographic check of request identity,
but it cannot distinguish legitimate service use from abuse of stolen service
credentials. Cinder policy is a strong boundary when defaults are preserved,
yet it becomes conditional if per-service policies diverge from project-wide
assumptions across Sunbeam releases. Ceph backend authorization is strong for direct data
path access, but those guarantees are weakened when overly broad backend
credentials are distributed to control-plane components.

Network boundaries shape all of the above. If storage and management traffic are
not adequately segmented, workloads or compromised pods gain more direct reach
to APIs and internal services that orchestrate volume lifecycle. Identity and
network posture therefore determine whether a storage incident remains local or
becomes deployment-wide.

Cross-service interactions are the rule, not the exception:

* Nova attach/detach workflows rely on Cinder decisions and service trust.
* Glance image import and volume-from-image flows tie artifact trust directly
  to storage integrity.
* RabbitMQ and MySQL carry control-plane state whose compromise can alter how
  storage actions are queued, replayed, or reconciled.


Operational tradeoffs that decide real outcomes
-----------------------------------------------

The strictest storage posture is rarely the easiest to operate. Tight policy,
short-lived credentials, and explicit approval for cross-project storage actions
reduce blast radius, but they slow incident handling when workloads are already
degraded and operators need rapid migration or recovery.

Segregating storage and management paths in MAAS-backed network designs improves
containment materially, yet it increases lifecycle complexity for node addition,
hardware replacement, and day-2 troubleshooting. Flat-network deployments are
simpler to run initially and often acceptable for trusted environments, but they
expand how far a storage-control compromise can propagate.

Frequent credential rotation and narrow service grants limit the usefulness of
stolen identities, but they require automation that handles rotation cleanly
during long-running storage operations. Teams that harden credential policy
without validating operational runbooks usually trade silent over-permissioning
for visible instability. The practical target is not maximum strictness in one
domain; it is a posture that remains strict under routine failure handling.


Summary
-------

Storage protection in Sunbeam is a system behavior across Cinder, Nova, Ceph,
Keystone, and the networks that connect them. The primary risk is trusted,
cross-service propagation after one component or credential is compromised,
especially during recovery periods when operators widen controls to restore
service quickly.

Reliable hardening comes from maintaining consistency across domains: preserve
RBAC intent, constrain service credentials, segment storage-relevant network
paths, and treat temporary operational exceptions as security changes that must
be removed deliberately.

For related details, see:

* :doc:`Identity and Access Model </explanation/security/identity-and-access-model>`
* :doc:`Network Exposure and Ingress Model </explanation/security/network-exposure-and-ingress-model>`
* :doc:`Compute Isolation </explanation/security/compute-isolation>`
* :doc:`Control Plane Integrity </explanation/security/control-plane-integrity>`
* :doc:`Secrets and Key Management </explanation/security/secrets-and-key-management>`
* :doc:`Observability and Audit </explanation/security/observability-and-audit>`