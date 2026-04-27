Encryption and data protection
==============================

Encryption coverage and key management for a Canonical OpenStack (Sunbeam)
deployment. Covers encryption in transit, encryption at rest, and key storage
locations.

**Default** = Sunbeam out-of-box state.
**Required configuration** = no default protection; operator must configure.


Encryption in transit
----------------------

**API and external traffic**

.. list-table::
   :header-rows: 1
   :widths: 28 15 15 42

   * - Path
     - Default
     - Required configuration
     - Notes
   * - Client → Traefik ingress (HTTP)
     - Plaintext
     - TLS CA or Vault
     - All OpenStack API endpoints reachable at the MetalLB IP. TLS requires
       operator configuration.
   * - Client → Traefik ingress (HTTPS)
     - Not active
     - TLS CA or Vault
     - Active ingress path once TLS is configured; port 80 redirects.
   * - Traefik → OpenStack service pods
     - Plaintext
     - Not available
     - Cleartext over the pod network regardless of ingress TLS state. No
       supported path to encrypt this leg independently.

**Internal service-to-service traffic**

.. list-table::
   :header-rows: 1
   :widths: 28 15 15 42

   * - Path
     - Default
     - Required configuration
     - Notes
   * - Nova → Neutron / Cinder / Glance
     - Plaintext
     - Not available
     - Internal API calls use bearer token authentication over plaintext HTTP.
       No configuration path to encrypt inter-service traffic in Sunbeam.
   * - Nova Compute → RabbitMQ (5672)
     - Plaintext
     - Not available
     - AMQP over plaintext. Not configurable in Sunbeam charm defaults.
   * - OpenStack services → MySQL (3306)
     - Plaintext
     - Not available
     - Unencrypted. Database access scoped by per-service credentials and
       ClusterIP network boundary.
   * - Neutron OVN driver → OVN NB DB (6641)
     - TLS (mTLS)
     - —
     - MicroOVN enables mutual TLS on all OVSDB connections by default.
   * - ovn-controller → OVN SB DB (6642)
     - TLS (mTLS)
     - —
     - MicroOVN enables mutual TLS on all OVSDB connections by default.
   * - Nova Compute → Ceph (via librbd)
     - Plaintext
     - msgr2 secure mode
     - Ceph msgr2 supports authenticated+encrypted mode (``secure``). Not
       enabled by default.

**Control plane traffic**

.. list-table::
   :header-rows: 1
   :widths: 28 15 15 42

   * - Path
     - Default
     - Required configuration
     - Notes
   * - Juju agents → Juju controller (17070)
     - TLS
     - —
     - Always encrypted. Certificate issued at bootstrap.
   * - sunbeam CLI / charm agents → MicroK8s API (16443)
     - TLS
     - —
     - Always encrypted. Certificate managed by MicroK8s.
   * - Kubernetes API server → etcd (2379)
     - TLS (mTLS)
     - —
     - Always encrypted. Managed by MicroK8s.
   * - etcd peer replication (2380)
     - TLS (mTLS)
     - —
     - Always encrypted. Managed by MicroK8s.
   * - sunbeam nodes → sunbeamd / clusterd (7000)
     - TLS
     - —
     - Always encrypted. Certificate issued at cluster bootstrap.

**Data plane traffic**

.. list-table::
   :header-rows: 1
   :widths: 28 15 15 42

   * - Path
     - Default
     - Required configuration
     - Notes
   * - Compute node → compute node (Geneve tunnel)
     - Plaintext
     - IPsec
     - OVN east-west tenant traffic is unencrypted by default. OVN IPsec
       encrypts and authenticates Geneve tunnels. Requires per-hypervisor
       key management.
   * - Compute node → compute node (VXLAN)
     - Plaintext
     - Not available
     - No encryption option for VXLAN in OVN.
   * - Nova Compute → Ceph OSD (6800–7300)
     - Plaintext
     - msgr2 secure mode
     - Block device I/O over Ceph librbd. Same msgr2 controls as MON above.


Encryption at rest
-------------------

**Block storage (Cinder)**

.. list-table::
   :header-rows: 1
   :widths: 28 20 52

   * - Volume type
     - Encryption
     - Notes
   * - Standard Ceph-backed volume
     - None (default)
     - Ceph stores data in plaintext on OSDs. No volume-level encryption
       unless explicitly configured.
   * - Encrypted Cinder volume
     - Operator-configured
     - Cinder supports volume encryption via dm-crypt / LUKS. Requires
       Barbican for key storage. Encryption is per-volume and must be
       specified at volume creation time via an encrypted volume type.
   * - Volume during live migration
     - Plaintext
     - Volume data moves over the storage network unencrypted during migration
       unless Ceph messenger encryption is enabled.

**Ephemeral instance storage**

.. list-table::
   :header-rows: 1
   :widths: 28 20 52

   * - Storage type
     - Encryption
     - Notes
   * - Ephemeral disk (local)
     - None (default)
     - Instance ephemeral disks on local compute node storage are not
       encrypted by default.
   * - Ephemeral disk (Ceph-backed)
     - None (default)
     - When ``nova`` is configured to use Ceph as the ephemeral backend,
       data is stored in a Ceph pool in plaintext.
   * - Instance memory
     - None
     - No memory encryption is applied at the hypervisor layer by Sunbeam.
       Hardware-based memory encryption (AMD SEV, Intel TDX) is not
       configured by default.

**Object and image storage (Glance / Swift / Ceph RadosGW)**

.. list-table::
   :header-rows: 1
   :widths: 28 20 52

   * - Backend
     - Encryption
     - Notes
   * - Glance images on Ceph
     - None (default)
     - Glance stores images in a Ceph pool without object-level encryption.
   * - Glance images on local filesystem
     - None
     - Not recommended for production. No encryption applied.

**Database storage (MySQL)**

.. list-table::
   :header-rows: 1
   :widths: 28 20 52

   * - Data
     - Encryption
     - Notes
   * - MySQL tablespace (on host disk)
     - None (default)
     - MySQL data is written to the Kubernetes persistent volume in plaintext.
       Encrypted PVs are operator-configured at the storage class level.
   * - Keystone Fernet keys (in Keystone DB)
     - Plaintext in DB
     - Fernet key material is stored in the Keystone database without
       additional encryption. Database access controls are the relevant
       boundary.
   * - Barbican secrets
     - Encrypted at rest
     - Barbican encrypts secret material before storing it. The encryption
       backend (simple-crypto or Vault) must be configured at deploy time.
       With simple-crypto, the master KEK is stored in Barbican's own
       configuration.


Key management summary
-----------------------

.. list-table::
   :header-rows: 1
   :widths: 28 20 20 32

   * - Key / secret type
     - Storage location
     - Management
     - Notes
   * - OpenStack service credentials
     - Juju relation data → Juju Secrets → pod environment
     - Juju / charm
     - Credentials are plaintext in pod environment. Any pod with namespace
       access to a service container can read its local process environment.
   * - Keystone Fernet signing keys
     - Keystone database; distributed via Juju peer relation
     - Keystone charm
     - Four active keys; daily rotation schedule. Rotation failure causes
       intermittent 401 errors if keys diverge across pods.
   * - Traefik TLS private key (TLS CA mode)
     - Operator-managed; provided as charm config or stored in a secrets system
     - Operator
     - Not managed by Sunbeam. Rotation is manual.
   * - Traefik TLS private key (Vault mode)
     - Vault PKI secrets engine
     - Vault charm (automated)
     - Vault holds the intermediate CA and issues certificates on TTL.
   * - Cinder volume encryption KEK
     - Barbican
     - Barbican / Vault
     - Required for encrypted Cinder volumes. Key is referenced by volume
       metadata; Barbican handles retrieval at attach time.
   * - OVN TLS keys and certificates
     - Host filesystem (``/var/snap/microovn``)
     - MicroOVN
     - Automatically rotated by MicroOVN. Not operator-accessible directly.
   * - Juju controller TLS key
     - Juju internal state
     - Juju
     - Not operator-accessible. Rotated automatically by Juju.
   * - MicroK8s / etcd TLS keys
     - ``/var/snap/microk8s/current/certs``
     - MicroK8s
     - ``microk8s refresh-certs`` rotates. All cluster members must be updated.


Notes and operator responsibilities
-------------------------------------

**Encryption layers are independent**
   TLS on the Traefik ingress does not protect data written to Ceph, MySQL,
   or ephemeral disk. Both transit and at-rest layers require independent
   configuration.

**Cinder volume encryption requires Barbican**
   Volume encryption cannot be used without a deployed Barbican service.
   Volumes created before Barbican is available cannot be retroactively
   encrypted.

**Barbican KEK storage (simple-crypto)**
   With the simple-crypto plugin, the master KEK is stored in the Barbican
   pod configuration. A pod compromise exposes the KEK. Vault-backed Barbican
   removes this risk.

**Juju Secret handling boundaries**
  Credentials are stored as Juju Secrets and granted through Juju relation
   permissions. They are not exposed as namespace-scoped secret resources by
   default. Once rendered into workload runtime configuration, a compromised
   pod can still extract its own active credentials.
   OVN IPsec requires per-hypervisor key management and adds CPU overhead
   without hardware offload. Physical data network isolation is the
   alternative for most deployments.

For related reference material, see:

* :doc:`Ports and Protocols </reference/security/ports-and-protocols>`
* :doc:`Certificates and TLS </reference/security/certificates-and-tls>`
* :doc:`Secrets and Key Management </explanation/security/secrets-and-key-management>`
* :doc:`Storage Protection </explanation/security/storage-protection>`
* :doc:`Network Exposure and Ingress Model </explanation/security/network-exposure-and-ingress-model>`
