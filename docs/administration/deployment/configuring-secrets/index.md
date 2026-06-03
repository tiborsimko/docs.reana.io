# Configuring deployment secrets

A REANA cluster relies on a small set of deployment credentials and
an application secret key that the Helm chart will not invent for
you. Every deployment must supply them explicitly via Helm values
(or `--set` flags). The chart fails fast with a clear `required`
error on any unset key, so missing a secret is loud rather than
silent.

This page lists the secrets the chart needs, separates the keys
every deployment must set from those that only matter when an
optional feature is enabled, and points at recipes for both local
development and production setups.

## Secrets required for every deployment

The following keys must be set on every REANA installation,
regardless of which optional features you enable:

- `secrets.reana.REANA_SECRET_KEY` — Flask application secret key,
  used to sign session cookies and email confirmation tokens, and
  as the encryption key for sensitive database columns. Must be a
  strong random value; a 32-byte hex string (`openssl rand -hex 32`)
  is the recommended size.
- `secrets.database.user`, `secrets.database.password` — PostgreSQL
  username and password used by REANA components.
- `secrets.message_broker.user`, `secrets.message_broker.password` —
  RabbitMQ username and password.
- `secrets.cache.user`, `secrets.cache.password` — Redis ACL
  username and password for the cache (session store) component.

## Secrets required only when an optional provider is enabled

The chart refuses to render a half-configured provider (some
sub-keys set, others missing); supply all of a group together or
none of it.

### OpenSearch

Required only when `opensearch.enabled: true`:

- `secrets.opensearch.user`, `secrets.opensearch.password` — Basic
  Auth credentials REANA components use to connect to OpenSearch.
- `opensearch.initialAdminPassword` — bootstrap password for the
  OpenSearch admin user. Required only when
  `opensearch.securityConfig.enabled: false` (the chart skips the
  bundled security configuration and you provision the admin user
  externally).
- `opensearch.customSecurityConfig.internalUsers.<user>.hash` —
  bcrypt password hash for each OpenSearch internal user (`reana`,
  `fluentbit`). Required only when
  `opensearch.securityConfig.enabled: true` (the default path,
  where the chart applies its own security configuration).
  Generate with the OpenSearch security plugin's hash tool, e.g.
  `plugins/opensearch-security/tools/hash.sh -p <new-password>`.

### GitLab integration

Required only if you want users to launch workflows from GitLab
repositories. Set all three together:

- `secrets.gitlab.REANA_GITLAB_HOST`
- `secrets.gitlab.REANA_GITLAB_OAUTH_APP_ID`
- `secrets.gitlab.REANA_GITLAB_OAUTH_APP_SECRET`

### CERN / EOSC SSO

Required only if you want users to authenticate via CERN SSO or
EOSC SSO. Set the consumer key and secret together:

- `secrets.cern.sso.CERN_CONSUMER_KEY`,
  `secrets.cern.sso.CERN_CONSUMER_SECRET`
- `secrets.eosc.sso.EOSC_CONSUMER_KEY`,
  `secrets.eosc.sso.EOSC_CONSUMER_SECRET`

When an SSO provider is configured, REANA disables local account
registration and local username/password login: SSO and local
accounts are mutually exclusive.

## Local development installations

For laptop-style single-user clusters, REANA provides an example
`etc/myvalues.yaml` file in the umbrella repository with weak,
clearly-labelled placeholder values (e.g.
`reana-db-password-for-development`):

```{ .console .copy-to-clipboard }
$ wget https://raw.githubusercontent.com/reanahub/reana/master/etc/myvalues.yaml
$ helm install reana reanahub/reana -f myvalues.yaml
```

The placeholder values are intentionally loud ("for-development")
so they fail an obvious "is this production?" sniff test. See
[Deploying locally](../deploying-locally/index.md) for the full
local setup recipe.

## Production installations

For production deployments, generate a strong random value for
every required secret, and for every optional provider you enable,
and keep them in a private values file that is **not** committed
to a public repository:

```{ .console .copy-to-clipboard }
$ openssl rand -hex 32       # for REANA_SECRET_KEY
$ openssl rand -base64 24    # for database/message-broker/cache passwords
```

A typical private `values.yaml` skeleton:

```yaml
secrets:
  reana:
    REANA_SECRET_KEY: <openssl rand -hex 32 output>
  database:
    user: reana
    password: <strong random value>
  message_broker:
    user: reana
    password: <strong random value>
  cache:
    user: reana
    password: <strong random value>
```

For sharing private values across an operator team, a secret
manager (HashiCorp Vault, Bitwarden, sealed-secrets, …) is
strongly preferred over storing the file unencrypted at rest. See
[Deploying at scale](../deploying-at-scale/index.md) for the full
production setup recipe.
