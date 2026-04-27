Security reference
==================

Structured lookup material for the security surface of a Canonical OpenStack
(Sunbeam) deployment: exact ports, protocols, cryptographic requirements,
encryption coverage, and key management responsibilities.

**When to use this section:**

* You need to identify which port a service uses or whether it must be
  network-restricted.
* You need to confirm whether a communication path is encrypted by default
  or requires operator configuration.
* You need the exact cryptographic requirements for TLS endpoints.
* You need to identify key storage location or management ownership for
  audit or compliance purposes.
* You need to confirm which roles and token types a service enforces, or
  how service account grants are structured.
* You need to verify how service-to-service trust is provisioned or which
  cross-service call paths exist.
* You need to understand access control, isolation, and authentication for
  MySQL or RabbitMQ.
* You need to enumerate all credential types, their storage paths, and
  revocation capability.
* You need to identify which events are audited, where logs are written,
  and what coverage gaps exist.

**When to use other sections instead:**

* To understand how security domains relate and how failures propagate, see
  :doc:`Security explanation </explanation/security/index>`.
* To configure TLS, rotate certificates, or apply hardening steps, see
  :doc:`Security how-to </how-to/security/index>`.


Available reference documents
-------------------------------

:doc:`Ports and protocols <ports-and-protocols>`
  All TCP/UDP ports by traffic category, with normative encryption status
  and restriction requirements.

:doc:`Certificates and TLS <certificates-and-tls>`
  Certificate types, TLS status per component, cryptographic requirements,
  CA models, and certificate lifecycle.

:doc:`Encryption and data protection <encryption-and-data-protection>`
  Encryption coverage for all transit and at-rest paths, key storage
  locations, and key management ownership.

:doc:`Identity and authorization <identity-and-authorization>`
  Keystone token types, Wallaby role model, service account grants, policy
  enforcement points, and authorization gaps.

:doc:`Service-to-service trust <service-to-service-trust>`
  Trust provisioning, service token enforcement per service, cross-service
  call paths, OVN certificate model, and Juju trust model.

:doc:`Database and messaging security <database-and-messaging-security>`
  MySQL and RabbitMQ access control, per-service database and vhost
  isolation, user privilege scope, and operational access model.

:doc:`Secrets and credential handling <secrets-and-credential-handling>`
  All credential types, storage locations, persistence, revocation
  capability, management systems, and credential delivery path.

:doc:`Observability and audit coverage <observability-and-audit-coverage>`
  Event categories, log destinations, retention defaults, CADF field
  reference, notification consumers, and audit coverage gaps.


.. toctree::
   :maxdepth: 1
   :hidden:

   certificates-and-tls
   database-and-messaging-security
   encryption-and-data-protection
   identity-and-authorization
   observability-and-audit-coverage
   ports-and-protocols
   secrets-and-credential-handling
   service-to-service-trust
