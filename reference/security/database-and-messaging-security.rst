Database and messaging security
===============================

Access control, isolation, authentication, and encryption characteristics
for MySQL and RabbitMQ in a Canonical OpenStack (Sunbeam) deployment.

For encryption coverage on these paths, see
:doc:`Encryption and data protection <encryption-and-data-protection>`.


MySQL
------

**Deployment model**

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - Property
     - Value
   * - Operator
     - ``mysql-innodb-cluster-k8s`` charm (Galera cluster)
   * - Deployment topology
     - Single shared cluster by default. Per-service instances are supported
       but not the Sunbeam default.
   * - Kubernetes exposure
     - ClusterIP. Not reachable outside the ``openstack`` namespace.
   * - NetworkPolicy
     - None applied by default. Any pod in the namespace can attempt
       connections.
   * - Transport encryption
     - Not used. Connections from service pods are plaintext.
   * - Host-level encryption
     - None by default. Encrypted Kubernetes PersistentVolume requires
       operator configuration at the storage class level.


**Per-service database isolation**

Each OpenStack service connects to MySQL using a dedicated database and
user account. No service shares a database with another.

.. list-table::
   :header-rows: 1
   :widths: 22 22 56

   * - Service
     - Database name
     - Notes
   * - Keystone
     - ``keystone``
     - Stores projects, users, roles, Fernet key repository, and
       application credentials.
   * - Nova API
     - ``nova_api``
     - Global Nova state: flavors, aggregates, build requests.
   * - Nova cell
     - ``nova``
     - Per-cell instance state. Sunbeam uses a single cell (cell1).
   * - Neutron
     - ``neutron``
     - Network, subnet, port, and OVN binding state.
   * - Cinder
     - ``cinder``
     - Volume, snapshot, and backup state. Volume encryption metadata
       includes Barbican secret UUIDs.
   * - Glance
     - ``glance``
     - Image metadata. Image binary data is stored in Ceph, not MySQL.
   * - Heat
     - ``heat``
     - Stack state, template revisions, and trust tokens.
   * - Barbican
     - ``barbican``
     - Secret metadata and (with simple-crypto plugin) encrypted secret
       material. With Vault backend, only metadata is stored here.
   * - Octavia
     - ``octavia``
     - Load balancer, listener, pool, and amphora state.
   * - Designate
     - ``designate``
     - DNS zone and record state.
   * - Placement
     - ``placement``
     - Resource provider inventory and allocation state.
   * - Magnum
     - ``magnum``
     - Cluster template and cluster state.
   * - Manila
     - ``manila``
     - Share and share network state.
   * - Aodh
     - ``aodh``
     - Alarm definitions and state.
   * - Gnocchi
     - ``gnocchi``
     - Metric metadata. Time-series data stored in Ceph or file backend.
   * - Watcher
     - ``watcher``
     - Audit and action plan state.
   * - CloudKitty
     - ``cloudkitty``
     - Rating and billing state.
   * - Ironic
     - ``ironic``
     - Baremetal node and conductor state. Baremetal deployments only.


**MySQL user privileges**

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - Property
     - Value
   * - User scope
     - Each service holds a dedicated MySQL user. Users are granted
       ``ALL PRIVILEGES`` on their own database only.
   * - Cross-database access
     - Not granted. Services cannot access other services' databases directly.
   * - Root / administrative user
     - Managed by the charm. Not distributed to service pods.
   * - Credential storage
     - Juju Secret; consumed by the charm and rendered as pod environment
       variables.
   * - Credential rotation
     - Manual. Requires charm action or reconfiguration.


RabbitMQ
---------

**Deployment model**

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - Property
     - Value
   * - Operator
     - ``rabbitmq-k8s`` charm
   * - Kubernetes exposure
     - ClusterIP on port 5672 (AMQP). Management API (15672) is ClusterIP only.
   * - NetworkPolicy
     - None applied by default.
   * - Transport encryption
     - Not used. AMQP connections are plaintext.
   * - Authentication
     - AMQP username and password per vhost. Credentials provisioned via Juju
       relation at deployment time.


**Per-service vhost isolation**

Each OpenStack service is assigned a dedicated RabbitMQ vhost and user.
No service shares messaging infrastructure with another.

.. list-table::
   :header-rows: 1
   :widths: 22 22 56

   * - Service
     - Vhost
     - Notes
   * - Nova
     - ``nova``
     - Conductor, scheduler, and compute worker communication.
   * - Neutron
     - ``neutron``
     - Agent and server RPC. OVN driver uses this vhost.
   * - Cinder
     - ``cinder``
     - Volume, scheduler, and backup worker communication.
   * - Heat
     - ``heat``
     - Engine and API worker communication.
   * - Octavia
     - ``octavia``
     - Health manager and worker communication.
   * - Designate
     - ``designate``
     - Zone manager and producer worker communication.
   * - Aodh
     - ``aodh``
     - Alarm evaluation and notification worker communication.
   * - Watcher
     - ``watcher``
     - Applier and decision engine communication.
   * - Ironic
     - ``ironic``
     - Conductor and API communication. Baremetal deployments only.


**RabbitMQ user privileges**

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - Property
     - Value
   * - User scope
     - Each service user has ``configure``, ``write``, and ``read``
       permissions on its own vhost only.
   * - Cross-vhost access
     - Not granted. Services cannot publish or consume messages on other
       services' vhosts.
   * - Admin user
     - Managed by the charm. Used only for provisioning. Not distributed to
       service pods.
   * - Credential storage
     - Juju Secret; consumed by the charm and rendered as pod environment
       variables.
   * - Credential rotation
     - Manual. Requires charm action or reconfiguration.


Operational access
-------------------

.. list-table::
   :header-rows: 1
   :widths: 28 22 50

   * - Interface
     - Access model
     - Notes
   * - MySQL client (``mysql`` CLI)
     - Requires operator retrieval of root credential from charm secret
     - Not distributed by default. Must be retrieved via Juju secret
       inspection.
   * - RabbitMQ management UI (15672)
     - ClusterIP. Not accessible without ``kubectl port-forward``
     - Admin credentials required. Not distributed by default.
   * - ``rabbitmqctl``
     - Available inside the RabbitMQ pod via ``kubectl exec``
     - Requires ``kubectl exec`` access to the RabbitMQ pod.


Notes
------

**NetworkPolicy is required for both services**
   RabbitMQ and MySQL have no TLS and no NetworkPolicy by default. Any pod
   in the ``openstack`` namespace can attempt connections using valid
   credentials. Applying deny-all ingress rules with per-service allow rules
   is required to enforce the isolation that the vhost and database model
   provides at the application layer.

**No TLS configuration path**
   Neither RabbitMQ (5672) nor MySQL (3306) has a supported TLS
   configuration path in the default Sunbeam charm set. These paths will
   remain plaintext within the cluster pod network in standard deployments.

**Credential theft impact**
   A service credential leaked from Juju Secret material or rendered workload
   configuration gives an attacker AMQP or SQL access to that service's vhost
   or database only, not all services. Cross-service escalation requires
   credentials for each individual service.

For related reference material, see:

* :doc:`Encryption and data protection <encryption-and-data-protection>`
* :doc:`Secrets and credential handling <secrets-and-credential-handling>`
* :doc:`Service-to-service trust <service-to-service-trust>`
* :doc:`Ports and protocols <ports-and-protocols>`
