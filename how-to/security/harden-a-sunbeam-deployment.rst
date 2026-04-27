Harden a Sunbeam deployment
============================

This guide provides actionable steps to reduce risk in a Canonical OpenStack
(Sunbeam) deployment. Steps are ordered from the outermost surface inward:
external exposure, identity, internal communication, storage, secrets, and
observability.

For background on the risk model driving these recommendations, see
:doc:`Security </explanation/security/index>`.

For a reference of ports, certificates, and encryption defaults, see
:doc:`Security reference </reference/security/index>`.


Before you begin
-----------------

* You have a functioning Sunbeam deployment.
* You have operator-level access (``sunbeam``, ``kubectl``, ``juju``).
* You have reviewed the
  :doc:`Security reference: Ports and protocols </reference/security/ports-and-protocols>`
  to understand the current exposure surface.
* You are prepared to test API connectivity after enabling TLS.


Reduce external exposure
-------------------------

**1. Enable TLS on the external API ingress.**

   All OpenStack API endpoints are reachable over plaintext HTTP by default.
   Enable TLS termination at the Traefik ingress.

   Using an external CA certificate:

   .. code:: text

      sunbeam enable tls --ca-cert <path-to-ca-cert> --ca-key <path-to-ca-key>

   Using Vault as a PKI backend:

   .. code:: text

      sunbeam enable tls --vault

   Verify by connecting to any API endpoint over HTTPS and confirming the
   certificate presented matches your CA:

   .. code:: text

      openstack --os-cacert <ca-cert> catalog list

   For full configuration details, see
   :doc:`Service endpoint encryption </explanation/service-endpoint-encryption>`.

**2. Confirm MetalLB IP allocation is restricted to your management network.**

   The MetalLB IP pool determines which addresses Traefik binds to. If the
   pool covers a publicly routable range, all API endpoints are reachable from
   outside your network.

   Review the current pool:

   .. code:: text

      kubectl get ipaddresspool -n metallb-system -o yaml

   If the pool is too broad, update the deployment manifest to restrict the
   range to your management VLAN before re-applying.

**3. Review and reduce which API services are enabled.**

   Disabled services have no API surface. Services you do not use (for example,
   Ironic in a virtual-only deployment, or Designate if not providing DNS)
   should remain disabled.

   List enabled features:

   .. code:: text

      sunbeam enabled-features

   Disable unused services:

   .. code:: text

      sunbeam disable <feature-name>

**4. Restrict operator SSH access.**

   Sunbeam nodes accept SSH on port 22. Restrict access to known operator
   IP ranges at the network level. Disable password authentication:

   .. code:: text

      sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
      sudo systemctl reload sshd

   Verify:

   .. code:: text

      ssh -o PasswordAuthentication=yes <node-ip>
      # Expected: Permission denied (publickey).


Strengthen identity and access
--------------------------------

**5. Remove or restrict the ``admin`` project from shared use.**

   By default the ``admin`` user has access to the ``admin`` project with
   full policy. Do not share this credential with application teams.

   Create per-project users and assign the minimum required roles:

   .. code:: text

      openstack user create --project <project> --password-prompt <username>
      openstack role add --user <username> --project <project> member

**6. Enforce ``service_token_roles_required`` in Nova and Neutron.**

   Without this, any valid user token can impersonate an internal service
   request. Confirm the setting in the charm configuration:

   .. code:: text

      juju config nova-k8s service_token_roles_required=true
      juju config neutron-k8s service_token_roles_required=true

   Verify that Nova API rejects requests using a plain user token on
   service-scoped endpoints.

**7. Audit Keystone application credentials.**

   Application credentials do not expire by default and inherit the project
   roles of the parent user. Unused credentials remain valid after the
   operator who created them is removed.

   List all application credentials across all projects:

   .. code:: text

      openstack application credential list --all-projects

   Delete credentials that are no longer associated with an active workload.

**8. Rotate Keystone Fernet keys on a defined schedule.**

   Fernet keys are rotated by the Keystone charm on a default schedule.
   Confirm rotation is active and check the key repository:

   .. code:: text

      juju run keystone-k8s/0 get-fernet-keys

   If the rotation schedule has not been customized, set a rotation interval
   appropriate for your token lifetime:

   .. code:: text

      juju config keystone-k8s token-expiration=3600

   A compromised token is only valid until the next rotation completes.


Harden internal communication
--------------------------------

**9. Apply Kubernetes NetworkPolicy to restrict pod-to-pod traffic.**

   By default, all pods within the ``openstack`` namespace can reach each
   other. NetworkPolicy deny-all ingress with per-service allow rules is the
   recommended baseline.

   Apply a default-deny policy to the openstack namespace:

   .. code:: yaml

      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      metadata:
        name: default-deny-ingress
        namespace: openstack
      spec:
        podSelector: {}
        policyTypes:
        - Ingress

   Then add per-service allow rules for each legitimate communication path.
   Consult
   :doc:`Security reference: Ports and protocols </reference/security/ports-and-protocols>`
   for the full list of required paths.

   .. note::

      Verify that all OpenStack API operations continue to function after
      applying deny rules. Begin in a non-production environment.

**10. Enable OVN IPsec for east-west tenant traffic.**

    Geneve tunnels between hypervisors carry tenant VM traffic in plaintext by
    default. Enable OVN IPsec to encrypt and authenticate these tunnels:

    .. code:: text

       sudo microovn.ovn-nbctl set nb_global . ipsec=true

    Verify that IPsec security associations are established on each compute
    node:

    .. code:: text

       sudo ip xfrm state list

    Expected output includes SAs for each peer compute node IP.

    .. note::

       IPsec adds CPU overhead on hosts without hardware offload support.
       Measure throughput before enabling on production workloads.

**11. Restrict RabbitMQ access to known consumer pods.**

    RabbitMQ (port 5672) is ClusterIP-scoped but accessible to all pods in the
    namespace. Use NetworkPolicy to allow only known OpenStack service pods to
    connect to the RabbitMQ service.

    Identify the pods that legitimately connect to RabbitMQ:

    .. code:: text

       kubectl get pods -n openstack -l app.kubernetes.io/name=rabbitmq -o wide

    Create ingress allow rules scoped to those pod selectors.


Protect storage and data
--------------------------

**12. Create and enforce encrypted Cinder volume types.**

    Cinder volume encryption requires Barbican. Once Barbican is deployed,
    create an encrypted volume type:

    .. code:: text

       openstack volume type create --encryption-provider nova.volume.encryptors.luks.LuksEncryptor \
         --encryption-cipher aes-xts-plain64 \
         --encryption-key-size 256 \
         --encryption-control-location front-end \
         LUKS

    Require project teams to create volumes using this type for workloads that
    handle sensitive data.

    Verify encryption is applied:

    .. code:: text

       openstack volume show <volume-id> | grep encryption

**13. Audit Ceph pool ACLs and client key capabilities.**

    Ceph clients (Nova, Cinder, Glance) authenticate with per-service keyring
    files. Confirm each client key has the minimum required capabilities:

    .. code:: text

       sudo ceph auth list

    Nova compute nodes should have access only to the ``vms`` pool. Cinder
    should have access only to the ``volumes`` pool. Any key with ``caps osd =
    "profile rbd"`` without pool restrictions should be updated:

    .. code:: text

       sudo ceph auth caps client.nova mon 'profile rbd' osd 'profile rbd pool=vms'

**14. Enable Ceph msgr2 secure mode (optional).**

    Ceph msgr2 protocol supports encrypted transport. Enable secure mode
    on all Ceph daemons and clients if the storage network is not physically
    isolated:

    .. code:: text

       sudo ceph config set global ms_cluster_mode secure
       sudo ceph config set global ms_service_mode secure
       sudo ceph config set global ms_client_mode secure

    Verify with:

    .. code:: text

       sudo ceph status
       # Confirm cluster health returns HEALTH_OK after mode change.


Secure secrets and credentials
--------------------------------

**15. Deploy Vault as the Barbican secrets backend.**

    The default Barbican simple-crypto plugin stores its master key encryption
    key in the pod configuration. A pod compromise exposes the KEK and all
    secrets it protects.

    Deploy Vault and configure Barbican to use Vault as its secrets backend:

    .. code:: text

       sunbeam enable vault

    After enabling, rotate all existing Barbican secrets so they are
    re-encrypted under the Vault-backed KEK.

**16. Audit Juju Secrets and secret grants in the openstack model.**

    OpenStack service credentials are stored as Juju Secrets and granted to
    applications through Juju relations.

    List all Secrets in the model:

    .. code:: text

       juju list-secrets -m openstack

    Inspect a secret to confirm grant scope and revision history:

    .. code:: text

       juju show-secret -m openstack <secret-id>

    Confirm secrets are granted only to the applications that require them,
    and remove stale grants or stale secret revisions.

**17. Remove or disable unused Juju credentials and model access.**

    Juju model access allows credential injection via charm relations. Operators
    who no longer manage the deployment should have their Juju access revoked:

    .. code:: text

       juju revoke <username> admin sunbeam/openstack


Validate observability and audit
----------------------------------

**18. Confirm Keystone audit events are enabled.**

    Keystone emits CADF-format audit notifications to RabbitMQ when the audit
    middleware is active. Verify the setting:

    .. code:: text

       juju config keystone-k8s audit-middleware=true

    Generate a test event:

    .. code:: text

       openstack token issue

    Confirm the event appears in the Keystone service log:

    .. code:: text

       kubectl logs -n openstack -l app.kubernetes.io/name=keystone | grep "audit"

**19. Confirm Gnocchi or an external log sink is receiving audit events.**

    If audit events are only in pod logs, they are lost when pods restart.
    Confirm Loki (or another external log sink) is receiving Keystone audit
    output:

    .. code:: text

       sunbeam dashboard

    Open Grafana, query the ``{job="keystone"}`` log stream, and filter for
    ``audit``. Confirm recent authentication events are present.

**20. Set log retention policy.**

    Pod log retention defaults to the Kubernetes node disk policy. For
    compliance use cases, ship logs to a durable external sink and set a
    retention window. Confirm the Loki retention period matches your
    requirements:

    .. code:: text

       juju config loki-k8s retention-period=<days>d


Verify your posture
---------------------

After completing the steps above, perform the following verification checks.

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Check
     - Command / Expected outcome
   * - TLS on API endpoints
     - ``curl -I https://<api-ip>:443`` returns HTTP 200 or redirect; no
       certificate errors.
   * - Plaintext API rejected
     - ``curl -I http://<api-ip>:80`` returns 301 redirect to HTTPS (if
       redirect is configured) or connection refused.
   * - Fernet key rotation active
     - ``juju run keystone-k8s/0 get-fernet-keys`` returns multiple key
       entries with recent timestamps.
   * - Encrypted volumes available
     - ``openstack volume type list`` includes the LUKS volume type.
   * - OVN IPsec SAs present (if enabled)
     - ``sudo ip xfrm state list`` on each compute node shows active SAs.
   * - Audit events flowing
     - Grafana Loki query for ``{job="keystone"} |= "audit"`` returns
       recent entries.
   * - Unused application credentials removed
     - ``openstack application credential list --all-projects`` returns only
       active credentials.


For reference material supporting this guide, see:

* :doc:`Security reference: Ports and protocols </reference/security/ports-and-protocols>`
* :doc:`Security reference: Certificates and TLS </reference/security/certificates-and-tls>`
* :doc:`Security reference: Encryption and data protection </reference/security/encryption-and-data-protection>`
* :doc:`Identity and access model </explanation/security/identity-and-access-model>`
* :doc:`Secrets and key management </explanation/security/secrets-and-key-management>`
* :doc:`Observability and audit </explanation/security/observability-and-audit>`
