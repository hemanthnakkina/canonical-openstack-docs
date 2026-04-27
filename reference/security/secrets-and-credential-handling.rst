Secrets and credential handling
===============================

Types, storage locations, persistence, revocation capabilities, and
management systems for all credentials and secrets in a Canonical OpenStack
(Sunbeam) deployment.

For key management and at-rest encryption, see
:doc:`Encryption and data protection <encryption-and-data-protection>`.
For role and token model, see
:doc:`Identity and authorization <identity-and-authorization>`.


Credential inventory
---------------------

.. list-table::
   :header-rows: 1
   :widths: 22 22 18 16 22

   * - Credential type
     - Storage location
     - Persistence
     - Revocable
     - Management system
   * - Keystone service account password
     - Juju Secret (relation-scoped grant)
     - Persistent (survives pod restart)
     - Yes (manual; charm action)
     - Juju / charm relation data
   * - MySQL database password (per service)
     - Juju Secret (relation-scoped grant)
     - Persistent
     - Yes (manual; charm action)
     - Juju / charm relation data
   * - RabbitMQ AMQP password (per service)
     - Juju Secret (relation-scoped grant)
     - Persistent
     - Yes (manual; charm action)
     - Juju / charm relation data
   * - Keystone Fernet keys
     - Keystone MySQL database (``keystone`` table)
     - Persistent; rotated on schedule
     - Yes (key rotation invalidates tokens signed by removed keys)
     - Keystone auto-rotation (default 1 day)
   * - TLS private keys (service endpoints)
     - Juju Secret or operator-provided secret backend (deployment-dependent)
     - Persistent; renewed before expiry
     - Yes (certificate revocation via CA; not enforced at runtime)
     - Sunbeam cert-manager / external CA
   * - OVN component certificates and keys
     - MicroOVN on each node (``/var/snap/microovn/``)
     - Persistent; rotated automatically by MicroOVN
     - Not practical at runtime; rotation is the mechanism
     - MicroOVN
   * - Ceph keyrings (per service)
     - Juju relation data → Juju Secret
     - Persistent
     - Yes (manual; ceph auth rm + charm reconfiguration)
     - Ceph / charm relation data
   * - Juju unit agent secret
     - Juju controller database
     - Persistent; set at unit deployment
     - Yes (``juju remove-unit`` revokes agent credential)
     - Juju controller
   * - Juju controller certificate and CA
     - Juju controller (``~/.local/share/juju/``)
     - Persistent; renewed by Juju
     - Yes (``juju regenerate-controller-cert``)
     - Juju
   * - MicroK8s API server certificate
     - ``/var/snap/microk8s/`` on each node
     - Persistent
     - Requires node re-join or manual rotation
     - MicroK8s
   * - MicroCluster peer certificates
     - ``/var/snap/openstack/`` on each node
     - Persistent; renewed automatically by MicroCluster
     - Not revocable at runtime; requires node removal
     - MicroCluster
   * - Barbican-managed secrets
     - Barbican database (metadata) + backend (payload)
     - Persistent unless explicitly deleted
     - Yes (``openstack secret delete``)
     - Barbican; operator-controlled backend
   * - Application credentials
     - Keystone MySQL database
     - Persistent; no default expiry
     - Yes (``openstack application credential delete``)
     - Keystone; user-managed
   * - Heat trust tokens
     - Keystone MySQL database
     - Persistent until stack is deleted
     - Yes (``openstack trust delete``); stack deletion removes trust
     - Heat / Keystone


Juju Secret exposure
---------------------

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - Property
     - Value
   * - Secret scope
     - Juju model ``openstack``
   * - Access control
     - Juju secret grants: only applications granted access by Juju relations
       can fetch secret values through charm code paths.
   * - Encryption at rest
     - Not enabled by default. MicroK8s stores etcd data on disk without
       encryption. Requires operator configuration of KMS-backed etcd
       encryption.
   * - Runtime projection
     - Credentials are rendered into service configuration and environment
       variables by charm handlers. Environment variables are visible in
       ``/proc/PID/environ`` on the node to any process with the same
       effective UID or root access.


Credential delivery path
--------------------------

.. list-table::
   :header-rows: 1
   :widths: 28 22 50

   * - Stage
     - Actor
     - Detail
   * - 1. Credential generation
     - Charm (Python)
     - Juju charm generates or retrieves a credential (e.g., a random
       password or CSR response from Keystone).
   * - 2. Relation data distribution
     - Juju relation bus
     - Credential is placed in Juju relation data. Juju encrypts relation
       data in transit via TLS to the controller.
   * - 3. Juju Secret creation
     - Charm
     - Charm stores credential as a Juju Secret and grants access via the
       corresponding relation.
   * - 4. Runtime rendering
     - Charm + workload manager
     - Charm retrieves the Juju Secret and renders it into service runtime
       configuration or environment variables at pod start.
   * - 5. Service use
     - Service process
     - Service reads credential from environment or file at runtime.
       Credential is not re-fetched during pod lifetime.


Barbican backend models
------------------------

.. list-table::
   :header-rows: 1
   :widths: 22 22 56

   * - Backend
     - Payload storage
     - Notes
   * - ``simple-crypto``
     - Barbican MySQL database (encrypted)
     - Default Sunbeam configuration. Payload encrypted with a master
       KEK stored in the Barbican pod configuration file.
   * - Vault
     - HashiCorp Vault
     - Optional. Vault stores the secret payload. Barbican stores metadata
       only. Requires manual Vault deployment and Juju relation.
   * - PKCS#11 HSM
     - Hardware HSM
     - Not supported in Sunbeam default configuration.


Notes
------

**No centralised secrets manager**
   Sunbeam does not use a centralised secrets manager (such as Vault) by
   default. Service-level credentials are distributed via Juju relation data
   and stored as Juju Secrets. Protection of these credentials depends on Juju
   secret grants and node-level access controls.

**Juju Secret grant boundaries**
   Juju Secrets are granted per relation and are not exposed as Kubernetes
   Secret objects by default. A compromised service pod can still expose its
   own active credentials from rendered runtime configuration, so least-
   privilege service identity design remains critical.

**Application credential expiry**
   Application credentials created without an explicit ``--expiration`` value
   never expire. Password changes on the creating user account do not
   invalidate the application credential.

**Fernet key rotation scope**
   Fernet key rotation invalidates all tokens signed by the rotated key set.
   This includes service tokens. Rotation intervals shorter than the service
   token lifetime can cause authentication failures on in-flight requests.

For related reference material, see:

* :doc:`Encryption and data protection <encryption-and-data-protection>`
* :doc:`Identity and authorization <identity-and-authorization>`
* :doc:`Service-to-service trust <service-to-service-trust>`
* :doc:`Database and messaging security <database-and-messaging-security>`
