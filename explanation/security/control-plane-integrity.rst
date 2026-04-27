Control plane integrity
=======================

Control plane integrity in Canonical OpenStack is the assurance that control
decisions remain authentic, consistent, and enforceable while the cloud is
under routine change and failure. In Sunbeam, that assurance is produced by
several systems acting together: API services in Kubernetes, Keystone identity,
RabbitMQ messaging, MySQL state, Juju orchestration, and the substrate services
that keep nodes and networking alive.

The practical point is simple. If integrity is lost in one control-plane
component, other components can keep accepting bad state as legitimate work.
The impact is usually not immediate outage. It is trusted drift: the cloud does
what it is told, but what it is told is no longer aligned with operator intent.


How control plane integrity operates in Sunbeam
-----------------------------------------------

Sunbeam runs OpenStack APIs as Kubernetes workloads and exposes them through
Traefik. Keystone validates caller identity and issues tokens. Services then
perform orchestration through message queues and database updates, while Juju
relations provide credentials, endpoint information, and lifecycle wiring.

This creates a layered integrity model.

The identity layer ensures that API and service requests carry valid
authorization claims. The service layer translates those requests into actions
such as scheduling instances, wiring Neutron ports, creating Cinder attachments,
or registering Glance artifacts. The state layer records intended and observed
outcomes in databases and queue backlogs. The orchestration layer keeps
configuration synchronized across units and charms as services change.

Each layer can look healthy while another is drifting. A token can be valid,
the API can return success, and yet downstream queue consumers may apply stale
or inconsistent policy assumptions. Control plane integrity depends less on any
single "secure component" and more on whether these layers remain mutually
consistent under pressure.


Where control plane integrity breaks down in real deployments
-------------------------------------------------------------

The most damaging failures start when one trusted control-plane identity is
compromised and its actions propagate through normal service paths.

The credential mechanics of this pattern are described in
:doc:`Identity and Access Model </explanation/security/identity-and-access-model>`.
What is specific to the control plane is how damage propagates once a principal
is compromised. Queue messages submitted to RabbitMQ carry scheduling and
attachment work consumed by agents on compute nodes and storage services.
Database entries in MySQL persist state that every subsequent API call reads and
acts upon. Juju relation data distributes configuration changes across service
units. Each of these channels accepts messages from a trusted identity and
executes them faithfully. A compromised credential does not need to break any
check — it exploits the reliability that makes the control plane function under
normal conditions. Network policy changes, storage movements, and image
operations follow, because the control plane was designed to act consistently on
what it receives from trusted participants.

A second breakdown pattern is partial convergence after operational disruption.
During upgrades, node maintenance, or transient message-bus instability,
operators may see API success while control-plane consumers lag or replay
backlogs. In that interval, one service may evaluate current policy while
another acts on stale state. The cloud appears functional and eventually
converges, but the temporary mismatch can create durable side effects: overly
permissive security group outcomes, volume attachments that outlast expected
ownership, or inventory state that no longer reflects effective access.

A third pattern is emergency exception debt. During incidents, teams broaden
permissions, bypass strict checks, or reuse high-privilege credentials to
restore service quickly. Those decisions are often rational in the moment.
Integrity erodes when they persist. The next compromise no longer needs to
break a control; it uses the emergency pathway as a routine path.

The central operational truth is that control plane integrity fails gradually,
then suddenly. It drifts through accepted exceptions, stale convergence, and
over-trusted service identities before it appears as an obvious security event.


Trust boundaries that matter most
---------------------------------

Some boundaries in this domain are strong, some are conditional, and some are
absent until operators create them.

Keystone token signing and validation are strong cryptographic controls, but
they verify who signed the request, not whether that identity has been stolen.
RabbitMQ and MySQL boundaries are strong only when network reachability and
credential scope are tight; broad intra-cluster reach turns them into weak
shared infrastructure boundaries. Juju relation data provides deterministic
configuration flow, but any compromise of relation consumers can turn that same
flow into a large-scale distribution channel for bad configuration or leaked
secrets.

Network design defines whether control-plane services are insulated from
tenant-reachable paths. In flat or weakly segmented deployments, compromise in
one zone gives practical reach to multiple control-plane APIs and internal
services. In MAAS-backed multi-plane deployments with well-enforced spaces,
that same compromise usually encounters routed boundaries and explicit policy
controls before reaching management surfaces.


Cross-service interactions with adjacent security domains
---------------------------------------------------------

Control plane integrity is the dependency behind other security domains.

Identity and access determines whether control actions start from valid and
appropriately scoped principals. Network exposure determines who can reach
control-plane entry points and internal service paths. Compute isolation depends
on control-plane correctness to keep scheduling, networking, and policy
enforcement coherent. Storage protection depends on the integrity of attachment,
detachment, snapshot, and restore workflows coordinated by trusted services.

Failures therefore stack. A network exposure issue without identity abuse may be
containable. Identity abuse without queue or database influence may be short.
Combined failures turn into durable integrity loss because each service treats
the others as trusted participants in the same control loop.


Operational tradeoffs
---------------------

Strict integrity controls increase day-2 effort. Narrow service credentials,
short token lifetimes, and tighter internal network policy reduce compromise
propagation, but they also increase the chance of brittle automation during
upgrades and incident response unless runbooks are updated and tested.

High availability settings reduce outage risk, yet they raise consistency
complexity. More replicas and failover paths improve resilience, but they also
increase the number of places where partial convergence can appear and where
operators must validate that policy and state are synchronized.

Emergency access procedures are unavoidable in production. The trade is whether
they are temporary, audited, and reversible, or convenient shortcuts that become
permanent architecture. The former preserves integrity under stress. The latter
trades short-term recovery speed for long-term compromise survivability.


Summary
-------

Control plane integrity in Sunbeam is sustained by consistency across identity,
orchestration, messaging, state, and network boundaries. The dominant risk is
trusted propagation after one principal or component is compromised, not an
obvious cryptographic failure.

Hardening this domain means reducing how far trust can travel: limit credential
scope, enforce internal segmentation, keep convergence visible during change,
and treat emergency exceptions as expiring controls that must be removed.

For related details, see:

* :doc:`Identity and Access Model </explanation/security/identity-and-access-model>`
* :doc:`Network Exposure and Ingress Model </explanation/security/network-exposure-and-ingress-model>`
* :doc:`Compute Isolation </explanation/security/compute-isolation>`
* :doc:`Storage Protection </explanation/security/storage-protection>`
* :doc:`Secrets and Key Management </explanation/security/secrets-and-key-management>`
* :doc:`Observability and Audit </explanation/security/observability-and-audit>`