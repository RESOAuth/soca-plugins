# SOCA plugins

**SOCA (Simple OAuth CIMD Access)** makes it easier to add single sign-on to an
application using an OpenID Connect provider that supports OAuth Client ID
Metadata Documents (CIMD).

A CIMD is a public JSON document describing the application, its callback URLs
and public signing keys. Its HTTPS URL is the OAuth `client_id`, allowing a
compatible provider to discover the client without a separate manual client
registration. SOCA publishes that metadata, completes the sign-in flow and applies
the application's rules for admitting users and selecting or creating local
accounts. The application continues to manage its own users, permissions and
sessions.

## Existing plugins

| Application | Repository | Purpose |
| --- | --- | --- |
| WordPress | [wordpress-soca-plugin](https://github.com/RESOAuth/wordpress-soca-plugin) | Adds SOCA sign-in to WordPress. |
| Wagtail | [wagtail-soca-plugin](https://github.com/RESOAuth/wagtail-soca-plugin) | Adds SOCA sign-in to the Wagtail admin. |

Each plugin's repository contains its installation instructions, supported versions,
release status and security guidance.

## Shared requirements

[requirements.md](requirements.md) describes the high-level functional and
non-functional requirements defining the best-case target for every SOCA plugin,
including self-hosted and proxied CIMD, key rotation, signing algorithms, new and
existing users, claim association, security and operations. They set the highest
bar for the finished product. Current plugin limitations are gaps to close; each
plugin documents its progress against the requirements.

Every plugin must provide complete code/deployment configuration for headless
setup. Where the host already has an admin GUI, plugins must integrate sensible
settings into it. A headless platform does not require a new GUI.

SOCA is developed by [RESOAuth](https://github.com/RESOAuth). The plugins start
with RESOAuth provider settings and can be configured for other compatible
providers. The shared requirements set
`https://dynamic-cimd.resoauth.cloud/cimd/%s`, provided by
[Dynamic CIMD](https://github.com/RESOAuth/dynamic-cimd), as the proxy-mode default
and require code/deployment configuration plus settings in any existing admin GUI.
