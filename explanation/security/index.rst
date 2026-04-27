Security
========

Security in Canonical OpenStack is not a collection of per-service settings.
It is a system property: the effective security posture of a deployment depends
on how its components behave together under normal conditions and under failure.
Understanding Sunbeam security means understanding where the real boundaries are,
where they are absent by default, and how a failure in one part of the system
propagates to others.

The documentation in this section is organized around security domains rather
than services. Each domain cuts across multiple components and describes a
distinct set of boundaries, risks, and tradeoffs. The domains are
interdependent — most failures of practical significance involve more than one.


How these domains connect
--------------------------

The domains form a dependency structure. Identity is the load-bearing domain:
every service interaction in Canonical OpenStack is mediated by a Keystone
token or service credential, which means identity failures have the widest
blast radius. Network exposure determines who can reach which entry points,
shaping how identity failures translate into practical attack paths. Secrets
and key management describes the lifecycle of the credentials that identity
depends on — if custody or rotation fails, the identity model breaks regardless
of how well it is configured.

Compute isolation, storage protection, and control plane integrity sit above
these foundations. They describe what an attacker can accomplish once entry is
established through an identity or network failure: which workloads can be
disrupted, which data can be accessed or moved, and how far operational control
can be subverted before the deployment becomes aware of the problem.

Observability and audit closes the loop. It describes what evidence exists to
detect and attribute cross-domain failures, and where that evidence chain
breaks under exactly the conditions when attackers rely on ambiguity.


Security domains
-----------------

:doc:`Network exposure and ingress <network-exposure-and-ingress-model>`
defines the physical and logical boundaries that determine who can reach which
services. In a default Sunbeam installation, many of these boundaries exist
only in configuration, not in physical separation — a distinction that directly
determines the blast radius of any subsequent compromise.

:doc:`Identity and access <identity-and-access-model>` describes how Keystone
tokens and service credentials authorize every API call in the deployment. It
explains the propagation model that makes service credential compromise so
consequential, and the policy and token-lifetime decisions that determine how
far a failure can travel. This document is the primary reference for
understanding cross-service compromise.

:doc:`Compute isolation <compute-isolation>` covers the boundaries that keep
workloads from interfering with one another or with the control plane through
the compute path. Isolation here is a system property across Nova, Neutron,
Cinder, Glance, and Keystone — not a hypervisor feature — and it fails most
often at the seams between those services.

:doc:`Storage protection <storage-protection>` addresses the controls that
prevent unauthorized access to block storage data across Cinder, Nova, and
Ceph. The primary risk is trusted automation running with broader authority
than intended, especially during recovery operations when operators widen
permissions to restore service quickly.

:doc:`Secrets and key management <secrets-and-key-management>` describes how
credentials, signing keys, and certificates are provisioned, distributed,
rotated, and consumed across Juju relations, Juju Secrets, and service
configuration. The risk is lifecycle drift: secrets that are created correctly
but age, spread, and outlive the controls meant to contain them.

:doc:`Control plane integrity <control-plane-integrity>` explains how the
orchestration, messaging, and state layers remain consistent under change and
failure. A compromised service identity can steer normal RabbitMQ, MySQL, and
Juju workflows toward unintended outcomes without any component producing a
visible error — which makes this domain the one where failures are most
difficult to distinguish from routine operations.

:doc:`Observability and audit <observability-and-audit>` covers the chain from
CADF event generation through retention, correlation, and incident attribution.
Security decisions are only as strong as their verifiability; this domain
explains where that chain is reliable and where it is absent or fragmented by
default.


Where to start
---------------

Operators new to this section should begin with
:doc:`Network exposure and ingress <network-exposure-and-ingress-model>` and
:doc:`Identity and access <identity-and-access-model>`. Network topology and
identity are decided at or shortly after deployment time, and most of the risks
described in the other domains depend on how those two are configured.

Operators hardening a running deployment or investigating an incident will find
:doc:`Control plane integrity <control-plane-integrity>` and
:doc:`Observability and audit <observability-and-audit>` most directly
relevant: the former explains how compromise propagates through trusted service
paths, the latter how it can be detected and attributed before damage becomes
systemic.

.. toctree::
   :maxdepth: 1
   :hidden:

   control-plane-integrity
   compute-isolation
   identity-and-access-model
   network-exposure-and-ingress-model
   observability-and-audit
   secrets-and-key-management
   storage-protection
