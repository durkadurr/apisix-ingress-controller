---
title: Reference Vault Secrets in ApisixConsumer
keywords:
  - APISIX ingress
  - Apache APISIX
  - Kubernetes ingress
  - HashiCorp Vault
  - secret management
description: Learn how to configure ApisixConsumer to reference authentication credentials stored in HashiCorp Vault instead of storing them inline or in Kubernetes Secrets.
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

APISIX has a built-in secret manager that can fetch credentials from external providers such as HashiCorp Vault at runtime. You can use the `$secret://` URI scheme inside `ApisixConsumer` value fields to reference secrets stored in Vault instead of embedding them in your manifests or Kubernetes Secrets.

The APISIX Ingress Controller passes these reference strings through to APISIX as-is — no resolution happens in the controller. APISIX resolves the value from Vault when the consumer is first used.

## Prerequisites

1. Complete [Get APISIX and APISIX Ingress Controller](../getting-started/get-apisix-ingress-controller.md).
2. HashiCorp Vault is running and accessible from your APISIX data plane.
3. APISIX is configured with a Vault secret provider (see [Configure Vault in APISIX](#configure-vault-in-apisix)).

## Configure Vault in APISIX

Add a `secret_providers` entry to your APISIX `config.yaml`:

```yaml
secret_providers:
  - id: 1
    name: vault
    prefix: /apisix/kv
    token: <your-vault-token>
    uri: http://vault.default.svc.cluster.local:8200
```

| Field    | Description                                                              |
|----------|--------------------------------------------------------------------------|
| `id`     | Numeric identifier referenced in the `$secret://` URI.                   |
| `prefix` | KV path prefix in Vault under which secrets are stored.                  |
| `token`  | Vault authentication token.                                              |
| `uri`    | Address of the Vault server reachable from the APISIX pod.               |

## Secret URI Format

```
$secret://vault/<id>/<path>/<key>
```

| Segment  | Description                                                              |
|----------|--------------------------------------------------------------------------|
| `vault`  | Secret provider type.                                                    |
| `<id>`   | The `id` value from your `secret_providers` config.                      |
| `<path>` | Sub-path appended to `prefix`. Maps to `{prefix}/{path}` in Vault.       |
| `<key>`  | The key name within that Vault KV entry.                                 |

For example, with `prefix: /apisix/kv`, the reference `$secret://vault/1/jack/key` resolves to the `key` field at Vault path `/apisix/kv/jack`.

## Store Credentials in Vault

Write the consumer credentials to Vault before creating the `ApisixConsumer`:

```shell
# key-auth
vault kv put /apisix/kv/jack key="my-secret-api-key"

# jwt-auth
vault kv put /apisix/kv/jack \
  jwt-key="jack" \
  jwt-secret="my-jwt-secret"
```

## Create an ApisixConsumer with Vault References

### Key Authentication

```yaml title="consumer-vault.yaml"
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

```yaml title="consumer-vault.yaml"
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
        secret: $secret://vault/1/jack/jwt-secret
```

For asymmetric algorithms (RS256, ES256), reference the key pair:

```yaml
    jwtAuth:
      value:
        key: jack
        algorithm: RS256
        public_key: $secret://vault/1/jack/public-key
        private_key: $secret://vault/1/jack/private-key
```

### Basic Authentication

```yaml title="consumer-vault.yaml"
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

```yaml title="consumer-vault.yaml"
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
        key_id: jack
        secret_key: $secret://vault/1/jack/hmac-secret
```

## Apply and Verify

Apply the consumer manifest:

```shell
kubectl apply -f consumer-vault.yaml
```

Confirm the consumer was created:

```shell
kubectl get apisixconsumer -n ingress-apisix jack
```

To verify Vault resolution is working, enable key authentication on a route (see [Key Authentication](../getting-started/key-authentication.md)) and send a request using the actual secret value stored in Vault:

```shell
curl -i "http://127.0.0.1:9080/ip" -H "apikey: my-secret-api-key"
```

You should receive an `HTTP/1.1 200 OK` response. If Vault is unreachable or the path is incorrect, APISIX returns `HTTP/1.1 401 Unauthorized`.

## How It Works

The APISIX Ingress Controller treats `value` fields as opaque strings and syncs them to APISIX without modification. APISIX detects the `$secret://` prefix and fetches the actual value from the configured secret provider at runtime. Credentials are never stored in etcd or Kubernetes.

```
ApisixConsumer manifest
      │
      ▼
Ingress Controller (passes $secret:// string as-is)
      │
      ▼
APISIX etcd  ──── stores: key = "$secret://vault/1/jack/key"
      │
      ▼
APISIX (on first use) ──── fetches value from Vault
```

## Troubleshooting

| Symptom | Likely cause |
|---------|-------------|
| `401 Unauthorized` even with correct key | Vault URI is unreachable from APISIX pod, or `prefix`/path is wrong |
| Consumer created but auth fails immediately | `secret_providers` not configured in APISIX `config.yaml` |
| `$secret://...` string appears literally in APISIX Admin API response | Expected — APISIX stores the reference, not the resolved value |
