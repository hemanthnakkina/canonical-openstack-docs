Service-to-service trust
========================

Trust provisioning, authentication mechanisms, and credential delivery for
service-to-service communication in a Canonical OpenStack (Sunbeam)
deployment.

For token types and role model, see
:doc:`Identity and authorization <identity-and-authorization>`.
For credential storage, see
:doc:`Secrets and credential handling <secrets-and-credential-handling>`.


Trust provisioning mechanism
-----------------------------

All OpenStack service credentials are provisioned through Juju relation data
at deployment time. No service discovers or negotiates credentials at runtime.

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - Property
     - Value
   * - Provisioning system
     - Juju / charm relation data
   * - Provisioning time
     - Deployment and relation creation
   * - Credential format
     - Keystone username, password, project, and domain; passed as relation
       data between charms
   * - Storage after provisioning
     - Juju Secret; consumed by charms and rendered into pod configuration
       or environment variables
   * - Rotation trigger
     - Manual (operator-initiated Juju action or charm configuration change)
   * - Automatic rotation
     - Not supported in default Sunbeam configuration


Service token enforcement
--------------------------

.. list-table::
   :header-rows: 1
   :widths: 28 22 50

   * - Service
     - ``service_token_roles_required``
     - Effect
   * - Nova API
     - ``True``
     - Rejects cross-service requests that do not present a valid ``service``
       role token alongside the user token.
   * - Neutron
     - ``True``
     - Same as Nova.
   * - Cinder
     - ``True``
     - Same as Nova.
   * - Glance
     - ``True``
     - Same as Nova.
   * - Placement
     - ``True``
     - Same as Nova. Nova is the primary caller.
   * - Keystone
     - N/A
     - Issues tokens; does not receive cross-service calls requiring service
       token validation.
   * - Barbican
     - ``True``
     - Service token required for automated secret retrieval (e.g., Cinder
       volume key fetch).


Service-to-service call paths
-------------------------------

.. list-table::
   :header-rows: 1
   :widths: 22 22 20 36

   * - Caller
     - Target
     - Authentication
     - Notes
   * - Nova
     - Neutron
     - User token + Nova service token
     - Port binding, security group operations on behalf of instances.
   * - Nova
     - Cinder
     - User token + Nova service token
     - Volume attachment and detachment.
   * - Nova
     - Glance
     - User token + Nova service token
     - Image fetch at instance boot.
   * - Nova
     - Placement
     - Nova service token only
     - Resource allocation; no user context required.
   * - Nova
     - Barbican
     - Nova service token
     - Ephemeral disk key retrieval for encrypted volumes.
   * - Cinder
     - Barbican
     - Cinder service token + user token
     - Encryption key storage and retrieval.
   * - Octavia
     - Neutron
     - Octavia service token
     - Amphora port and network management.
   * - Heat
     - Any service
     - Delegated trust token
     - Heat uses Keystone trust tokens scoped to the stack owner's project.
       Trust tokens inherit the owner's roles at trust creation time.
   * - Designate
     - Neutron (optional)
     - Designate service token
     - Floating IP PTR record management, if integration is enabled.
   * - Ironic
     - Neutron
     - Ironic service token
     - Baremetal deployments only. Port provisioning.
   * - Ironic
     - Glance
     - Ironic service token
     - Image deployment to bare metal nodes.
   * - Horizon
     - All OpenStack APIs
     - User token only
     - Horizon presents the authenticated user's Keystone token to each API
       call. No service token is issued or used. The receiving service
       applies its own policy against the user token.
   * - Nova
     - Console proxy (noVNC / SPICE)
     - Console token (UUID)
     - Nova issues a console token on a console URL request. The console
       proxy (noVNC or SPICE) validates the token against the Nova API
       on first WebSocket connection. The token is single-use.
   * - Neutron metadata agent
     - Nova metadata API (8775)
     - Metadata proxy shared secret
     - Instances reach the metadata endpoint at 169.254.169.254; OVN
       routes the traffic to the Neutron metadata agent, which proxies
       the request to the Nova metadata API using a shared secret header
       (``X-Metadata-Provider-Signature``). No Keystone token is used
       on this path.


Metadata proxy trust
---------------------

The Nova metadata API uses a shared secret for authentication from the
Neutron metadata agent, not a Keystone token.

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - Property
     - Value
   * - Shared secret configuration key
     - ``metadata_proxy_shared_secret`` in Nova and Neutron configuration
   * - Transport
     - HTTP. The shared secret is sent as the
       ``X-Metadata-Provider-Signature`` request header.
   * - Scope
     - Used only on the internal path from the Neutron metadata agent pod
       to the Nova metadata API pod. Not externally reachable.
   * - Storage
     - Juju Secret provisioned via Juju relation between the Nova and Neutron
       charms.
   * - Rotation
     - Manual. Requires charm reconfiguration and pod restart on both
       Nova and Neutron sides.
   * - Instance identity forwarding
     - Neutron metadata agent appends the instance UUID and project ID as
       headers after validating the shared secret. Nova uses these headers
       to retrieve instance-specific metadata.


OVN trust model
----------------

OVN components do not use Keystone for authentication. Trust is established
via TLS certificates managed by MicroOVN.

.. list-table::
   :header-rows: 1
   :widths: 28 22 50

   * - Connection
     - Authentication
     - Notes
   * - Neutron OVN driver → OVN Northbound DB (6641)
     - Mutual TLS (client certificate)
     - Certificate issued per-component by MicroOVN at bootstrap.
   * - ovn-controller → OVN Southbound DB (6642)
     - Mutual TLS (client certificate)
     - One certificate per compute node.
   * - ovn-northd → OVN NB DB / SB DB
     - Mutual TLS (client certificate)
     - Internal to ovn-central pod.
   * - ovn-central peers (raft 6643, 6644)
     - Mutual TLS (peer certificate)
     - Raft cluster membership verified by certificate.

No Keystone token is presented on any OVN connection. OVN access control
is entirely certificate-based.


Juju trust model
-----------------

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - Property
     - Value
   * - Agent authentication
     - Each Juju unit agent holds a unique secret (``JUJU_API_ADDRESSES``,
       ``JUJU_PASSWORD``) provisioned at unit deployment.
   * - Controller authentication
     - TLS on port 17070. Certificate issued at bootstrap.
   * - Credential distribution
     - Juju passes credentials to charms via relation data and Juju Secrets;
       charms render credentials into workload runtime configuration.
   * - Cross-model trust
     - Not used in default Sunbeam configuration.
   * - Revocation
     - Unit credential revoked on ``juju remove-unit``; controller credential
       revoked on ``juju revoke``.


MicroK8s and Kubernetes component coverage
-------------------------------------------

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - Component
     - Authorization model
   * - sunbeam-operator
     - Kubernetes ServiceAccount with ClusterRole granting full access to the
       ``openstack`` namespace. Used by charm controllers.
   * - OpenStack service pods
     - No Kubernetes ServiceAccount by default. Pods use the ``default``
       service account in the ``openstack`` namespace.
   * - etcd access
     - Restricted to kube-apiserver. etcd does not accept direct client
       connections outside of MicroK8s internal mTLS.
   * - Juju agent pods
     - Juju model-operator ServiceAccount; scoped to the Juju model namespace.

Kubernetes control plane security characteristics:

.. list-table::
   :header-rows: 1
   :widths: 22 18 22 38

   * - Component
     - Exposure
     - Authentication
     - Notes
   * - MicroK8s API server (16443)
     - Node address (internal)
     - Client certificate or service account bearer token
     - TLS required on all connections. RBAC enforced. No external access
       in default Sunbeam configuration.
   * - etcd (2379 / 2380)
     - Internal to Kubernetes control plane
     - Mutual TLS (peer certificates)
     - Not directly accessible outside the MicroK8s internal network.
       etcd data is not encrypted at rest by default.
   * - kube-scheduler / kube-controller-manager
     - Not externally reachable
     - Kubernetes API server RBAC (service account tokens)
     - Communicate with the API server over the in-cluster network.
   * - MicroK8s DNS (CoreDNS)
     - ClusterIP
     - None
     - Internal name resolution only. Not exposed outside the cluster.
   * - kubectl access
     - kubeconfig on sunbeam nodes (``/var/snap/microk8s/``)
     - Client certificate (admin kubeconfig)
     - Admin kubeconfig grants cluster-admin privileges. Must be restricted
       to operator accounts.


Notes
------

**No service mesh or mTLS between OpenStack pods**
   OpenStack services call each other over plaintext HTTP within the pod
   network. The only authentication on these calls is the Keystone token
   and service token pair. No mutual TLS is configured between service pods.

**Heat trust delegation**
   Heat trust tokens are created at stack creation time and inherit the
   stack owner's roles. If the owner's roles are widened after trust
   creation, the trust does not automatically inherit the change.

**OVN certificate rotation**
   MicroOVN rotates OVN component certificates automatically. No operator
   action is required. Rotation does not interrupt OVSDB connections.

For related reference material, see:

* :doc:`Identity and authorization <identity-and-authorization>`
* :doc:`Certificates and TLS <certificates-and-tls>`
* :doc:`Secrets and credential handling <secrets-and-credential-handling>`
* :doc:`Database and messaging security <database-and-messaging-security>`
