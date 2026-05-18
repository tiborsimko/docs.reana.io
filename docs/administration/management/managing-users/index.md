# Managing users

## Managing users by administrators

To manage users you will need to obtain administration credentials:

```{ .console .copy-to-clipboard }
$ export KUBECONFIG=~/mycluster/config
$ export REANA_ACCESS_TOKEN=$(kubectl get secret reana-admin-access-token -o json | jq -r '.data | map_values(@base64d) | .ADMIN_ACCESS_TOKEN')
```

### Create users

```console
$ kubectl exec -i -t deployment/reana-server -- flask reana-admin user-create --email john.doe@example.org --admin-access-token $REANA_ACCESS_TOKEN
User was successfully created.
ID                                     EMAIL                  ACCESS_TOKEN
aa37d63d-3186-45d5-aa40-5d221cb170bf   john.doe@example.org   xxxxxxxxxxxx
```

If you have enabled [Single Sign-On (SSO) access](../../configuration/configuring-access/#user-registration-via-single-sign-on), then user creation via
CLI is not necessary since users will be created via SSO when they first
sign in.

### List users

```console
$ kubectl exec -i -t deployment/reana-server -- flask reana-admin user-list --admin-access-token $REANA_ACCESS_TOKEN
ID                                     EMAIL                      ACCESS_TOKEN                    ACCESS_TOKEN_STATUS
b5ff2c90-d2aa-4455-805d-599990043c39   john.doe@example.org       xxxxxxxxxxxx                    active
6d0a83d3-a5fb-415e-bc90-e2abed807ffe   new.web.user@example.org   None                            requested
```

### Grant access tokens

```console
$ kubectl exec -i -t deployment/reana-server -- flask reana-admin token-grant --email new.web.user@example.org --admin-access-token $REANA_ACCESS_TOKEN
Token for user aa37d63d-3186-45d5-aa40-5d221cb170bf (new.web.user@example.org) granted.

Token: c0fa47fa00ae4013a13fd7n
```

### Revoke access tokens

```console
$ kubectl exec -i -t deployment/reana-server -- flask reana-admin token-revoke --email new.web.user@example.org --admin-access-token $REANA_ACCESS_TOKEN
User token c0fa47fa00ae4013a13fd7n (new.web.user@example.org) was successfully revoked.
```

If you need to revoke user tokens remotely without Kubernetes cluster access,
you can optionally enable a token-management REST API endpoint:

```yaml
components:
  reana_server:
    environment:
      REANA_TOKEN_MANAGEMENT_SECRET: "my_secret"
```

When this secret is non-empty, you can revoke a user's active token with:

```console
$ curl -X DELETE https://reana.example.org/api/token \
    -H 'Content-Type: application/json' \
    -H "X-Token-Management-Secret: $REANA_TOKEN_MANAGEMENT_SECRET" \
    --data '{"email": "new.web.user@example.org"}'
```

You can also identify the user by REANA user ID instead of email, for example:

```console
$ curl -X DELETE https://reana.example.org/api/token \
    -H 'Content-Type: application/json' \
    -H "X-Token-Management-Secret: $REANA_TOKEN_MANAGEMENT_SECRET" \
    --data '{"user_id": "aa37d63d-3186-45d5-aa40-5d221cb170bf"}'
```

If `REANA_TOKEN_MANAGEMENT_SECRET` is empty, this endpoint is disabled.

Note that when `REANA_ACCESS_TOKEN_ISSUANCE_POLICY=auto`, revoking a token only
invalidates the current token; the user will obtain a new token on the next
login.

### Export users

```{ .console .copy-to-clipboard }
$ kubectl exec -i -t deployment/reana-server -- flask reana-admin user-export --admin-access-token $REANA_ACCESS_TOKEN > myusers.csv
```

### Import users

```{ .console .copy-to-clipboard }
$ # put myusers.csv onto the node in the /var/reana directory and run:
$ kubectl exec -i -t deployment/reana-server -- flask reana-admin user-import --admin-access-token $REANA_ACCESS_TOKEN --file /var/reana/myusers.csv
```
