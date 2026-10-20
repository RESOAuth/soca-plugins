# Shared SOCA plugin requirements

These are high-level product and engineering targets for every Simple OAuth CIMD
Access (SOCA) plugin. They describe the desired common capabilities across host
applications, with platform-specific installation, account and permission models.

Each plugin should document requirements as **supported**, **partially supported**,
**planned** or **not applicable**, with the relevant version, evidence and any
operational workaround. A partial implementation should state its limitations;
listing a plugin in this repository does not establish that it meets every target.
Security checks needed for a supported login flow should fail closed.

## Functional requirements

### Provider connection and sign-in

- **F-01 — Provider configuration.** Provide sensible RESOAuth defaults and allow
  an administrator to configure another compatible OIDC/CIMD provider. Pin the
  trusted issuer, validate discovery and check required endpoints and capabilities.
  Explicitly unsupported CIMD should be rejected; missing discovery support should
  be reported as unconfirmed until a real login establishes interoperability.
- **F-02 — Secure login flow.** Use OIDC authorization code flow with PKCE S256,
  independent state and nonce, exact callback URLs and asymmetric
  `private_key_jwt` client authentication. Protect against request forgery, code
  replay and issuer mix-up, and bind each transaction to the initiating browser.
- **F-03 — Identity validation.** Verify the ID-token signature and required issuer,
  subject, audience, nonce and time claims before admitting a user. Validate
  authorization-response issuer and other conditional claims where applicable.
  UserInfo, when used, should belong to the same validated subject and should not
  override contradictory identity claims. OAuth access alone is not authentication.

### Client metadata: hosted and proxied CIMD

- **F-04 — Self-hosted documents.** Serve public client metadata and public JWKS
  over HTTPS without requiring login. Include the exact document URL as
  `client_id`, registered callback URLs and accurate client authentication and
  algorithm metadata. Use valid JSON, appropriate content types and cache headers.
- **F-05 — Proxy mode.** Support a configurable HTTPS metadata proxy, such as
  [Dynamic CIMD](https://github.com/RESOAuth/dynamic-cimd), when the application
  cannot host publicly reachable metadata, including local HTTPS development.
  Validate that the retrieved document matches the generated public metadata and
  exact client ID. Keep provider authentication independent of the metadata proxy.
- **F-06 — Public data boundary.** Publish only public client information and
  public keys, using `jwks_uri` or inline `jwks` as appropriate. Never publish
  private keys, client secrets, tokens, user claims or access-policy lists. Explain
  that proxy URLs and their encoded metadata are public and visible to the proxy.
- **F-07 — Client identity changes.** Prefer stable client IDs. Explain when a
  metadata or key change creates a new proxy URL/client ID, and its effect on
  consent, pairwise subjects and account association. Handle in-flight logins
  consistently by retaining their original configuration or invalidating them
  safely. Report provider restrictions on metadata/callback origins clearly.

### Keys and algorithm support

- **F-08 — Persistent keys.** Generate or provision strong, persistent asymmetric
  signing keys with stable key identifiers. Protect private material using the
  host's secret storage or encryption facilities; share the intended key set
  consistently across workers and retain it across restarts and upgrades.
- **F-09 — Safe client-key rotation.** Aim for automated, overlapping rotation:
  publish a replacement before use, allow for provider/CDN metadata and JWKS caches,
  then retain the old key long enough for outstanding transactions and assertions.
  Support operator-triggered rotation and emergency replacement. A plugin limited
  to manual rotation should document its maintenance window and recovery procedure.
- **F-10 — Algorithm profile.** Document client-assertion algorithms separately
  from ID-token verification algorithms. The common interoperability target is
  ES256/P-256 assertions and ES256 or RS256 ID tokens. Pin allowed algorithms and
  compatible key types; reject unsigned tokens, symmetric substitutions and silent
  downgrade. Additional algorithms, such as ES384 or PS256, require explicit
  provider/library support and tests before being advertised.
- **F-11 — Provider-key changes.** Support provider signing-key rollover with
  bounded JWKS retrieval/refresh and clear cache expiry rules. Unknown keys and
  invalid signatures should not trigger unlimited fetches or bypass validation.
  Check assertion claims and audience against the configured provider profile.

### Admission, existing accounts and new accounts

- **F-12 — Admission policy.** Support explicit email/domain allow and deny rules
  with documented matching semantics, deny precedence and safe defaults. Where
  email controls admission or account selection, require a validated email with
  `email_verified` equal to boolean `true` from the trusted provider. Apply policy
  on every SSO login, including existing users.
- **F-13 — Existing users.** Select exactly one eligible local account using the
  documented association policy. Respect disabled accounts, site membership and
  the host's access requirements. Reject ambiguous matches. Preserve local
  passwords, usernames, roles, groups and elevated privileges unless an explicit,
  separately authorized operation changes them; SSO should not reactivate users.
- **F-14 — New users.** Make first-login provisioning opt-in. Create an account
  only after authentication and admission succeed, with an explicitly configured,
  least-privileged role or group. Validate required local fields, avoid usable
  default passwords, and handle collisions, concurrent logins and failed creation
  without duplicate accounts, partial privileges or a session for an invalid user.
- **F-15 — Authorization boundary.** Keep authentication, admission and local
  permissions distinct. Provider profile or role claims should not grant
  administrative privileges implicitly. Document how removal, policy changes and
  disabling provisioning affect future logins and already established sessions.

### Claim association and profile mapping

- **F-16 — Account association.** State exactly which validated claims select an
  account: for example, current verified email or a persisted issuer/subject pair.
  Do not silently switch strategies. For email matching, explain that reassignment
  or changes of provider/local email can change the selected account, including an
  administrator account. For persisted links, define safe linking, unlinking and
  recovery without treating mutable profile claims as proof of account ownership.
- **F-17 — Identity lifecycle.** Define behavior when issuer, subject, client ID or
  email changes, and when an address is reused. Reject conflicting associations
  and provide an administrator-controlled recovery path. Do not infer that a new
  pairwise subject belongs to an existing user from a display name or username.
- **F-18 — Profile claims.** Support documented mappings for common claims such
  as `given_name`, `family_name`, `name` and `preferred_username` where the host has
  suitable fields. Separate profile updates from account selection and admission.
  Define precedence between signed ID-token claims and subject-matched UserInfo,
  validate types/lengths/content, handle missing claims and username collisions,
  and preserve local edits by default. Make overwrite or synchronization explicit.
- **F-19 — Claim diagnostics.** Provide a safe way to understand available claims
  and mapping decisions. If claim inspection is offered, make it privileged,
  explicitly enabled, short-lived and redacted, with clear capture scope and
  deletion. Do not retain raw tokens or turn ordinary login logs into claim dumps.

### Administration, user experience and recovery

- **F-20 — Configuration and validation.** Provide an appropriate admin UI or
  deployment configuration for provider, metadata mode, admission, provisioning and
  profile settings. Restrict changes to authorized administrators. Offer connection
  and document checks plus a real test-login path; distinguish their results.
- **F-21 — Login and logout.** Integrate a clear SSO entry point into the host's
  login experience, create normal host sessions after successful admission, and
  restrict post-login destinations to safe local URLs. Explain local logout versus
  provider logout. Automatic redirects, if offered, should preserve a recovery path.
- **F-22 — Local recovery.** Retain a usable local administrator login and a
  documented SSO bypass/disable procedure. Provider, proxy, key or configuration
  failures should produce actionable errors without locking administrators out or
  silently relaxing authentication checks.

## Non-functional requirements

- **N-01 — Secure implementation.** Use maintained protocol/cryptographic
  libraries and host security APIs. Protect admin actions against CSRF, escape
  displayed content, consume login transactions atomically once, and expire and
  clean up temporary state. Use secure host sessions and a strong browser binding;
  matching User-Agent strings alone is not sufficient.
- **N-02 — Network safety.** Validate TLS for outbound HTTPS, defend discovery,
  metadata and key fetches against SSRF and DNS rebinding, and bound response sizes,
  timeouts and redirects. CIMD fetches should reject redirects. Configure canonical
  external URLs and trusted reverse proxies rather than trusting arbitrary Host
  or forwarded headers.
- **N-03 — Privacy and secrets.** Request only necessary scopes/claims, minimize
  stored identity data, and document external services, data flows, retention and
  deletion. Keep secrets and tokens out of metadata, logs, errors, URLs and release
  artifacts; the protocol's short-lived callback code/state parameters should be
  redacted from access logs. Avoid undisclosed telemetry.
- **N-04 — Reliability and concurrency.** Work across supported multi-worker,
  multi-site and database deployments with correctly scoped configuration, shared
  state where needed and safe concurrent account creation/key maintenance. Fail
  safely during provider/proxy outages, interrupted maintenance and cache delays.
  Document backup, restore and lost-key behavior.
- **N-05 — Performance and abuse resistance.** Keep normal application requests
  inexpensive, use bounded caching where appropriate, rate-limit expensive login
  and diagnostic operations, and prevent refresh storms. Document cleanup and
  scheduled-maintenance needs, resource bounds and deployment rate-limit controls.
- **N-06 — Observability and usability.** Provide accessible login/admin screens,
  understandable errors and redacted diagnostic references. Distinguish protocol,
  metadata, admission and local-account failures for operators without exposing
  secrets or enabling public account enumeration. Audit sensitive configuration
  changes and account-association/provisioning operations appropriately.
- **N-07 — Compatibility and maintenance.** Publish supported host/runtime
  versions, dependency requirements, installation/upgrade/uninstall behavior,
  licensing, release notes and security reporting instructions. Use reproducible
  packaging, track dependency vulnerabilities and document migrations and rollback.
- **N-08 — Verification and interoperability.** Test the packaged plugin and
  supported platform combinations, including negative security cases and browser
  flows. Record provider profiles, algorithm coverage and dated live-login results.
  Keep simulated tests, successful document fetches and real end-user login evidence
  distinct; state untested behavior and known gaps explicitly.

## Evidence each plugin should maintain

Use the requirement IDs above in each plugin's support matrix, issues or release
checklist. At minimum, retain evidence for:

| Area | Representative checks |
| --- | --- |
| Metadata | Hosted and proxy documents, exact client ID, public-only keys, unavailable/incorrect documents and proxy URL changes. |
| Authentication | Successful login and cancellation; expired/replayed/wrong-browser callbacks; issuer, audience, nonce and signature failures; unsupported algorithms. |
| Key lifecycle | Restarts, multiple workers, provider rollover, client-key replacement, cache overlap and recovery after failure. |
| Accounts and claims | Existing and new users, provisioning disabled, deny precedence, unverified/missing email, duplicate and disabled accounts, concurrent creation, changed claims and preserved local edits/permissions. |
| Operations | Local recovery, outage behavior, redacted diagnostics, clean package install and supported-version upgrades. |

## Starting point and references

As reviewed on 20 October 2026, both existing plugins document verified-email
account selection. WordPress documents managed overlapping key rotation; Wagtail
documents a manually managed single key with a maintenance window. These are
implementation differences to track against the targets above. Consult the
plugins for current behavior:

- [WordPress overview](https://github.com/RESOAuth/wordpress-soca-plugin/blob/main/README.md)
  and [security decisions](https://github.com/RESOAuth/wordpress-soca-plugin/blob/main/docs/SECURITY.md).
- [Wagtail overview](https://github.com/RESOAuth/wagtail-soca-plugin/blob/main/README.md)
  and [security decisions](https://github.com/RESOAuth/wagtail-soca-plugin/blob/main/docs/SECURITY.md).

The targets above combine product choices with protocol requirements. Implementers
should follow the applicable specifications and document the revisions and
provider-specific profiles they implement:

- [OAuth Client ID Metadata Document, draft 02](https://www.ietf.org/archive/id/draft-ietf-oauth-client-id-metadata-document-02.html)
  — the CIMD revision reviewed for this initial requirements document; an
  Internet-Draft, not a final RFC.
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
  and [Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html).
- [OAuth security best current practice (RFC 9700)](https://www.rfc-editor.org/rfc/rfc9700.html)
  and [PKCE (RFC 7636)](https://www.rfc-editor.org/rfc/rfc7636.html).
- [JWT client authentication (RFC 7523)](https://www.rfc-editor.org/rfc/rfc7523.html),
  [JWT best current practices (RFC 8725)](https://www.rfc-editor.org/rfc/rfc8725.html)
  and [JWK (RFC 7517)](https://www.rfc-editor.org/rfc/rfc7517.html).
