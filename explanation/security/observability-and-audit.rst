Observability and Audit
=======================

Observability and audit in Canonical OpenStack is the control that turns
runtime behavior into defensible evidence. In Sunbeam, this is not one
logging feature. It is a chain across API middleware, service notifications,
message transport, Kubernetes workload logs, and operator workflows that
correlate events under incident pressure.

The domain matters because security decisions are only as strong as their
verifiability. A policy can be strict and a token can be valid, yet if
operators cannot reconstruct who did what, from where, and through which
service path, compromise remains operationally invisible until damage is
already systemic.


How this domain operates in Sunbeam
-----------------------------------

Most OpenStack API services in Sunbeam emit CADF-formatted audit events
through keystonemiddleware. A request typically generates an
``audit.http.request`` event and a corresponding ``audit.http.response`` event,
including initiator identity, request path, endpoint target, outcome, and
request identifiers that can be correlated across services.

Keystone is a special case. It does not emit those API middleware events in
the same way; it produces notification events for successful create, modify,
and delete operations. This means an operator investigating identity abuse must
combine two telemetry models: middleware request/response pairs for most
services and Keystone-specific action notifications for identity mutations.

At runtime, audit visibility depends on cross-service propagation of context.
Traefik ingress, Keystone token claims, Nova/Neutron/Cinder API handlers,
oslo.messaging notifications, and Kubernetes pod logs all contribute a piece
of the same transaction. The useful boundary is correlation quality: whether an
operator can move from a suspicious user action to all downstream effects in
compute, network, and storage workflows.

This boundary is partially strong and partially operator-dependent. Event
schemas are structured and reliable where emitted, but end-to-end retention,
normalization, and correlation across services are not guaranteed by default
hardening alone. Observability integrity therefore includes both generation and
operational handling of audit data.


Where observability and audit breaks down in real deployments
-------------------------------------------------------------

The main failure mode is not missing logs everywhere. It is missing continuity
across the moments that matter: rejection paths, degraded service windows,
and cross-service propagation after a trusted identity is abused.

A practical scenario starts with compromised service credentials. The attacker
obtains a valid token and performs actions that are individually legitimate:
querying Nova inventory, modifying Neutron security groups, detaching Cinder
volumes, and replacing Glance images. Each service may emit events for its own
surface. The investigative gap appears when these events are not retained or
correlated with consistent request identifiers and initiator context.
Operators see many valid actions, but cannot prove they are one attack chain.

A second scenario appears at the ingress and filter boundary. Some requests are
rejected by upstream filters before the audit middleware emits CADF events.
During token replay or spray activity, this creates an asymmetry: there is high
authentication pressure at the edge, but only partial service-level audit
coverage. If operators rely only on application audit streams and do not
correlate ingress and authentication signals, reconnaissance and credential
abuse attempts are undercounted.

A third scenario appears during incident recovery. Under queue backlog,
restarts, or partial network failure, services can process requests with
latency and retry behavior that reorders event timing. Without disciplined time
synchronization and durable log shipping, responders often reconstruct the
wrong sequence, applying containment to effects instead of causes. The cloud
returns to service, but the root compromise path remains unresolved and reused.

The central operational problem is therefore evidence integrity under stress.
If audit data is fragmented by service boundaries, rejected-path blind spots,
or retention inconsistency, the environment remains observable in aggregate but
not attributable in detail. That is exactly where persistent attackers survive.


Trust boundaries and domain interactions
----------------------------------------

Observability relies on trust boundaries in other domains and, in turn,
validates whether those domains are actually enforcing policy.

Identity and access supplies initiator claims and request scope. If identity
credentials are stolen, audit trails must distinguish normal service behavior
from misuse of trusted principals. Network exposure determines whether ingress
telemetry can be correlated with internal API and queue activity; weak
segmentation expands paths where actions occur without coherent attribution.

Control plane integrity determines whether event generation and transport remain
consistent during failures. If queue, database, or orchestration state drifts,
audit records may still be produced while representing stale or reordered
control flow. Compute isolation and storage protection depend on this domain to
prove whether a policy breach is isolated or has propagated to volume,
snapshot, image, or east-west traffic state.

The trust boundary here is not only "who can call an API." It is "who can
change, suppress, delay, or orphan evidence." That includes workloads with
broad namespace access, operators with excessive log-system permissions, and
retention pipelines that fail open during pressure.


Operational tradeoffs
---------------------

Deep audit capture and long retention improve forensic confidence, but increase
cost, pipeline pressure, and noise. Teams that retain everything without
correlation strategy often end with high-volume telemetry and low incident
clarity.

Strict normalization and central shipping improve integrity across services, but
they introduce dependency on shared logging infrastructure that can become a
single operational bottleneck during incidents. Lightweight per-service logging
reduces coupling, but weakens cross-service attribution when compromise spans
multiple APIs.

Real-time alerting on suspicious patterns improves detection speed, yet overly
sensitive rules can desensitize operators and suppress useful signal. Practical
hardening requires tuning around real control-plane behavior, not generic rule
packs.


Summary
-------

In Sunbeam, observability and audit is a security boundary because it decides
whether trusted actions remain attributable as they cross identity, network,
compute, and storage domains. The dominant risk is evidence fragmentation
during exactly the conditions when attackers rely on ambiguity.

Effective posture combines structured event generation, cross-service
correlation, retention discipline, and incident runbooks that account for
rejected-path blind spots and degraded-time ordering. Without that, the cloud
can be functional and still be unknowable.

For related details, see:

* :doc:`API auditing </reference/api-auditing>`
* :doc:`Identity and Access Model </explanation/security/identity-and-access-model>`
* :doc:`Control Plane Integrity </explanation/security/control-plane-integrity>`
* :doc:`Network Exposure and Ingress Model </explanation/security/network-exposure-and-ingress-model>`
* :doc:`Compute Isolation </explanation/security/compute-isolation>`
* :doc:`Storage Protection </explanation/security/storage-protection>`
* :doc:`Secrets and Key Management </explanation/security/secrets-and-key-management>`