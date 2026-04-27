Secrets and key management
==========================

Secrets and key management in Canonical OpenStack is the discipline of keeping
the cloud's trust material valid, scoped, and recoverable under change. In
Sunbeam, this material spans service passwords, database credentials, message
bus credentials, token-signing keys, TLS private keys, and certificate chains
distributed across multiple services.

This domain is not a single control. It is a supply chain of trust artifacts
moving through Juju relations, Juju Secrets, service configuration, and
runtime memory. The security question is not only "is the secret encrypted"
but "who can obtain it, where can it be replayed, and how quickly can it be
rotated without breaking service behavior".


How this domain operates in Sunbeam
-----------------------------------

Sunbeam provisions and wires secrets primarily through Juju and charms. Service
credentials are generated and exchanged over relation data, then rendered into
service configuration for workload consumption. Keystone issues and validates
tokens using rotating Fernet keys. Traefik TLS termination relies on
certificate and key material supplied through the configured TLS path, either
third-party CA integration or Vault-backed intermediary signing workflows.

At runtime, most control-plane workflows are secret-dependent cross-service
calls. Nova uses credentials to call Keystone, Neutron, Cinder, and RabbitMQ.
Cinder and Glance use credentials to reach backend and control services. Each
component can be healthy in isolation while the secret lifecycle between them is
degraded.

Trust boundaries in this model are uneven. Cryptographic validation of tokens
and certificates is strong where keys are current and trust stores align.
Distribution boundaries are weaker: secret material may exist simultaneously in
Juju state, rendered files, and process environments. Integrity of the full
chain depends on reducing unnecessary copies and constraining where those
copies are readable.


Where secrets and key management breaks down in real deployments
----------------------------------------------------------------

The most common failure is not key generation weakness. It is lifecycle drift:
secrets are created correctly, then age, spread, and outlive the controls that
were meant to contain them.

A realistic starting point is compromise of one control-plane pod. Service
credentials are rendered into the pod environment by the Juju charm, so code
execution is sufficient to extract a service password. The cross-service
propagation that follows is described in
:doc:`Identity and Access Model </explanation/security/identity-and-access-model>`.
The question specific to this domain is different: why was the credential still
valid when extracted, where else did it exist in plaintext, and can it be
rotated before it is replayed?

Rotation often fails in ways that look like reliability incidents instead of
security incidents. During certificate renewal on Traefik or CA chain updates,
some services can continue using stale trust roots while others switch to new
roots. Operators then observe intermittent 401 and 503 behavior and focus on
availability triage. In that window, emergency rollback or temporary trust
relaxations are common. If those changes persist, the deployment ends with a
weaker trust model than before the rotation started.

Fernet key rotation has a similar pattern. If key distribution across Keystone
units is uneven, token validation behavior fragments across API paths. Clients
and services retry, operators widen exceptions, and stale assumptions stay in
place longer than intended. The immediate symptom is authentication instability;
the lasting effect can be extended acceptance of credentials and unclear
revocation guarantees.

The most damaging propagation appears when secret handling practices overlap
with incident pressure. Teams under outage conditions commonly reuse broad
admin credentials in automation, copy secrets into ad hoc scripts, or bypass
normal approval to restore service quickly. Those actions solve the page but
create durable credential sprawl. The next compromise does not need a new
vulnerability; it finds old high-privilege material still accepted across
multiple services.

The center of gravity in this domain is therefore operational: can the
deployment rotate, revoke, and redistribute trust material without creating new
long-lived secret exposure paths.


Trust boundaries and cross-domain interactions
----------------------------------------------

Secrets and key management sits at the intersection of identity, network,
control-plane integrity, and storage protection.

Identity depends on key custody. Keystone policy and token semantics only hold
if service credentials and signing keys remain controlled. Network exposure
changes the secret threat model directly: if tenant-reachable or weakly
segmented paths can reach management and control services, secret replay becomes
a practical attack path rather than a theoretical one.

Control plane integrity depends on secret freshness and scope. A compromised
credential allows queue submissions and API actions that look legitimate to
downstream services. Storage and compute boundaries then inherit that trust and
execute side effects as normal workflows. What looks like an identity problem
quickly becomes a storage or compute incident because service interactions are
credential-mediated.

This is why strong cryptography alone is insufficient. The relevant boundary is
where secrets are stored, copied, rotated, and consumed across service
interactions.


Operational tradeoffs
---------------------

Aggressive rotation reduces credential lifetime and replay value, but it raises
operational fragility unless rotation automation and rollback paths are tested
against real upgrade and failure workflows. The practical trade is outage risk
from brittle rotation versus compromise survivability from stale credentials.

Centralized secret issuance through Vault-style workflows improves governance
and auditability, but introduces dependency and latency during bootstrap,
recovery, and certificate renewal events. Simpler manual or semi-manual
certificate processes reduce platform dependencies but increase human handling
of private key material and broaden error surfaces.

Least-privilege service credentials reduce blast radius, yet they require
service-by-service review as new features and integrations are enabled. Teams
that avoid this maintenance cost typically converge on broad credentials that
work everywhere and fail safely nowhere.


Summary
-------

In Sunbeam, secrets and key management is a continuous control process, not a
one-time setup task. The dominant risk is trusted cross-service propagation
after secret exposure, combined with lifecycle drift introduced during normal
operations.

Effective hardening focuses on custody and lifecycle: minimize where secrets
exist, keep scope narrow, rotate predictably, validate convergence after every
rotation, and treat emergency credential workarounds as temporary controls with
explicit expiry.

For related details, see:

* :doc:`Identity and Access Model </explanation/security/identity-and-access-model>`
* :doc:`Control Plane Integrity </explanation/security/control-plane-integrity>`
* :doc:`Network Exposure and Ingress Model </explanation/security/network-exposure-and-ingress-model>`
* :doc:`Observability and Audit </explanation/security/observability-and-audit>`
* :doc:`Service endpoint encryption </explanation/service-endpoint-encryption>`