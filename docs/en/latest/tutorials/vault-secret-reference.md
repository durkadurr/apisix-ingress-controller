---
title: Reference Vault Secrets in ApisixConsumer
---

<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

APISIX has a built-in secret manager that fetches credentials from external providers such as HashiCorp Vault at runtime. You can use the `$secret://` URI scheme inside `ApisixConsumer` auth parameter value fields to reference secrets stored in Vault instead of embedding them in your manifests or Kubernetes Secrets.

The APISIX Ingress Controller passes these reference strings to APISIX as-is — no resolution happens in the controller. APISIX resolves the value from Vault when the consumer is first used.

## Prerequisites

1. HashiCorp Vault is running and accessible from your APISIX data plane pods.
2. APISIX is configured with a Vault secret provider (see below).

## Configure Vault in APISIX

How you register a Vault secret provider depends on your APISIX deployment mode.

### etcd mode (traditional deployment)

Use the Admin API to register the provider. It is stored in etcd and picked up at runtime:

```bash
curl -X PUT http://<apisix-admin>:9180/apisix/admin/secrets/vault/1 \
  -H "X-API-KEY: <admin-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "uri": "http://vault.default.svc.cluster.local:8200",
    "prefix": "kv/apisix",
    "token": "<vault-token>"
  }'
```

### Standalone mode (apisix.yaml)

Add a `secrets` section to your `apisix.yaml`:

```yaml
secrets:
  - id: vault/1
    uri: http://vault.default.svc.cluster.local:8200
    prefix: kv/apisix
    token: <vault-token>
#END
```

---

The `id` field follows the format `{manager}/{id}` where:

| Segment     | Description                                                         |
|-------------|---------------------------------------------------------------------|
| `vault`     | Secret manager type. Built-in options: `vault`, `aws`, `gcp`.      |
| `1`         | Numeric ID referenced in `$secret://vault/1/...` URIs.             |

| Field    | Description                                                                  |
|----------|------------------------------------------------------------------------------|
| `uri`    | Address of the Vault server reachable from the APISIX pod.                   |
| `prefix` | KV mount path in Vault. The full lookup path becomes `{prefix}/{path}`.      |
| `token`  | Vault authentication token.                                                  |

## Secret URI Format

```
$secret://vault/<id>/<path>/<key>
```

With `prefix: kv/apisix` and the reference `$secret://vault/1/jack/secret`, APISIX looks up the `secret` key at Vault path `kv/apisix/jack`.

## Store Credentials in Vault

Write the consumer credentials to Vault before creating the `ApisixConsumer`:

```shell
# key-auth
vault kv put kv/apisix/jack key="my-secret-api-key"

# jwt-auth
vault kv put kv/apisix/jack \
  secret="my-jwt-secret" \
  public_key=@/path/to/public.pem \
  private_key=@/path/to/private.pem
```

## Create an ApisixConsumer with Vault References

### Key Authentication

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixConsumer
metadata:
  namespace: ingress-apisix
  name: jack
spec:
  ingressClassName: apisix
  authParameter:
    keyAuth:
      value:
        key: $secret://vault/1/jack/key
```

### JWT Authentication

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixConsumer
metadata:
  namespace: ingress-apisix
  name: jack
spec:
  ingressClassName: apisix
  authParameter:
    jwtAuth:
      value:
        key: jack
        secret: $secret://vault/1/jack/secret
```

For asymmetric algorithms (RS256, ES256):

```yaml
    jwtAuth:
      value:
        key: jack
        algorithm: RS256
        public_key: $secret://vault/1/jack/public_key
        private_key: $secret://vault/1/jack/private_key
```

### Basic Authentication

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixConsumer
metadata:
  namespace: ingress-apisix
  name: jack
spec:
  ingressClassName: apisix
  authParameter:
    basicAuth:
      value:
        username: jack
        password: $secret://vault/1/jack/password
```

### HMAC Authentication

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixConsumer
metadata:
  namespace: ingress-apisix
  name: jack
spec:
  ingressClassName: apisix
  authParameter:
    hmacAuth:
      value:
        access_key: jack
        secret_key: $secret://vault/1/jack/secret_key
```

## Apply and Verify

```shell
kubectl apply -f consumer.yaml
kubectl get apisixconsumer -n ingress-apisix jack
```

To verify, enable key authentication on a route and send a request using the actual secret value stored in Vault:

```shell
curl -i "http://127.0.0.1:9080/ip" -H "apikey: my-secret-api-key"
```

You should receive `HTTP/1.1 200 OK`. If Vault is unreachable or the path is wrong, APISIX returns `HTTP/1.1 401 Unauthorized`.

## How It Works

```
ApisixConsumer manifest
      │
      ▼
Ingress Controller  ──  passes $secret:// string to APISIX as-is
      │
      ▼
APISIX (etcd/apisix.yaml)  ──  stores: key = "$secret://vault/1/jack/key"
      │
      ▼
APISIX (on first request)  ──  fetches resolved value from Vault
```

Credentials are never stored in etcd or Kubernetes — only the reference URI is persisted.

## Troubleshooting

| Symptom | Likely cause |
|---------|-------------|
| `401 Unauthorized` even with correct credential | Vault URI unreachable from APISIX pod, or `prefix`/path mismatch |
| Auth fails immediately after consumer creation | Secret provider not registered (missing `secrets` entry or Admin API call) |
| `$secret://...` string appears in APISIX Admin API response | Expected — APISIX stores the reference, not the resolved value |
| `secret manager not exits` in APISIX error log | Invalid manager name in `id` field — must be `vault`, `aws`, or `gcp` |
