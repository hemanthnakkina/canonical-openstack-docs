Identity and authorization
==========================

Identity and authorization mechanisms for a Canonical OpenStack (Sunbeam)
deployment. Covers identity domains, token types, role model, policy
enforcement, and account types.

For trust relationships between services, see
:doc:`Service-to-service trust <service-to-service-trust>`.


Identity domains
-----------------

.. list-table::
   :header-rows: 1
   :widths: 22 25 53

   * - Domain
     - Contents
     - Notes
   * - ``Default``
     - All operator accounts, project-scoped users, service accounts
     - The only domain in a default Sunbeam deployment. Custom identity
       domains are not configured by Sunbeam.
   * - Juju identity
     - Juju controller users
     - Separate from Keystone. Managed by Juju. No Keystone integration.
   * - MAAS identity
     - MAAS operator accounts
     - MAAS deployments only. Separate from Keystone.


Token model
------------

.. list-table::
   :header-rows: 1
   :widths: 22 20 18 40

   * - Token type
     - Format
     - Default lifetime
     - Notes
   * - User token (scoped)
     - Fernet
     - 1 hour
     - Issued by Keystone on successful authentication. Carries project or
       system scope and role assignments. Validated locally by each service
       using the shared Fernet key set.
   * - Service token
     - Fernet
     - 1 hour
     - Issued to service accounts. Presented alongside user tokens on
       cross-service calls. Required by ``service_token_roles_required``.
   * - Application credential token
     - Fernet
     - No default expiry
     - Long-lived credentials created by users for automated clients.
       Inherit the role set of the parent user at creation time. Do not
       expire automatically unless an expiry is set at creation.
   * - Unscoped token
     - Fernet
     - 1 hour
     - Issued before project selection. Limited use; not accepted by most
       service APIs.
   * - Console token
     - Random UUID
     - ~10 minutes (default)
     - Issued by Nova on a ``GET /os-console-auth-tokens`` or console API
       call (``os-getVNCConsole``, ``os-getSPICEConsole``). Not a Fernet
       token. Opaque UUID embedded in the console URL; validated by the
       noVNC or SPICE proxy on first WebSocket connection. Single-use.


Role model
-----------

Sunbeam deploys the Wallaby RBAC role model. All services ship with
policies that enforce scope-based role separation.

.. list-table::
   :header-rows: 1
   :widths: 22 20 58

   * - Role
     - Scope
     - Effective permissions
   * - ``admin`` (system scope)
     - System
     - Cloud-wide administrative access. Can operate across all projects and
       services. Assigned to the ``admin`` user in the ``Default`` domain.
   * - ``admin`` (project scope)
     - Project
     - Administrative access within the scoped project only. Cannot perform
       system-scoped operations without a separate grant.
   * - ``member``
     - Project
     - Standard project user. Can create and manage resources within the
       project. Default role for new users.
   * - ``reader``
     - Project or system
     - Read-only access. Cannot create or modify resources.
   * - ``service``
     - System
     - Reserved for internal service accounts. Required for service token
       validation when ``service_token_roles_required = True``.
   * - ``load-balancer_admin``, ``load-balancer_member``, etc.
     - Project
     - Octavia-specific sub-roles. Follow the same scope model.


Policy enforcement
-------------------

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - Property
     - Value
   * - Policy engine
     - oslo.policy, per-service
   * - Policy files
     - Embedded in service containers. Sunbeam does not support operator
       policy overrides.
   * - Policy scope enforcement
     - Enabled by default; ``enforce_scope = True`` in all Sunbeam-shipped services
   * - Cross-service consistency
     - Not enforced. Each service holds an independent policy. No mechanism
       validates consistency across services.
   * - ``service_token_roles_required``
     - ``True`` on all services receiving cross-service calls (Nova, Neutron,
       Cinder, Glance). Rejects cross-service requests that lack a valid
       service token alongside the user token.


Service accounts
-----------------

Each OpenStack service is provisioned with a dedicated Keystone service account.

.. list-table::
   :header-rows: 1
   :widths: 22 20 18 40

   * - Service
     - Account
     - Role
     - Notes
   * - Nova
     - ``nova``
     - ``service`` (system scope)
     - Used for cross-service API calls to Neutron, Cinder, Glance.
   * - Neutron
     - ``neutron``
     - ``service`` (system scope)
     - Used for OVN state and metadata API calls.
   * - Cinder
     - ``cinder``
     - ``service`` (system scope)
     - Used for volume operations across projects.
   * - Glance
     - ``glance``
     - ``service`` (system scope)
     - Used for image store access.
   * - Heat
     - ``heat``
     - ``service`` (system scope)
     - Used for trust delegation and stack ownership.
   * - Octavia
     - ``octavia``
     - ``service`` + ``load-balancer_admin`` (system scope)
     - Requires elevated role for amphora management.
   * - Barbican
     - ``barbican``
     - ``service`` (system scope)
     - Used to retrieve and store secrets on behalf of tenants.
   * - Designate
     - ``designate``
     - ``service`` (system scope)
     - Used for DNS record management.
   * - Placement
     - ``placement``
     - ``service`` (system scope)
     - Service endpoint registered in Keystone. Nova uses its own service
       token when calling Placement; Placement validates the token via
       Keystone middleware.
   * - Ironic
     - ``ironic``
     - ``service`` (system scope)
     - Baremetal deployments only.
   * - Horizon
     - None
     - N/A
     - Horizon does not hold a Keystone service account. It authenticates
       users directly against Keystone and presents the user's token to
       all subsequent OpenStack API calls. No service token is issued or
       used.


Authorization enforcement points
----------------------------------

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Component
     - Enforcement behavior
   * - Keystone
     - Issues and validates Fernet tokens. Does not re-validate tokens
       presented to other services; validation is local to each service using
       the shared Fernet key set.
   * - Nova
     - Validates user token scope and role on all API requests. Validates
       service token on cross-service requests when
       ``service_token_roles_required = True``.
   * - Neutron
     - Same as Nova. Also enforces project-scoped network isolation.
   * - Cinder
     - Same as Nova. Volume attachment enforces project scoping unless an
       admin-scoped override is applied.
   * - Glance
     - Same as Nova. Image visibility (public/private/shared) is enforced by
       Glance policy independently of Nova.
   * - Barbican
     - Enforces ACL on secrets per secret UUID. Only the owning project or
       explicitly granted users can retrieve a secret.
   * - Horizon
     - No independent authorization enforcement. Presents the authenticated
       user's Keystone token to downstream OpenStack API calls. Access
       decisions are made entirely by each OpenStack service policy.
       Horizon does not inspect roles or scopes.
   * - Traefik ingress
     - No authentication. Routes requests to services without inspecting tokens.
       All authentication enforcement occurs inside service pods.


Component coverage
-------------------

Summary of authentication model and external exposure for all major
identity-aware components.

.. list-table::
   :header-rows: 1
   :widths: 22 18 22 38

   * - Component
     - Exposure
     - Authentication
     - Notes
   * - Keystone
     - External (Traefik)
     - Password / application credential / federated IdP
     - Identity provider for the deployment. Issues Fernet tokens.
   * - Horizon
     - External (Traefik)
     - Keystone (username + password or federated)
     - Web session established on authentication. Subsequent API calls use
       the user's Keystone token.
   * - Nova API
     - External (Traefik)
     - Keystone token
     - Service token also required on cross-service calls.
   * - Neutron
     - External (Traefik)
     - Keystone token
     - Service token also required on cross-service calls.
   * - Cinder
     - External (Traefik)
     - Keystone token
     - Service token also required on cross-service calls.
   * - Glance
     - External (Traefik)
     - Keystone token
     - Service token also required on cross-service calls.
   * - Placement
     - External (Traefik)
     - Keystone token (service token from Nova)
     - Not typically called directly by end users.
   * - Barbican
     - External (Traefik)
     - Keystone token + ACL
     - ACL enforced per secret UUID.
   * - Heat
     - External (Traefik)
     - Keystone token (trust delegation for stack ops)
     - Trust tokens scoped to stack owner's project.
   * - Nova noVNC / SPICE proxy
     - External (Traefik or direct)
     - Console token (UUID, single-use)
     - Console token issued by Nova; not a Keystone Fernet token.
   * - RabbitMQ
     - Internal (ClusterIP)
     - AMQP username + password (per vhost)
     - No Keystone integration. See database-and-messaging-security.
   * - MySQL
     - Internal (ClusterIP)
     - Database username + password (per service)
     - No Keystone integration. See database-and-messaging-security.
   * - MicroK8s API server
     - Internal (node address, port 16443)
     - Client certificate or service account token
     - No Keystone integration. RBAC enforced by Kubernetes.
   * - Juju controller
     - Internal (port 17070)
     - TLS + unit agent secret
     - No Keystone integration.
   * - OVN databases
     - Internal (ClusterIP)
     - Mutual TLS (component certificates)
     - No Keystone integration. MicroOVN manages certificates.
   * - Nova metadata API
     - Internal (instances via 169.254.169.254)
     - Metadata proxy shared secret (not Keystone token)
     - Neutron metadata agent presents the shared secret to Nova.


Notes
------

**Token lifetime and revocation**
   Fernet tokens are self-contained and cannot be individually revoked.
   The only revocation mechanism is Fernet key rotation: rotating all
   active keys invalidates all outstanding tokens signed by rotated keys.
   Token lifetime directly determines the window during which a stolen token
   remains valid.

**Application credentials**
   Application credentials do not expire by default. They remain valid after
   the creating user's password is changed. They must be explicitly deleted
   to revoke access.

For related reference material, see:

* :doc:`Service-to-service trust <service-to-service-trust>`
* :doc:`Secrets and credential handling <secrets-and-credential-handling>`
* :doc:`Observability and audit coverage <observability-and-audit-coverage>`
* :doc:`Certificates and TLS <certificates-and-tls>`
