# Horreum Operator

This operator installs [Horreum](https://github.com/Hyperfoil/Horreum), services it depends on - PostgreSQL database and Keycloak SSO - and for convenience also [Hyperfoil Report](https://github.com/Hyperfoil/report).

See an [example](deploy/crds/hyperfoil.io_v1alpha1_horreum_cr.yaml) of the `horreum` resource:

```yaml
apiVersion: hyperfoil.io/v1alpha1
kind: Horreum
metadata:
  name: example-horreum
spec:
  route:
    host: horreum.apps.mycloud.example.com
  keycloak:
    route:
      host: keycloak.apps.mycloud.example.com
  postgres:
    persistentVolumeClaim: horreum-postgres
  report:
    route:
      host: hyperfoil-report.apps.mycloud.example.com
    persistentVolumeClaim: hyperfoil-report
```

For detailed description of all properties [refer to the CRD](deploy/olm-catalog/horreum-operator/0.1.0/hyperfoil.io_horreums_crd.yaml).

When using persistent volumes make sure that the access rights are set correctly and the pods have write access; in particular the PostgreSQL database requires that the mapped directory is owned by user with id `999`.

If you're planning to use secured routes (edge termination) it is recommended to set the `tls: my-tls-secret` at the first deploy; otherwise it is necessary to update URLs for clients `horreum` and `horreum-ui` in Keycloak manually. Also the Horreum pod needs to be restarted after keycloak route update.

Currently you must set both Horreum and Keycloak route host explicitly, otherwise you could not log in (TODO).

When the `horreum` resource gets ready, login into Keycloak using administrator credentials (these are automatically created if you don't specify existing secret) and create a new user in the `hyperfoil` realm, a new team role (with `-team` suffix) and assign it to the user along with other appropriate predefined roles. Administrator credentials can be found using this:

```sh
NAME=$(oc get horreum -o jsonpath='{$.items[0].metadata.name}')
oc get secret $NAME-keycloak-admin -o json | \
    jq '{ user: .data.user | @base64d, password: .data.password | @base64d }'
```

For details of roles in Horreum please refer to [its documentation](https://github.com/Hyperfoil/Horreum)

## Hyperfoil integration

For your convenience this operator creates also a config map (`*-hyperfoil-upload`) that can be used in [Hyperfoil resource](https://github.com/Hyperfoil/hyperfoil-operator) to upload Hyperfoil results to this instance - you can use it directly or merge that into another config map you use for post-hooks. However, it is necessary to define & mount a secret with these keys:

```sh
# Credentials of the user you've created in Keycloak
HORREUM_USER=user
HORREUM_PASSWORD=password
# Role for the team the user belongs to (something you've created)
HORREUM_GROUP=engineers-team
# This is a non-confidential client ID we'll reuse to login for upload
HORREUM_CLIENT_ID=horreum-ui

oc create secret generic hyperfoil-horreum \
    --from-literal=HORREUM_USER=$HORREUM_USER \
    --from-literal=HORREUM_PASSWORD=$HORREUM_PASSWORD \
    --from-literal=HORREUM_GROUP=$HORREUM_GROUP \
    --from-literal=HORREUM_CLIENT_ID=$HORREUM_CLIENT_ID \
```

Then set it up in the `hyperfoil` resource:

```yaml
apiVersion: hyperfoil.io/v1alpha1
kind: Hyperfoil
metadata:
  name: example-hyperfoil
  namespace: hyperfoil
spec:
  # ...
  postHooks: example-horreum-hyperfoil-upload
  secretEnvVars:
  - hyperfoil-horreum
```

This operator automatically inserts a webhook to convert test results into Hyperfoil report; In order to link from test to report you have to add a schema (matching the URI used in your Hyperfoil version, usually something like `http://hyperfoil.io/run-schema/0.8` and add it an extractor `info` with JSON path `$.info`. Subsequently go to the test and add a view component with header 'Report', accessor you've created in the previous step and this rendering script (replacing the hostname):

```js
(value, all) => {
  let info = JSON.parse(value);
  return '<a href="http://example-horreum-report-hyperfoil.apps.mycloud.example.com/' + all.id + '-' + info.id +'-' + info.benchmark + '.html" target=_blank>Show</a>'
}
```