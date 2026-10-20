# Shared SOCA plugin requirements

These are the best-case requirements for Simple OAuth CIMD Access (SOCA) plugins,
independent of implementation cost or current capabilities. They cover browser-based
OIDC sign-in, admission, local accounts and host sessions. Declare the protected
login surfaces; API authorisation, other grants and delegated access need separate
profiles.

Imperatives define the target. Track each ID as supported, partially supported or
planned, with versioned evidence and gaps (N-09). Use not applicable only for a
justified platform/protocol constraint. Operator-disabled features remain required
capabilities; workarounds and incremental delivery do not lower the bar.

## Functional requirements

### Provider connection and sign-in

- **F-01: Provider configuration.** Provide sensible RESOAuth defaults and multiple
  explicitly trusted OIDC/CIMD profiles, isolating issuer, tenant/site, algorithm and
  admission policies. Validate discovery, exact issuer, endpoints and capabilities
  before activation. Reject explicit CIMD non-support; missing support remains
  unconfirmed until a real login succeeds. Never derive issuer trust from claims.
- **F-02: Secure login flow.** Use OIDC authorisation code flow, PKCE S256,
  independent state and nonce, exact callbacks and asymmetric `private_key_jwt`.
  Prevent CSRF, code replay and issuer mix-up with F-25 transaction/browser binding.
- **F-03: Identity validation.** Validate ID-token signatures, issuer, subject,
  audience, nonce, times and conditional claims, including response issuer.
  Match UserInfo to the validated subject and reject conflicting identity claims.
  Support fresh authentication and enforce configured assurance using validated
  `auth_time`, `acr` and `amr` with documented provider semantics. Email verification does not
  establish MFA; OAuth access does not establish identity.

### Client metadata

- **F-04: Metadata and self-hosting.** Serve public CIMD and JWKS over HTTPS without
  login. Include the exact document URL as `client_id`, callbacks and accurate
  authentication/algorithm metadata. Use valid JSON, suitable content types,
  explicit cache lifetimes and predictable invalidation. Verify public reachability
  and contents before enabling login or promoting keys.

  In both hosted and proxy modes, populate `client_name`, `client_uri`, `logo_uri`,
  `policy_uri` (privacy policy) and `tos_uri` (terms) from application identity and
  published pages exposed by the CMS/platform to plugins. Use these defaults where
  available, with overrides under F-20. Identify the application, using its own
  branding/legal pages and public HTTPS URLs; omit unavailable optional fields.
- **F-05: Proxy mode.** Support publicly hosted metadata for applications without
  public document endpoints, including local HTTPS development. Default to
  `https://dynamic-cimd.resoauth.cloud/cimd/%s`. Make the template configurable
  under F-20. Replace one `%s` in the HTTPS path with unpadded base64url-encoded
  UTF-8 public metadata JSON. Validate template safety and exact returned metadata,
  including `client_id` equal to the resulting URL. Provide URL preview, document
  check and reset to default. Keep provider authentication independent of the proxy
  and preserve the hosted mode's security guarantees.
- **F-06: Public data boundary.** Support `jwks_uri` and inline `jwks` where the
  provider permits them. Publish only public client information/keys, never private
  keys, secrets, tokens, user claims or admission rules. Explain that proxy URLs,
  including encoded metadata, are public and visible to the proxy.
- **F-07: Client identity changes.** Keep client IDs stable where possible. Control
  metadata/key migrations that change IDs, explaining consent, pairwise-subject and
  account-link effects. Preserve routine in-flight logins using bounded snapshots;
  invalidate affected transactions immediately on security revocation. Check
  provider metadata/callback-origin restrictions before changes.

### Keys and algorithms

- **F-08: Persistent keys.** Provision strong asymmetric keys with stable IDs.
  Protect private material in secret storage or authenticated encryption separate
  from application data. Support external key management and non-exportable keys.
  Coordinate workers, survive restarts/upgrades and test backup, restore and
  lost-key recovery.
- **F-09: Automated rotation.** Rotate client keys unattended, with overlap and no
  planned SSO outage in hosted and proxy modes. Confirm publication, respect
  provider/CDN cache bounds, warm replacements and retain retiring keys through
  outstanding transactions/assertions. Promote atomically across workers; monitor
  overdue/failing rotation and support safe retries, manual triggers and emergency
  revocation.
- **F-10: ES256 default and algorithm agility.** Use **ES256/P-256** as the required
  default and recommended profile for assertion signing and ID-token verification.
  Generate P-256 client keys and explicitly set `token_endpoint_auth_signing_alg`
  and `id_token_signed_response_alg`
  to `ES256`. Sign with client keys; verify with trusted issuer keys. Configure
  purposes independently, enabling alternatives only with explicit administrator
  approval for that issuer/purpose. Pin compatible algorithms/key types to the
  intersection of local policy and provider support. Reject unsigned tokens,
  symmetric substitutions, deprecated profiles and silent fallback, including
  selection driven solely by token headers or provider ordering.

  Also support `ES384`, `ES512`, `PS256`, `PS384`, `PS512`, `RS256`, fully specified
  EdDSA (`Ed25519`, `Ed448`) and standardised `ML-DSA-44`, `ML-DSA-65`, `ML-DSA-87`,
  with orderly migration and published test vectors. Support does not imply enabled
  defaults. Report provider incompatibility clearly; missing plugin support is a gap.
- **F-11: Provider-key changes.** Bound JWKS fetches/refreshes, expiry and unknown-key
  or signature-failure retries. Isolate caches by issuer/purpose; validate assertion
  claims and audience against that profile. Honour known revocations and expired
  trust without accepting stale keys for availability.

### Admission and accounts

- **F-12: Admission policy.** Deny by default; provide email/domain allow and deny
  rules, deny precedence, sensitive-account/site controls and policy preview. Apply
  admission on every login. Email-based rules require a validated email and boolean
  `email_verified: true` from the trusted issuer.

  Validate DNS name syntax and length on save and startup. Accept exact domains or
  one leading `*.` followed by a dotted domain: `example.com` or `*.example.com`. Reject
  `*example.com`, embedded/repeated wildcards, URLs, email addresses, empty labels
  and single-label domains/suffixes. Match case-insensitively on whole DNS labels:
  exact rules match only that domain; `*.example.com` matches one or more subdomain
  labels, including `a.b.example.com`, but neither `example.com` nor
  `badexample.com`. Report invalid rules; never silently ignore or broaden them.
- **F-13: Existing users.** Resolve exactly one eligible account; reject ambiguity,
  disabled users and missing site/access permissions. Preserve local credentials,
  usernames, roles, groups and privileges unless separately authorised. Never
  reactivate an account through SSO.
- **F-14: New users.** Make provisioning opt-in, after authentication and admission,
  into an explicitly configured least-privileged role/group. Validate local fields;
  handle collisions, concurrent creation and failures atomically without duplicate
  accounts, usable default passwords, partial privileges or invalid-user sessions.
- **F-15: Authorisation boundary.** Separate identity, admission and local permissions.
  Support explicit allowlisted claim-to-role/group mappings with issuer binding,
  privilege ceilings and audited changes. Never grant implicit admin privileges;
  elevation needs independent local approval. Enforce removal/policy revocation for
  future logins and affected SSO sessions within a documented propagation bound.

### Association and claims

- **F-16: Protected association.** Persist unique issuer/subject links for returning
  users. Linking an existing account requires local reauthentication or audited
  administrator approval; verified email may suggest a match, never establish
  ownership alone. Provide linking, unlinking and recovery, prevent concurrent
  conflicting links and separate links from admission/profile data.
- **F-17: Identity lifecycle.** Handle changed issuer, subject, client ID or email
  and reused addresses through audited, administrator-controlled migration/recovery.
  Reject conflicts, duplicates and silent relinking. Never infer a replacement
  pairwise subject from mutable profile claims.
- **F-18: Profile claims.** Map `given_name`, `family_name`, `name`,
  `preferred_username` and allowlisted custom claims to suitable fields. Provide
  preview and per-field fill-empty, create-only or synchronisation policies.
  Define ID-token/UserInfo precedence; validate types, lengths and content, handle
  missing claims/collisions, preserve local edits by default and audit mapping
  changes. Keep profile mapping separate from selection/admission; prohibit writes
  to credentials, identity links and privilege fields.
- **F-19: Claim diagnostics.** Provide privileged, opt-in, short-lived, redacted
  inspection scoped to a selected test transaction. Explain captured data, encrypt
  retained reports and delete automatically on expiry. Never retain raw tokens or
  include claim dumps in routine logs.

### Configuration, sessions and operations

- **F-20: Host UI and headless configuration.** Provide documented code/deployment
  configuration for every supported setting. Support headless installation, setup,
  changes, checks, status, enable/disable and maintenance without visiting an admin
  UI or saving a form. Integrate accessible settings into an existing host admin
  GUI only; do not invent a separate GUI. Follow host conventions with clear inputs,
  sensible defaults, help and actionable validation. Keep secrets deployment-managed;
  any UI handles references and redacted status.

  Share defaults, validation and security policy across code/UI. Document precedence;
  expose effective values/sources through non-UI tools and any existing GUI, with
  deployment-locked fields read-only. Make repeated deployment safe/reproducible;
  audit changes, reject stale edits and support rollback. Require document checks
  and a real test login before profile activation, reporting results separately.
  Test mode previews admission/mapping without account creation, linking, permission
  changes or replacing the operator's session.
- **F-21: Login/logout.** Use host login entry points/sessions and safe local return
  URLs. Support session lifetime, reauthentication, revocation, local logout and
  advertised RP-initiated/back-channel logout. Validate provider logout messages;
  local logout must survive provider outages. Optional automatic redirects need
  loop prevention and local recovery.
- **F-22: Local recovery.** Retain local administrator login and documented SSO
  bypass/disable procedures. Protect recovery access with strong local
  authentication, least privilege and audit. Test before rollout/after upgrades;
  failures must be actionable without lockout or weakened authentication.
- **F-23: Host/plugin lifecycle.** Respect host account, permission, session,
  password-reset and extension APIs. Support canonical HTTPS, trusted reverse
  proxies and path prefixes without exposing protected routes. Define install,
  upgrade, disable, re-enable and uninstall behaviour. Disable stops new SSO and
  invalidates pending transactions; define existing-session handling. Preserve local
  accounts, permissions and recovery. Make permanent deletion explicit, scoped and
  audited, covering keys, links and backup implications.
- **F-24: Token lifecycle.** Request minimum scopes for configured features; ordinary
  sign-in needs neither offline access nor retained refresh tokens. Discard codes
  and access tokens after use. Retain ID tokens only for a configured protocol need,
  encrypted, access-controlled and expiring. Separate delegated-access profiles must
  define storage, audience/resource binding, refresh rotation, revocation and deletion.
- **F-25: Platform sessions and browser storage.** Reuse CMS/platform/framework
  session/security facilities. Protect the per-transaction PKCE **verifier**, expected
  nonce and state association with confidentiality and tamper resistance; derive
  the S256 challenge from the verifier. Bind transactions to browser, issuer,
  client ID and callback; User-Agent matching alone is insufficient. Expire,
  atomically consume once across workers and clean up temporary state; support
  parallel tabs and rotate the authenticated session ID.

  Favour stateless plugin operation using platform session/crypto/replay facilities,
  avoiding duplicate stores and new cookies. Browser-carried state needs integrity,
  encryption of secrets, expiry and browser binding. Signing/encryption alone cannot
  establish browser binding or single use; keep minimal shared replay state if needed.
  Justify any extra cookie; minimise its scope/lifetime and use `Secure`, `HttpOnly`
  and appropriate `SameSite`. Avoid `localStorage`, `sessionStorage` and IndexedDB;
  unavoidable use needs documented necessity/security review. Never put secret
  transaction material in JavaScript-readable storage or plaintext URLs. Statelessness
  and cookie avoidance must preserve confidentiality, browser binding and replay checks.

## Non-functional requirements

- **N-01: Secure implementation.** Use maintained protocol/cryptographic libraries,
  host security APIs, a maintained threat model, secure defaults and least privilege.
  Protect admin actions against CSRF, escape output and apply F-25. Do not permit
  production bypasses of protocol, certificate, identity or authorisation checks.
- **N-02: Network safety.** Verify HTTPS/TLS, prevent SSRF/DNS rebinding and bound
  response sizes, total deadlines and redirects; reject CIMD fetch redirects. Never
  trust arbitrary Host/forwarded headers for canonical origins. Support restricted
  egress and test IPv4/IPv6, mixed DNS answers and slow/large responses.
- **N-03: Privacy/secrets.** Minimise identity storage; document external services,
  data flows, retention, deletion and backups. Encrypt sensitive retained data and
  delete automatically. Exclude secrets/tokens from logs, errors, URLs and releases;
  redact callback codes/state from access logs. Telemetry must be opt-in and exclude
  authentication data.
- **N-04: Reliability.** Support scoped multi-site/multi-worker/database deployments,
  shared state and safe concurrent provisioning/key maintenance. Routine maintenance
  and rolling upgrades must avoid planned SSO outages. Publish availability/recovery
  objectives; test failover/restore. Fail closed on outages/expired trust; restore
  must not revive revoked credentials, links or transactions.
- **N-05: Performance/abuse.** Keep normal requests inexpensive; bound caches, login
  and diagnostic rates, refreshes, storage and cleanup. Document scheduling and
  deployment rate controls. Publish latency/throughput/resource budgets and load-test
  evidence, including sustained abuse.
- **N-06: Observability/usability.** Meet WCAG 2.2 AA for plugin-owned screens,
  including keyboard/assistive-technology checks. Provide clear errors/redacted
  references that distinguish protocol, metadata, admission and account failures
  without secrets or enumeration. Audit configuration, links and provisioning using
  access-controlled, tamper-evident records with retention limits. Expose health,
  rotation and security/availability metrics without identity data.
- **N-07: Maintenance/releases.** Document supported hosts/runtimes, dependencies,
  licensing, installation/upgrades/uninstall, releases and security reporting.
  Provide reproducible packages, signed artefacts/provenance, an SBOM, immutable
  dependency/workflow references and vulnerability checks. Publish response/patch
  policy, block releases with unresolved critical/high findings and test migrations/
  rollback.
- **N-08: Verification.** Test packaged releases/platform combinations, negative
  security cases, browsers, concurrency, faults and recovery. Run applicable OIDC
  conformance suites, protocol/security checks and independent security review with
  tracked remediation. Record provider/algorithm coverage, vectors and dated live
  logins in both metadata modes. Distinguish simulations, document checks and real
  logins; disclose gaps.
- **N-09: Implementation evidence.** Publish the assessed requirements revision and
  per-ID support/version/evidence/gaps, provider profiles and protected login
  surfaces. Document security defaults, units, ranges and hard limits for
  transaction/assertion lifetimes, clock skew, rotation, metadata/JWKS caching,
  session revocation and diagnostic retention. Explain derived overlap/retention, unconfirmed
  provider cache bounds and operator choices. Supply reproducible fixtures and
  acceptance procedures with dated results.

## Acceptance evidence

Verify each supported ID, including failure and concurrency cases. Explicitly test:

| Area | Cases |
| --- | --- |
| Domains | Exact/apex/subdomain/nested matches; `*.example.com` versus invalid `*example.com`; deny precedence and rejected malformed rules. |
| Metadata/configuration | Hosted/default/custom proxy checks; platform name, branding/legal URLs, overrides and omitted unavailable fields in both modes; wrong documents/IDs, changed URLs, clean headless setup, prior UI settings, precedence/locks and repeat deployment. |
| Authentication/algorithms | Cancellation, wrong issuer/audience/nonce/signature, ES256 defaults, independent overrides, prohibited fallback and vectors for every algorithm. |
| Transactions/sessions | Copied callbacks, replay, parallel tabs, state tampering/expiry, fixation, logout/revocation/assurance and provider outages. |
| Keys/accounts | Cache overlap, rollover/revocation across workers, duplicate/disabled/reassigned accounts, identity migrations, concurrent creation/linking and preserved local edits/permissions. |
| Operations | Recovery, failover/restore, upgrades, accessibility/load, release provenance, test-mode side effects, disable/uninstall, token disposal and scoped deletion. |

## Standards

Implement and record applicable revisions. CIMD draft 02 is an Internet-Draft;
ES256 as default is a SOCA product choice. Registry listing alone does not prove
implementation safety or provider support.

- [CIMD draft 02](https://www.ietf.org/archive/id/draft-ietf-oauth-client-id-metadata-document-02.html).
- OIDC [Core](https://openid.net/specs/openid-connect-core-1_0.html),
  [Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html) and
  [client metadata](https://openid.net/specs/openid-connect-registration-1_0.html#ClientMetadata)
  (metadata vocabulary; no dynamic registration endpoint required).
- [OAuth security BCP](https://www.rfc-editor.org/rfc/rfc9700.html) and
  [PKCE](https://www.rfc-editor.org/rfc/rfc7636.html).
- [JWT assertions](https://www.rfc-editor.org/rfc/rfc7523.html),
  [JWT BCP](https://www.rfc-editor.org/rfc/rfc8725.html),
  [JWK](https://www.rfc-editor.org/rfc/rfc7517.html),
  [algorithms](https://www.rfc-editor.org/rfc/rfc7518.html) and
  [JOSE registry](https://www.iana.org/assignments/jose).
- [Client metadata fields](https://www.rfc-editor.org/rfc/rfc7591.html#section-2).
- [RP-initiated logout](https://openid.net/specs/openid-connect-rpinitiated-1_0.html)
  and [back-channel logout](https://openid.net/specs/openid-connect-backchannel-1_0.html).
- [WCAG 2.2](https://www.w3.org/TR/WCAG22/), Level AA.
