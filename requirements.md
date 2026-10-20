# Shared SOCA plugin requirements

This document sets the **best-case, highest-bar target** for every Simple OAuth
CIMD Access (SOCA) plugin: a complete, secure, reliable and easy-to-operate product.
The requirements are independent of current plugin capabilities and implementation
cost. Platform-specific integration must preserve the same security and product
outcomes.

This profile covers browser-based OIDC sign-in to a host application using CIMD,
including admission, local accounts and host sessions. Each implementation must
identify the login surfaces it protects, such as administration or end-user access.
API authorisation, other OAuth grants and delegated access to unrelated services
require separate, explicit profiles; they must not be inferred from SSO support.

The imperative statements below are requirements for that target. Each plugin must
track them as **supported**, **partially supported** or **planned**, with a version,
acceptance evidence and open gaps. A feature can be disabled by an operator while
still being a required capability. Use **not applicable** only with an evidenced
platform/protocol reason, never because an implementation is an MVP or lacks a
library. An operational workaround does not count as support for the capability
it replaces. Plan delivery incrementally without lowering these acceptance criteria.

## Functional requirements

### Provider connection and sign-in

- **F-01: Provider configuration.** Provide sensible RESOAuth defaults and support
  multiple explicitly trusted OIDC/CIMD provider profiles with isolated issuer,
  tenant/site, algorithm and admission policies. Validate discovery, exact issuer,
  endpoints and capabilities before activation. Reject explicitly unsupported CIMD;
  report missing discovery support as unconfirmed until a real login establishes
  interoperability. Never discover a trusted issuer from untrusted identity claims.
- **F-02: Secure login flow.** Use OIDC authorisation code flow with PKCE S256,
  independent state and nonce, exact callback URLs and asymmetric
  `private_key_jwt` client authentication. Protect against request forgery, code
  replay and issuer mix-up, and bind each transaction to the initiating browser.
- **F-03: Identity validation.** Verify the ID-token signature and required issuer,
  subject, audience, nonce and time claims before admitting a user. Validate
  authorisation-response issuer and other conditional claims where applicable.
  Bind UserInfo to the same validated subject and reject conflicting identity
  claims. Support fresh authentication and enforce configured assurance requirements
  using validated `auth_time`, `acr` and `amr` with explicit provider semantics;
  email verification alone cannot establish MFA. OAuth access alone is not
  authentication.

### Client metadata: hosted and proxied CIMD

- **F-04: Self-hosted documents.** Serve public client metadata and public JWKS
  over HTTPS without requiring login. Include the exact document URL as
  `client_id`, registered callback URLs and accurate client authentication and
  algorithm metadata. Use valid JSON, appropriate content types, explicit cache
  lifetimes and predictable cache invalidation. Validate public reachability and
  document contents before enabling login or promoting a new key.
- **F-05: Proxy mode.** Support a configurable HTTPS metadata proxy when the
  application cannot host publicly reachable metadata, including local HTTPS
  development. When proxy mode is selected, default to
  `https://dynamic-cimd.resoauth.cloud/cimd/%s`. Provide a privileged admin UI field
  and code/deployment configuration for replacing the proxy URL template. Use one
  `%s` placeholder in the HTTPS path for unpadded base64url-encoded UTF-8 JSON
  containing public client metadata; the proxy must return `client_id` equal to
  the exact resulting document URL. Validate custom templates and provide a
  generated-URL preview, connection/document check and reset to the default.

  Validate that the retrieved document matches the generated public metadata and
  exact client ID. Support both inline public keys and externally hosted JWKS where
  the provider profile permits them. Keep provider authentication independent of
  the metadata proxy, with the same security guarantees in both modes.
- **F-06: Public data boundary.** Publish only public client information and
  public keys, using `jwks_uri` or inline `jwks` as appropriate. Never publish
  private keys, client secrets, tokens, user claims or access-policy lists. Explain
  that proxy URLs and their encoded metadata are public and visible to the proxy.
- **F-07: Client identity changes.** Maintain stable client IDs wherever the
  hosting mode permits. Provide a controlled migration when a metadata or key
  change creates a new proxy URL/client ID. Explain the effect on consent,
  pairwise subjects and account association. Preserve in-flight logins through
  routine changes using bounded configuration/key snapshots; invalidate
  affected transactions immediately for security revocations. Validate provider
  restrictions on metadata/callback origins before applying a change.

### Keys and algorithm support

- **F-08: Persistent keys.** Generate or provision strong, persistent asymmetric
  signing keys with stable key identifiers. Protect private material using the
  host's protected secret storage or authenticated encryption, separated from
  application data. Support external key-management services and non-exportable
  signing keys. Coordinate keys across workers, retain them across restarts and
  upgrades, and provide tested backup, restore and lost-key recovery procedures.
- **F-09: Automated client-key rotation.** Rotate keys automatically with overlap
  and no planned SSO outage in both hosted and proxy modes. Confirm replacement
  publication, account for provider/CDN metadata and JWKS cache bounds, warm up
  replacement keys and retain retiring keys through outstanding transactions and
  assertions. Coordinate promotion atomically across workers, monitor overdue or
  failed rotation, and provide safe retry, operator-triggered rotation and emergency
  revocation. A manual maintenance window is a gap against this requirement.
- **F-10: ES256 default and algorithm agility.** Implement **ES256 with P-256 as
  the required default and recommended algorithm** for client-assertion signing and
  ID-token verification. Generate P-256 client keys by default and explicitly set
  `token_endpoint_auth_signing_alg` and `id_token_signed_response_alg` to `ES256`
  in client metadata. Sign assertions with client keys and verify ID tokens with
  the trusted issuer's keys. Configure these purposes independently; use only
  ES256 until an administrator explicitly approves a different algorithm for the
  relevant issuer and purpose. Fail clearly if the provider cannot support the
  configured profile; never select an alternate
  solely from provider ordering or an incoming token header.

  The highest-bar interoperability target also includes `ES384`, `ES512`, RSA-PSS
  (`PS256`, `PS384`, `PS512`), `RS256`, fully specified EdDSA (`Ed25519`, `Ed448`)
  and standardised post-quantum profiles (`ML-DSA-44`, `ML-DSA-65`, `ML-DSA-87`).
  Treat support for these algorithms separately from enabling them. Use maintained,
  reviewed libraries and published test vectors. Select only the intersection of
  explicit local policy and provider capabilities, pin key types and algorithms,
  support orderly migration, and reject unsigned tokens, symmetric substitutions,
  deprecated profiles and silent downgrade. Missing provider support is a reported
  compatibility constraint; missing plugin support remains an implementation gap.
- **F-11: Provider-key changes.** Support provider signing-key rollover with
  bounded JWKS retrieval/refresh and clear cache expiry rules. Unknown keys and
  invalid signatures must not trigger unlimited fetches or bypass validation.
  Check assertion claims and audience against the configured provider profile and
  isolate key caches by issuer and purpose. Retire expired trust and honour known
  key revocations without accepting stale keys to preserve availability.

### Admission, existing accounts and new accounts

- **F-12: Admission policy.** Support explicit email/domain allow and deny rules
  with documented matching semantics, deny precedence and deny-by-default settings.
  Provide a policy preview and explicit rules for sensitive accounts/sites. Where
  email controls admission or account selection, require a validated email with
  `email_verified` equal to boolean `true` from the trusted provider. Apply policy
  on every SSO login, including existing users.
- **F-13: Existing users.** Select exactly one eligible local account using the
  documented association policy. Respect disabled accounts, site membership and
  the host's access requirements. Reject ambiguous matches. Preserve local
  passwords, usernames, roles, groups and elevated privileges unless an explicit,
  separately authorised operation changes them; SSO must not reactivate users.
- **F-14: New users.** Make first-login provisioning opt-in. Create an account
  only after authentication and admission succeed, with an explicitly configured,
  least-privileged role or group. Validate required local fields, avoid usable
  default passwords, and handle collisions, concurrent logins and failed creation
  without duplicate accounts, partial privileges or a session for an invalid user.
- **F-15: Authorisation boundary.** Keep authentication, admission and local
  permissions distinct. Provider profile or role claims must not grant
  administrative privileges implicitly. Support explicit, allowlisted claim-to-group
  or role mappings with issuer binding, privilege ceilings and audited changes;
  administrative elevation requires independent local approval. Enforce removal and
  policy revocation for future logins and affected existing SSO sessions, with a
  documented maximum propagation time.

### Claim association and profile mapping

- **F-16: Protected account association.** Use a persisted, unique issuer/subject
  association for returning users. Linking an existing local account requires
  local reauthentication or an audited administrator-approved process. A verified
  email can propose a match but cannot silently take over an existing account,
  especially a privileged one. Provide explicit linking, unlinking and recovery,
  prevent concurrent conflicting links, and never use mutable profile fields as
  proof of ownership. Keep account links separate from admission and profile data.
- **F-17: Identity lifecycle.** Define behaviour when issuer, subject, client ID or
  email changes, and when an address is reused. Reject conflicting associations
  and provide an administrator-controlled, audited migration and recovery path.
  Support provider/client migrations without duplicate accounts or silent relinking.
  Do not infer that a new pairwise subject belongs to an existing user from a
  display name, username or newly matching email.
- **F-18: Profile claims.** Support documented mappings for common claims such
  as `given_name`, `family_name`, `name` and `preferred_username` where the host has
  suitable fields, plus administrator-configured allowlisted custom-field mappings.
  Provide preview, validation and explicit fill-empty, create-only and synchronisation
  policies for each field. Separate profile updates from account selection and
  admission. Define precedence between signed ID-token claims and subject-matched
  UserInfo, validate types/lengths/content, handle missing claims and collisions,
  preserve local edits by default and audit mapping changes. Do not allow profile
  mappings to write credentials, identity links or privilege fields.
- **F-19: Claim diagnostics.** Provide a safe way to understand available claims
  and mapping decisions through a privileged, explicitly enabled, short-lived,
  redacted claim inspector. Scope capture to an explicitly selected test transaction
  and explain the data displayed. Encrypt retained reports and expire/delete them
  automatically. Do not retain raw tokens or turn ordinary login logs into claim
  dumps.

### Administration, user experience and recovery

- **F-20: Configuration and validation.** Provide an accessible admin experience
  and code/deployment configuration, plus automation interfaces, for provider,
  metadata mode and proxy URL, admission, provisioning, mapping and key lifecycle
  settings. Document precedence, show the effective value and configuration source,
  and make deployment-locked settings read-only in the UI. Support
  deployment-managed secrets and policy locks. Validate and preview changes,
  reject stale concurrent edits, audit updates and support rollback. Require
  connection/document checks and a successful real
  test login before activating a new profile; report each result separately. Keep
  validation mode free of account creation, linking and permission changes, and
  preserve the operator's current session. Report the effective admission and
  mapping decisions without applying them.
- **F-21: Login and logout.** Integrate a clear SSO entry point into the host's
  login experience, create normal host sessions after successful admission, and
  restrict post-login destinations to safe local URLs. Support configurable session
  lifetime, reauthentication and revocation, local logout, RP-initiated provider
  logout and validated back-channel logout for providers advertising those features.
  Keep local logout reliable during provider outages. Offer optional automatic SSO
  redirects with loop prevention and a reliable local recovery path.
- **F-22: Local recovery.** Retain a usable local administrator login and a
  documented SSO bypass/disable procedure. Provider, proxy, key or configuration
  failures must produce actionable errors without locking administrators out or
  silently relaxing authentication checks. Protect recovery accounts with strong
  local authentication, least privilege and audited use, and make recovery testable
  before SSO rollout and after upgrades.
- **F-23: Host integration and plugin lifecycle.** Respect the host's account,
  permission, session, password-reset and extension APIs. Support canonical HTTPS
  origins, reverse proxies and mounted path prefixes without exposing protected
  routes. Define install, upgrade, disable, re-enable and uninstall behaviour.
  Disabling SSO must stop new SSO transactions and invalidate pending ones; define
  what happens to existing SSO sessions. Preserve local accounts, permissions and
  recovery access. Make permanent removal of plugin data explicit, scoped and
  auditable, including the implications for keys, links and retained backups.
- **F-24: Token lifecycle.** Request only the scopes needed for the configured
  login and profile features. Do not request offline access or retain refresh
  tokens for ordinary sign-in. Discard authorisation codes and access tokens after
  their immediate use; retain an ID token only when a configured protocol feature
  requires it, with encryption, access controls and a defined expiry. Any separate
  delegated-access profile must define secure token storage, audience/resource
  binding, refresh rotation, revocation and deletion explicitly.

### Session and transaction storage

- **F-25: Platform sessions and minimal browser storage.** Prefer the existing
  CMS/platform/framework session and security facilities for login transactions
  and the resulting authenticated session. Securely retain the per-transaction
  PKCE **verifier**, expected nonce and state association with confidentiality and
  tamper resistance. The S256 challenge is derived from the verifier and does not
  replace it. Bind these values to the initiating browser, issuer, client ID and
  callback; expire and consume each transaction once, support parallel login tabs,
  and rotate the authenticated session identifier using the platform's APIs.

  Favour stateless plugin operation using platform-provided session, cryptographic
  and replay facilities. Avoid a separate plugin session system, duplicate
  persistent transaction storage and additional cookies wherever the platform can
  provide the required guarantees. Browser-carried transaction state must be
  authenticated, encrypted if it contains secrets, short-lived and bound to the
  initiating browser. A signed or encrypted state envelope alone does not establish
  browser binding or prevent replay; retain minimal shared replay state when
  necessary to enforce one-time consumption across workers.

  If an additional cookie is necessary for secure browser binding, document why
  the platform facilities are insufficient. Use the smallest necessary,
  short-lived, narrowly scoped cookie with `Secure`, `HttpOnly` and an appropriate
  `SameSite` policy. Avoid JavaScript storage, including `localStorage`,
  `sessionStorage` and IndexedDB; any unavoidable use needs a documented necessity
  and security review. Never expose verifiers, tokens or other secret transaction
  material in JavaScript-readable storage or plaintext URLs. Cookie avoidance or
  statelessness must never weaken confidentiality, browser binding or replay checks.

## Non-functional requirements

- **N-01: Secure implementation.** Use maintained protocol/cryptographic
  libraries and host security APIs, backed by a maintained threat model. Protect
  admin actions against CSRF, escape displayed content, consume login transactions
  atomically once, and expire and clean up temporary state. Follow F-25 for platform
  sessions and strong browser binding; matching User-Agent strings alone is not
  sufficient. Use secure defaults and least privilege throughout, with no
  production switch that bypasses required
  protocol, certificate, identity or authorisation checks.
- **N-02: Network safety.** Validate TLS for outbound HTTPS, defend discovery,
  metadata and key fetches against SSRF and DNS rebinding, and enforce response-size
  limits, total request deadlines and redirect policies. Reject CIMD fetch redirects.
  Configure canonical external URLs and trusted reverse proxies rather than trusting
  arbitrary Host or forwarded headers. Test IPv4/IPv6, mixed DNS answers, rebinding
  and slow/large responses, and support deployment egress restrictions without
  weakening checks.
- **N-03: Privacy and secrets.** Request only necessary scopes/claims, minimise
  stored identity data, and document external services, data flows, retention and
  deletion. Keep secrets and tokens out of metadata, logs, errors, URLs and release
  artefacts; the protocol's short-lived callback code/state parameters must be
  redacted from access logs. Encrypt retained sensitive data, define bounded
  retention and automatic deletion, and document backup implications. Make any
  optional telemetry explicitly opt-in and keep authentication data out of it.
- **N-04: Reliability and concurrency.** Work across supported multi-worker,
  multi-site and database deployments with correctly scoped configuration, shared
  state where needed and safe concurrent account creation/key maintenance. Support
  rolling upgrades and routine maintenance without a planned SSO outage. Publish
  availability and recovery objectives for supported deployment profiles, test
  failover and restore, and fail closed during outages or expired trust. Recovery
  must not resurrect revoked credentials, links or transactions.
- **N-05: Performance and abuse resistance.** Keep normal application requests
  inexpensive, use bounded caching where appropriate, rate-limit expensive login
  and diagnostic operations, and prevent refresh storms. Document cleanup and
  scheduled-maintenance needs, resource bounds and deployment rate-limit controls.
  Publish measurable login latency, throughput, storage and cleanup budgets with
  representative load-test results; verify bounded behaviour under sustained abuse.
- **N-06: Observability and usability.** Meet WCAG 2.2 AA for plugin-owned
  login/admin screens, including keyboard and assistive-technology checks. Provide
  understandable errors and redacted diagnostic references. Distinguish protocol,
  metadata, admission and local-account failures for operators without exposing
  secrets or enabling public account enumeration. Audit sensitive configuration
  changes and account-association/provisioning operations using access-controlled,
  tamper-evident audit records with documented retention. Expose health, rotation
  state and security/availability metrics without disclosing identity data.
- **N-07: Compatibility and maintenance.** Publish supported host/runtime
  versions, dependency requirements, installation/upgrade/uninstall behaviour,
  licensing, release notes and security reporting instructions. Use reproducible
  packaging, signed release artefacts and build provenance, a software bill of
  materials, immutable dependency/workflow references and automated vulnerability
  checks. Publish a security response/patch policy, enforce release gates for
  unresolved critical/high findings, and test migrations and rollback.
- **N-08: Verification and interoperability.** Test the packaged plugin and
  supported platform combinations, including negative security cases and browser
  flows, concurrency, fault injection and recovery. Run applicable OIDC conformance
  suites, automated protocol/security checks and independent security review with
  tracked remediation. Record provider profiles, algorithm coverage and dated
  live-login results for hosted and proxy modes. Keep simulated tests, successful
  document fetches and real end-user login evidence distinct; state untested
  behaviour and known gaps explicitly.
- **N-09: Implementation profile and traceability.** Publish the assessed revision
  of these requirements, supported protocol/provider profiles, protected login
  surfaces and a requirement-by-requirement evidence matrix. Publish security
  defaults, units, configurable ranges and hard limits for transaction/assertion
  lifetimes, clock skew, key rotation, metadata/JWKS cache bounds, session revocation
  and diagnostic retention. State how overlap/retention is derived from those
  limits, how unconfirmed provider cache bounds are handled, and which settings are
  operator choices. Include executable fixtures and acceptance procedures that
  another developer can reproduce; a published support claim needs dated evidence.

## Required acceptance evidence

Use the requirement IDs above in each plugin's support matrix and release
checklist. A requirement is supported only when its acceptance evidence verifies
the full outcome; a manual workaround does not close the gap. Retain evidence for:

| Area | Representative checks |
| --- | --- |
| Metadata | Hosted and proxy documents, the default Dynamic CIMD template, custom proxy configuration through both UI and code, precedence/locks, template validation, exact client ID, public-only keys, unavailable/incorrect documents and proxy URL changes. |
| Authentication | Successful login and cancellation; expired/replayed/wrong-browser callbacks; issuer, audience, nonce and signature failures; unsupported algorithms. |
| Algorithm policy | ES256/P-256 as the fresh-install default for both purposes; explicit independent overrides; rejection of provider-order/header-driven fallback; published vectors for every supported algorithm. |
| Key lifecycle | Unattended overlapping rotation in both metadata modes, algorithm migration, restarts, multiple workers, provider rollover, cache overlap, emergency revocation and recovery after failure. |
| Accounts and claims | Protected linking of existing users, new users, provisioning disabled, deny precedence, unverified/missing email, duplicate/disabled accounts, concurrent creation/linking, reassigned email, provider/client migration and preserved local edits/permissions. |
| Sessions and assurance | Fresh authentication, assurance policy, permission revocation, session expiry, local/provider/back-channel logout, and provider outages. |
| Session storage | Reuse of platform facilities, protected verifier/nonce, copied-callback and replay rejection, parallel tabs, session fixation prevention, stateless-envelope tampering/expiry, and documented necessity for any extra cookie or JavaScript storage. |
| Operations | Tested local recovery, rolling upgrades, failover/restore, fault injection, latency/load budgets, accessibility checks, redacted diagnostics and verified release provenance. |
| Lifecycle and privacy | Validation without account/session side effects; disable/re-enable/uninstall; scoped data deletion; token disposal; backup implications and preserved local accounts. |
| Developer handoff | Requirements revision, evidence matrix, documented security defaults and limits, and reproducible fixtures for the declared deployment profile. |

## References

The targets above combine product requirements with protocol requirements. Follow
the applicable specifications and document implemented revisions and provider
profiles:

- [OAuth Client ID Metadata Document, draft 02](https://www.ietf.org/archive/id/draft-ietf-oauth-client-id-metadata-document-02.html):
  the CIMD revision reviewed for this initial requirements document; an
  Internet-Draft, not a final RFC.
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
  and [Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html).
- [OIDC client metadata definitions](https://openid.net/specs/openid-connect-registration-1_0.html#ClientMetadata):
  metadata vocabulary for the client profile, including explicit signing
  algorithm preferences; this does not require a dynamic registration endpoint.
- [OAuth security best current practice (RFC 9700)](https://www.rfc-editor.org/rfc/rfc9700.html)
  and [PKCE (RFC 7636)](https://www.rfc-editor.org/rfc/rfc7636.html).
- [JWT client authentication (RFC 7523)](https://www.rfc-editor.org/rfc/rfc7523.html),
  [JWT best current practices (RFC 8725)](https://www.rfc-editor.org/rfc/rfc8725.html)
  and [JWK (RFC 7517)](https://www.rfc-editor.org/rfc/rfc7517.html).
- [JSON Web Algorithms (RFC 7518)](https://www.rfc-editor.org/rfc/rfc7518.html),
  including the ES256/P-256 signing profile. ES256 as the recommended default is
  a SOCA product requirement, independent of algorithm preferences in other profiles.
- [IANA JOSE algorithm registry](https://www.iana.org/assignments/jose), including
  the registered fully specified EdDSA and ML-DSA signing profiles. Registry
  registration alone does not establish implementation safety or provider support.
- [RP-Initiated Logout 1.0](https://openid.net/specs/openid-connect-rpinitiated-1_0.html)
  and [Back-Channel Logout 1.0](https://openid.net/specs/openid-connect-backchannel-1_0.html).
- [Web Content Accessibility Guidelines 2.2](https://www.w3.org/TR/WCAG22/), Level AA.
