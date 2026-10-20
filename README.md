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
non-functional requirements each SOCA plugin should aim to support, including
self-hosted and proxied CIMD, key rotation, signing algorithms, new and existing
users, claim association, security and operations. These are shared development
targets; each plugin documents its current coverage and limitations.

SOCA is developed by [RESOAuth](https://github.com/RESOAuth). The plugins start
with RESOAuth provider settings and can be configured for other compatible
providers. [Dynamic CIMD](https://github.com/RESOAuth/dynamic-cimd) is a related
metadata hosting service that can be used for proxy mode.
