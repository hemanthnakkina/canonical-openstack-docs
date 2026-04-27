Certificates and TLS
====================

This page defines certificate types, TLS termination points, supported
versions, certificate authority models, and certificate lifecycle expectations
for a Canonical OpenStack (Sunbeam) deployment.

TLS is not enabled by default. All entries marked **operator-configured**
require explicit setup before taking effect. See
:doc:`Service endpoint encryption </explanation/service-endpoint-encryption>`
for configuration guidance.


Certificate types
------------------

.. list-table::
   :header-rows: 1
   :widths: 22 20 58

   * - Certificate type
     - Scope
     - Description
   * - Traefik ingress certificate
     - External
     - Terminates TLS for all OpenStack API endpoints. Single certificate
       covers the full API surface via Traefik path routing. Operator-configured.
   * - OVN inter-component certificate
     - Internal
     - Secures OVSDB connections between OVN Northbound/Southbound DB servers
       and ovn-controller agents on compute nodes. Issued per-component.
       Enabled by default within MicroOVN.
   * - Juju controller certificate
     - Control plane
     - Secures the Juju controller API (port 17070). Issued at bootstrap.
       Rotated by Juju internally.
   * - MicroK8s / k8s API certificate
     - Control plane
     - Secures the Kubernetes API server (port 16443). Issued and managed by
       MicroK8s. Includes the cluster CA.
   * - etcd peer and client certificates
     - Control plane
     - Mutual TLS between etcd members and between etcd and the Kubernetes API
       server. Managed by MicroK8s.
   * - sunbeamd / clusterd certificate
     - Control plane
     - Secures the MicroCluster membership API (port 7000). Issued at cluster
       bootstrap.
   * - Vault intermediate CA certificate
     - Internal (optional)
     - When Vault is configured as a PKI backend, it acts as an intermediate CA
       for issuing service certificates. Operator-configured.
   * - Ceph Dashboard certificate
     - Storage (optional)
     - TLS for the Ceph management web UI (port 8443). Operator-configured.
       Not required for normal Sunbeam operations.
   * - MAAS region controller certificate
     - Control plane (optional)
     - TLS for the MAAS API (port 5249). MAAS deployments only.


TLS termination points
-----------------------

.. list-table::
   :header-rows: 1
   :widths: 25 20 20 35

   * - Component
     - Termination point
     - TLS status
     - Notes
   * - External OpenStack APIs
     - Traefik ingress
     - Required (operator-configured)
     - Single certificate covers the full API surface. Traffic between Traefik
       and backend pods is plaintext; no pod-level TLS.
   * - OVN Northbound DB (6641)
     - ovn-central pod
     - Required (default)
     - Mutual TLS managed by MicroOVN. Enabled at cluster bootstrap.
   * - OVN Southbound DB (6642)
     - ovn-central pod
     - Required (default)
     - Mutual TLS managed by MicroOVN. Enabled at cluster bootstrap.
   * - Kubernetes API (16443)
     - MicroK8s API server
     - Required
     - Certificate managed by MicroK8s. Always active.
   * - Juju controller API (17070)
     - Juju controller
     - Required
     - Certificate issued at bootstrap. Always active.
   * - sunbeamd / clusterd (7000)
     - sunbeamd process
     - Required
     - Certificate issued at cluster bootstrap. Always active.
   * - RabbitMQ (5672)
     - RabbitMQ pod
     - Not used
     - Plaintext AMQP. Not configurable in Sunbeam defaults. NetworkPolicy
       restriction is the required compensating control.
   * - MySQL (3306)
     - MySQL pod
     - Not used
     - Plaintext. Not configurable in Sunbeam defaults. NetworkPolicy
       restriction is the required compensating control.
   * - Ceph Monitor / OSD
     - Ceph nodes
     - Not used
     - msgr2 does not encrypt by default. msgr2 secure mode is
       operator-configured.


Cryptographic requirements
--------------------------

Protocol versions
~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 30 20 50

   * - Protocol version
     - Status
     - Requirement
   * - TLS 1.3
     - Preferred
     - Enable where component support exists (Traefik, modern OpenSSL/Go
       stacks).
   * - TLS 1.2
     - Required minimum
     - Minimum allowed protocol for all operator-configured TLS endpoints.
   * - TLS 1.1
     - Disallowed
     - Must be disabled.
   * - TLS 1.0
     - Disallowed
     - Must be disabled.
   * - SSLv3 and earlier
     - Disallowed
     - Must be disabled.

Cipher and key exchange profile
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 30 25 45

   * - Control
     - Requirement
     - Accepted values
   * - Forward secrecy
     - Mandatory
     - ECDHE key exchange (TLS 1.2) or TLS 1.3 default key schedule.
   * - Bulk encryption algorithms
     - Mandatory
     - AEAD ciphers only: AES-GCM or ChaCha20-Poly1305.
   * - TLS 1.3 cipher suites
     - Mandatory set
     - ``TLS_AES_128_GCM_SHA256``, ``TLS_AES_256_GCM_SHA384``,
       ``TLS_CHACHA20_POLY1305_SHA256``.
   * - TLS 1.2 cipher suites
     - Allowed set
     - ``ECDHE-ECDSA-AES128-GCM-SHA256``,
       ``ECDHE-ECDSA-AES256-GCM-SHA384``,
       ``ECDHE-RSA-AES128-GCM-SHA256``,
       ``ECDHE-RSA-AES256-GCM-SHA384``,
       ``ECDHE-ECDSA-CHACHA20-POLY1305``,
       ``ECDHE-RSA-CHACHA20-POLY1305``.
   * - Key exchange groups
     - Required baseline
     - ``X25519`` or ``secp256r1`` (P-256) minimum.

Certificate requirements
~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 35 25 40

   * - Certificate control
     - Requirement
     - Notes
   * - End-entity public key algorithm
     - Required
     - RSA (minimum 2048-bit) or ECDSA (P-256 or stronger).
   * - CA public key algorithm
     - Required
     - RSA (minimum 3072-bit preferred, 2048-bit minimum) or ECDSA (P-256 or
       stronger).
   * - Signature algorithm
     - Required
     - SHA-256 or stronger (for example ``sha256WithRSAEncryption``,
       ``ecdsa-with-SHA256``, SHA-384 variants).
   * - Certificate validity period
     - Required
     - Maximum 398 days for externally trusted ingress certificates; internal
       service certificates follow component defaults but must support regular
       rotation.
   * - Subject Alternative Name (SAN)
     - Required
     - All API FQDNs and IP endpoints in use must be present in SAN.

Deprecated and disallowed algorithms
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 35 65

   * - Protocol / algorithm
     - Status
   * - SSLv2, SSLv3, TLS 1.0, TLS 1.1
     - Disallowed.
   * - Static RSA key exchange (non-PFS)
     - Disallowed.
   * - RC4
     - Disallowed.
   * - 3DES (``DES-CBC3-SHA``)
     - Disallowed.
   * - AES-CBC suites without AEAD
     - Disallowed.
   * - MD5 or SHA-1 certificate signatures
     - Disallowed.
   * - RSA keys smaller than 2048-bit
     - Disallowed.
   * - Diffie-Hellman groups smaller than 2048-bit
     - Disallowed.


Certificate authority models
------------------------------

Two CA models are supported. They are mutually exclusive per deployment.

**External CA (TLS CA mode)**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Property
     - Value
   * - CA location
     - Operator-managed; provided as charm configuration
   * - Certificate issuance
     - Manual or operator-automated
   * - Scope
     - Traefik ingress certificate only
   * - Rotation
     - Manual; operator responsibility

**Vault PKI (TLS Vault mode)**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Property
     - Value
   * - CA location
     - Vault PKI secrets engine
   * - Certificate issuance
     - Automated via Vault charm relations
   * - Scope
     - Traefik ingress certificate; optionally service-level certificates
   * - Rotation
     - Automated; governed by TTL configured in Vault PKI role


Certificate lifecycle
----------------------

.. list-table::
   :header-rows: 1
   :widths: 20 20 20 40

   * - Certificate
     - Issuance
     - Rotation
     - Notes
   * - Traefik ingress (TLS CA mode)
     - Manual
     - Manual
     - Operator must redistribute new CA trust bundle to service pods after
       rotation. Pods holding the old CA will reject Traefik connections until
       updated.
   * - Traefik ingress (Vault mode)
     - Automated (Vault charm)
     - Automated (Vault TTL)
     - CA trust distribution window must complete before the old certificate
       expires. Pods still holding the old CA produce Nova 503 or Cinder
       connection errors during the window.
   * - OVN (MicroOVN)
     - Automated at cluster bootstrap
     - Automated by MicroOVN
     - Certificates stored on host filesystem under ``/var/snap/microovn``.
   * - Juju controller
     - Automated at bootstrap
     - Automated by Juju
     - Not operator-visible. Juju agents reconnect on rotation.
   * - MicroK8s / etcd
     - Automated at bootstrap
     - Automated by MicroK8s
     - ``microk8s refresh-certs`` command available for manual rotation.
   * - sunbeamd / clusterd
     - Automated at cluster bootstrap
     - Not currently automated
     - Certificate tied to cluster lifetime. Consult sunbeam documentation
       for renewal procedures.
   * - Ceph Dashboard
     - Manual
     - Manual
     - Operator responsibility. Not required for Sunbeam API operations.


Notes and operator responsibilities
-------------------------------------

**Traefik-to-backend traffic**
   TLS terminates at Traefik. Traffic between Traefik and OpenStack service
   pods travels as plaintext within the pod network. NetworkPolicy controls
   are the relevant boundary; none are applied by default.

**CA trust distribution during rotation**
   After a CA or certificate rotation, all service pods must receive the
   updated trust bundle before the old certificate expires. Incomplete
   distribution produces intermittent 401 or 503 errors on API paths.

**No mTLS between OpenStack services**
   Internal API calls between OpenStack services use plaintext HTTP within
   the pod network. No configuration path exists to enable service-to-service
   mTLS in Sunbeam defaults.

For related reference material, see:

* :doc:`Ports and Protocols </reference/security/ports-and-protocols>`
* :doc:`Encryption and Data Protection </reference/security/encryption-and-data-protection>`
* :doc:`Service endpoint encryption </explanation/service-endpoint-encryption>`
* :doc:`Secrets and Key Management </explanation/security/secrets-and-key-management>`
