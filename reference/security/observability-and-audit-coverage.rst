Observability and audit coverage
================================

Event coverage, log destinations, formats, retention defaults, and gaps
for observability and audit in a Canonical OpenStack (Sunbeam) deployment.


OpenStack API audit events
---------------------------

Keystone emits CADF-formatted audit events for authentication and token
operations. Other services emit oslo.messaging notifications.

.. list-table::
   :header-rows: 1
   :widths: 25 18 22 15 20

   * - Event category
     - Source service
     - Format
     - Destination
     - Default enabled
   * - Authentication (success/failure)
     - Keystone
     - CADF
     - Keystone log / oslo.messaging
     - Yes
   * - Token issuance and revocation
     - Keystone
     - CADF
     - Keystone log / oslo.messaging
     - Yes
   * - Project and user operations
     - Keystone
     - CADF
     - Keystone log / oslo.messaging
     - Yes
   * - Policy changes
     - Keystone
     - CADF
     - Keystone log
     - Yes (logged, not notified)
   * - Instance create / delete / start / stop
     - Nova
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes (notifications)
   * - Instance state transitions
     - Nova
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes
   * - Volume create / delete / attach / detach
     - Cinder
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes
   * - Network create / delete / update
     - Neutron
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes
   * - Security group create / update / delete
     - Neutron
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes
   * - Floating IP associate / disassociate
     - Neutron
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes
   * - Image create / update / delete / upload
     - Glance
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes
   * - Image download (fetch)
     - Glance
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes
   * - Secret create / retrieve / delete
     - Barbican
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes
   * - Load balancer create / delete
     - Octavia
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes
   * - Stack create / update / delete
     - Heat
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes
   * - DNS zone / record create / delete
     - Designate
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes
   * - Alarm create / state transition
     - Aodh
     - oslo.messaging notification
     - oslo.messaging exchange; log
     - Yes


Log destinations and retention
--------------------------------

.. list-table::
   :header-rows: 1
   :widths: 22 25 53

   * - Log source
     - Destination
     - Notes
   * - OpenStack service logs
     - Kubernetes pod stdout/stderr → MicroK8s journald
     - Retained until journald log rotation. Default journald retention
       is based on disk space (10% of disk). No configurable day-based
       retention in default Sunbeam configuration.
   * - oslo.messaging notifications
     - RabbitMQ exchange
     - Transient. Notifications are consumed by subscribers (e.g., Aodh,
       Gnocchi, Ceilometer). Unconsumed notifications are dropped at queue
       expiry. No persistent notification store by default.
   * - Juju agent logs
     - Juju controller database → ``juju debug-log``
     - Retained in the Juju controller. Log level configurable per model.
       No automatic purge unless controller disk pressure is reached.
   * - MicroK8s Kubernetes audit logs
     - Not enabled
     - Kubernetes API server audit logging is not enabled in default
       MicroK8s configuration. See gaps section below.
   * - MicroOVN logs
     - journald on each node
     - OVN daemon logs. Standard journald rotation applies.
   * - MicroCluster / sunbeamd logs
     - journald on each node
     - Control plane daemon logs. Standard journald rotation applies.
   * - Horizon access logs
     - Kubernetes pod stdout/stderr → MicroK8s journald
     - Apache2 access logs (mod_wsgi). Includes URL path, HTTP status, and
       source IP (Traefik ingress IP, not the end-user IP). Not CADF format;
       not correlated with OpenStack request IDs.


CADF event fields
------------------

Keystone CADF events include the following fields relevant to security review:

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Field
     - Value
   * - ``typeURI``
     - ``http://schemas.dmtf.org/cloud/audit/1.0/event``
   * - ``action``
     - Mapped CADF action (``authenticate``, ``read``, ``create``, ``delete``, etc.)
   * - ``outcome``
     - ``success`` or ``failure``
   * - ``initiator.id``
     - Keystone user UUID
   * - ``initiator.host.address``
     - Source IP address of the request
   * - ``target.id``
     - UUID of the resource being acted on
   * - ``target.typeURI``
     - Resource type URI
   * - ``observer.id``
     - Keystone service UUID
   * - ``eventTime``
     - ISO 8601 timestamp with microsecond precision


Notification consumers
-----------------------

oslo.messaging notifications are consumed by the following components when
deployed:

.. list-table::
   :header-rows: 1
   :widths: 22 22 56

   * - Consumer
     - Source events
     - Notes
   * - Ceilometer / Gnocchi
     - Nova, Cinder, Neutron, Glance
     - Telemetry metering. Events stored in Gnocchi time-series backend
       (Ceph or file). Not deployed by default in Sunbeam.
   * - Aodh
     - Ceilometer, Neutron
     - Alarm evaluation. Requires Ceilometer pipeline to be deployed.
   * - Watcher
     - Nova (compute node metrics)
     - Optimization actions. Not deployed by default.

Without at least one persistent notification consumer deployed, oslo.messaging
notifications are transient and are lost after the queue expiry window.


Audit coverage gaps
--------------------

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Area
     - Gap description
   * - Kubernetes API server audit
     - MicroK8s does not enable Kubernetes audit logging by default. API
       server requests (including kubectl exec, Secret reads, and pod
       lifecycle events) are not logged.
   * - RabbitMQ access audit
     - RabbitMQ does not produce per-message or per-connection audit logs.
       AMQP connections are not logged. Authentication failures may appear
       in RabbitMQ node logs at ``warning`` level only.
   * - MySQL query audit
     - Per-query MySQL audit logging is not configured. Database access
       events (including unauthorized connection attempts) are not logged.
   * - OVN Northbound/Southbound DB access
     - OVSDB connections to OVN are not audit-logged. Connection events
       are not emitted by MicroOVN.
   * - Juju relation data access
     - Juju does not produce an audit log of which charm accessed which
       relation data key. Charm actions are logged, but relation reads
       are not.
   * - Notification persistence
     - oslo.messaging notifications are transient without a deployed consumer.
       There is no default mechanism to persist or forward notifications to
       a SIEM or log aggregation system.
   * - Horizon user actions
     - Horizon does not emit audit events. User actions via the dashboard
       are only logged at the receiving OpenStack service (e.g., Nova,
       Neutron). The initiator IP in service logs reflects the Traefik
       ingress IP rather than the user's source address.
   * - Console session access
     - noVNC and SPICE console sessions are not audit-logged. A console
       connection event does not appear in Nova or Neutron audit logs.
       The console token issuance is logged by Nova at ``INFO`` level only.
   * - Nova metadata API access
     - Instance access to the metadata endpoint (169.254.169.254) is not
       audited. No log entry is generated for metadata fetch events at any
       layer of the proxy chain.
   * - Node-level access
     - SSH sessions and shell commands on the underlying Ubuntu host are
       not audited by Sunbeam. auditd must be configured separately at the
       OS level.


Notes
------

**No centralised log aggregation**
   Sunbeam does not deploy a log aggregation stack (ELK, Loki, Splunk, etc.)
   by default. Logs are distributed across pod stdout, journald instances,
   and the Juju controller. Centralised collection requires operator
   configuration.

**Correlating events across services**
   CADF events carry an ``initiator.id`` (user UUID) and ``target.id``.
   oslo.messaging notifications carry ``payload.user_id`` and
   ``payload.project_id``. Correlation across services requires either a
   shared SIEM or log aggregation with a common request ID field.
   OpenStack does not propagate a single trace ID across all service log
   lines by default.

**Audit log integrity**
   Pod stdout logs are written to journald, which does not provide tamper
   evidence or log signing. An attacker with root access on a node can
   modify journald logs. Forwarding logs to an external, append-only
   destination before they can be modified is required to maintain
   audit integrity.

For related reference material, see:

* :doc:`Identity and authorization <identity-and-authorization>`
* :doc:`Ports and protocols <ports-and-protocols>`
* :doc:`Secrets and credential handling <secrets-and-credential-handling>`
