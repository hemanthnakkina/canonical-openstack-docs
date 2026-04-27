Identity and Access Model
===============================================

Every authorization decision in Canonical OpenStack flows through Keystone.
It is the source of tokens that services accept as proof of identity, the
registry where service accounts are provisioned, and the authority against
which roles are evaluated. Its centrality creates a characteristic risk: a
misconfiguration or compromise at any point in this model has consequences
that extend across every service simultaneously.


The identity model
-------------------

The domain structure determines how service accounts are isolated and what
authority they carry — the foundation for understanding why credential
compromise has deployment-wide consequences.

Keystone organises identities into three distinct domains.

The admin_domain
~~~~~~~~~~~~~~~~~

Created at bootstrap, this domain holds operator user accounts — by default the
``admin`` user produced by ``sunbeam credentials``. Users authenticate with
password credentials and receive tokens scoped to the projects and roles they
have been assigned. The admin credential bundle is system-wide root-equivalent
and requires careful rotation after bootstrap, protection from version control
and shell history, and deliberate delegation through application credentials
rather than direct exposure to automated tooling.

The service_domain
~~~~~~~~~~~~~~~~~~~

Service accounts for each OpenStack component live in this domain: a ``nova``
user for Nova, a ``neutron`` user for Neutron, a ``cinder`` user for Cinder,
and so on. These accounts allow services to authenticate with one another and
to validate the tokens that users present. All service accounts belong to the
``service`` project and carry the ``admin`` role within it — a privilege
necessary for cross-tenant operations like Nova listing all instances across
projects or Neutron managing ports on behalf of any user.

The consequence is that every service account credential authorizes admin-level
operations against the service's own API and, through cross-service calls,
against other services.

The Default domain
~~~~~~~~~~~~~~~~~~~

Holds operator-created tenant users and projects unless they are explicitly
placed elsewhere. This isolation prevents multi-tenant workloads from
interfering with administrative accounts.

Three facts define the security posture of this model:

* Service identities are intentionally privileged
* Service credentials are distributed automatically through charm relations
* Those credentials are accepted across multiple services, not just one

Service accounts are provisioned automatically by the Keystone charm via the
Juju identity service relation. When nova-k8s registers with Keystone over the
``identity-service`` relation, the Keystone charm creates the corresponding
user in ``service_domain``, assigns it to the ``service`` project, generates a
random password, and returns the credentials — along with endpoint URLs, domain
IDs, project IDs, and the admin role name — through the relation. The nova-k8s
charm renders those values into Nova's configuration inside the Kubernetes pod.
Nova's service identity then lives in two places: the Keystone database and the
Juju Secret referenced from relation data. Both are authoritative.

The admin credential bundle from ``sunbeam credentials`` carries system-wide
admin authorization and must be treated as root-equivalent: rotated after
bootstrap, never stored in version control or shell history, and not issued to
automated tooling directly. For automation that needs to call OpenStack APIs,
application credentials are the appropriate mechanism — they delegate a
constrained subset of roles to a non-human caller without exposing the
underlying password, are scoped to a specific project, and expire on demand.


Service-to-service trust and where it can fail
------------------------------------------------

Token validation ties every service call back to Keystone, but the stateless
design that makes this efficient also shapes what revocation can and cannot do.

Keystone issues Fernet tokens — compact, signed strings that do not require a
database lookup to validate. Each token encodes the issuing time, expiry, user
ID, project and domain scope, and role assignments, and is signed using a
rotating set of Fernet keys. Sunbeam configures four active keys with a daily
rotation schedule.

Token validation is stateless. When a service receives an API request, its
``keystonemiddleware`` validates the token locally against the current Fernet
keys without consulting Keystone. Keystone cannot instantly invalidate a token
already in circulation.

Token revocation
~~~~~~~~~~~~~~~~

Keystone provides a revocation endpoint and creates a revocation event when
called, but each service polls for the revocation list on a schedule (typically
every 10 seconds), so a revoked token continues to be accepted until services
catch up. More critically, if a user is deleted, their project disabled, or their
role removed, tokens already issued remain valid at every service until they
expire — 1 hour by default.

Password rotation in Keystone does not close existing token-based access to
Nova, Cinder, or Neutron until those tokens expire.

Cross-service calls widen this gap. When a user creates an instance, the
cross-service workflow looks approximately like this:

1. Nova calls Glance to fetch a virtual machine image
2. Nova calls Neutron to provision a network port
3. Neutron in turn updates OVN's port database
4. Cinder attaches a volume if one is requested

Each downstream service receives and independently validates the user's token.
A single token with project-admin scope authorizes operations across the full
depth of this call chain simultaneously.

Service tokens
~~~~~~~~~~~~~~

To prevent a user who has obtained a valid token from replaying it directly
against an internal service endpoint, Sunbeam configures
``service_token_roles_required = True`` on all services that receive
cross-service calls. A request to Neutron that claims to originate from Nova
must present Nova's service token alongside the user's token. Neutron rejects
the call if no service token is present. This prevents a user-token-only
attacker from directly impersonating a service — but it does not protect
against a compromised service using its own token legitimately. A service whose
credentials have been stolen can make any call that service is authorized to
make, with a valid service token and no anomaly to detect.

.. warning::
   A compromised service credential looks like legitimate service traffic.

Fernet key rotation introduces a specific failure mode that is difficult to
diagnose in production. The Keystone charm distributes new keys to all
Keystone pods through Juju's peer relation. If a pod fails to receive a key
update — due to a restart during rotation, a Kubernetes scheduling event, or a
transient peer relation failure — that pod will start rejecting tokens signed
with the new key while still accepting tokens signed with the now-expired old
key. The symptom is intermittent 401 errors with no consistent pattern, because
Traefik distributes requests across Keystone pods. The failure is
indistinguishable from a misconfigured token expiry or a clock drift issue
without directly inspecting the current key inventory across all pods.


Where the access model breaks down
------------------------------------

The access model works as intended when tokens are short-lived, roles map
accurately to what each identity needs, and service policies remain consistent.
Each of these has a characteristic failure mode — and because Keystone is the
shared root of trust, failures compound across services rather than staying
contained.

The default roles follow the secure RBAC model introduced in the OpenStack
Wallaby release: ``admin``, ``member``, and ``reader``, with system-scope
``admin`` carrying cloud-wide authority distinct from project-scope ``admin``.
Canonical's Rocks ship with policies that enforce this distinction. Sunbeam does
not support operator policy overrides through custom policy files in order to
prevent policy drift away from secure defaults.

Service account compromise
~~~~~~~~~~~~~~~~~~~~~~~~~~

Service account compromise has the widest blast radius. When an attacker gains
code execution inside a service pod — through a vulnerability in the service
Rock, a supply-chain compromise, or a misconfigured pod security context — the
service account credentials are present in the pod's environment as plaintext
values rendered by the charm. The attacker authenticates directly with
Keystone using those credentials and obtains a fresh service token with the
full admin role in ``service_domain``. From that point, they do not need to
stay inside the compromised pod.

Propagation from one compromised service follows a predictable path:

* The attacker uses the service credential to request a fresh token
* The token is accepted by downstream services as a valid admin-level service identity
* Cross-service APIs execute normally, so the attack blends into expected control-plane traffic

Using Nova's service token, the attacker can call Neutron to list tenant
networks and ports and modify security group rules across projects. The same
identity can call Cinder to detach volumes from running instances and re-attach
them elsewhere, then call Glance to replace or delete images.

Because each service validates the token and receives a legitimate admin-role
grant, no service-level check fails. The compromise propagates from one pod to
the full API surface of the cloud, authenticated at each step, with no
network-level boundary to cross because all services share the same
``openstack`` Kubernetes namespace and the same Traefik ingress.

Juju secrets
~~~~~~~~~~~~

Juju Secrets are not exposed through namespace-scoped secret resources.
However, credentials are still rendered into workload configuration and process
environments so services can authenticate at runtime. A privileged compromise
of a service pod can therefore extract that service's active credentials and
pivot through authorized cross-service calls.


Operational tradeoffs
-----------------------

Each of the following controls involves a genuine tradeoff between security
posture and operational overhead. Token lifetimes that limit compromise windows
also break automation that does not handle token refresh. Service token
enforcement closes an impersonation surface but rejects callers that have not
been updated. Application credentials reduce the blast radius of a compromised
caller but require explicit lifecycle management that does not happen
automatically. These are not defaults to adjust experimentally — each one
changes the shape of a failure mode that would otherwise be difficult to detect.

Token expiry
~~~~~~~~~~~~~

Shortening ``[token] expiration`` reduces the maximum window a compromised
credential remains usable. A 15-minute expiry limits an intercepted token to a
15-minute window, regardless of when Keystone propagates the revocation event.
The operational cost is concrete: automation that runs long jobs without
handling token refresh will start receiving 401 errors mid-execution, and those
errors typically surface deep in job logs as generic failures rather than
authentication events. The trade is not hypothetical — operators who shorten
token expiry without auditing their automation almost always discover failing
pipelines within the first rotation cycle. The right expiry is the shortest
value the deployment's automation can reliably tolerate, not the default.

Service token enforcement
~~~~~~~~~~~~~~~~~~~~~~~~~

``service_token_roles_required = True`` closes the service-impersonation
surface with a real enforcement boundary. Any tool that calls internal service
endpoints directly with only a user token, will receive 403 responses.
The correct response is to update the integration, not disable the setting
since turning it off removes the only mechanism that prevents user-token-only
access to endpoints designed for service callers. If an integration cannot be
updated, the service it calls should be isolated behind a dedicated Traefik
route with a stricter allowlist, not given a blanket exemption from service
token validation.

Application credentials for automation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Application credentials limit the blast radius of automated caller compromise
to a single project and role subset, rather than the full admin credential.
Their management cost is real: 

1. they do not rotate automatically
2. must be explicitly invalidated when an operator's access changes
3. an expiry that was set and forgotten will break automation at an
   inconvenient time

The alternative — automation that uses the admin password or a long-lived admin
token — is not simpler to manage. It is simpler to configure once and then
ignore until something goes wrong. When it does, the exposure is total. The
effort of tracking application credential expiry is proportionate to the
risk reduction it provides.


Summary
---------

The model holds when service accounts are isolated, token lifetimes are
calibrated to what the deployment's automation can tolerate, and credentials
are handled with the discipline their privilege level demands.

The failures that matter in practice are not protocol failures. A compromised
service account produces valid tokens that pass every downstream check. A
long token expiry means that a password rotation does not close access. Neither
of these failures produce an error — they produce correct-looking behavior from
a compromised identity. The controls that prevent them require deliberate
operator decisions at deployment time and consistent maintenance thereafter.

For related configuration and implementation details, see:

* :doc:`Network Exposure and Ingress Model </explanation/security/network-exposure-and-ingress-model>`
* :doc:`Secrets and Key Management </explanation/security/secrets-and-key-management>`
* :doc:`Compute Isolation </explanation/security/compute-isolation>`
* :doc:`Control Plane Integrity </explanation/security/control-plane-integrity>`
* :doc:`Storage Protection </explanation/security/storage-protection>`
* :doc:`Observability and Audit </explanation/security/observability-and-audit>`
* :doc:`Service endpoint encryption </explanation/service-endpoint-encryption>`
* :doc:`Design considerations </explanation/design-considerations>`
