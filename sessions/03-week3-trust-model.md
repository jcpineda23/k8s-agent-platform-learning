# Session 03: The cluster's trust model

**Dates:** 2026-10-21 onward
**Plan position:** Phase 0, Week 3 · *Kubernetes the Hard Way* labs 04–06
**Status:** labs 04–06 done (2026-10-23). Open: prediction exercise and check-your-understanding answers.

[← Back to index](../index.md) · [Previous: Session 02](02-week2-containers-and-hard-way.md)

---

## The idea

Every connection in Kubernetes answers two questions, in order:

1. **Who are you?** (*authentication*) Components connect with a **client certificate**. Kubernetes has no user database: **the certificate is the identity.** Its Common Name (CN) becomes the **username**; its Organization (O) becomes the **group**.
2. **What may you do?** (*authorization*) The API server checks permissions for that username and group, using RBAC and, for kubelets, the Node Authorizer.

Plus a quieter third check: **"am I talking to the real server?"** The client checks that the server's certificate is signed by the CA it trusts and lists the name it dialed.

It all rests on **one CA**. Whoever holds `ca.key` can mint any identity, including cluster admin.

## The trust map

| Identity | CN (username) | O (group) | Talks to | Lives on |
|---|---|---|---|---|
| Cluster CA | `CA` | | signs everything | `ca.crt` everywhere; `ca.key` on jumpbox and server |
| API server | `kubernetes` | | serves everyone; calls kubelets | server |
| Scheduler | `system:kube-scheduler` | `system:system:kube-scheduler` (lab typo) | API server | server |
| Controller manager | `system:kube-controller-manager` | `system:kube-controller-manager` | API server | server |
| Admin | `admin` | `system:masters` | API server | server (`admin.kubeconfig`) |
| kubelet | `system:node:node-0` / `node-1` | `system:nodes` | API server, both directions | each node |
| kube-proxy | `system:kube-proxy` | `system:node-proxier` | API server | each node |
| Pods | `system:serviceaccount:<ns>:<name>` | | API server | a token signed with the `service-accounts` key, not a certificate |

Things to remember:
- **`system:masters` is hard-wired superuser.** It bypasses RBAC entirely, and a certificate can't be revoked before it expires (10 years here). Never give people certificates in this group; give them RBAC-bound identities.
- **Permissions are granted to the username (CN).** That's why the lab's typo in the scheduler's group (`system:system:kube-scheduler`) is harmless.
- **Pods prove identity with service account tokens, not certificates.** That's workload identity, the basis for how agent pods prove who they are (Project 3).

## Lab 04: Certificate authority (done 2026-10-21)

All commands run on the jumpbox in `/root/kubernetes-the-hard-way`, from the lab's `ca.conf`.

### The CA

```
openssl genrsa -out ca.key 4096
openssl req -x509 -new -sha512 -noenc -key ca.key -days 3653 -config ca.conf -out ca.crt
```

Result: subject and issuer are both `CN = CA` (**self-signed**), `CA:TRUE`, key usage "Certificate Sign, CRL Sign", valid **2026-10-22 to 2036-10-22**.
- Nothing vouches for the CA. Everything trusts it because `ca.crt` is copied there.
- When the CA expires, the whole cluster stops trusting itself. CA rotation is one of the hardest Kubernetes operations, and one that managed services do for you.

### The 8 component certificates

For each identity: a 4096-bit key, a signing request from its section of `ca.conf`, then the CA signs it.

```
admin.crt                   CN=admin,O=system:masters
node-0.crt                  CN=system:node:node-0,O=system:nodes
node-1.crt                  CN=system:node:node-1,O=system:nodes
kube-proxy.crt              CN=system:kube-proxy,O=system:node-proxier
kube-scheduler.crt          CN=system:kube-scheduler,O=system:system:kube-scheduler
kube-controller-manager.crt CN=system:kube-controller-manager,O=system:kube-controller-manager
kube-api-server.crt         CN=kubernetes
service-accounts.crt        CN=service-accounts
```

All 8 pass `openssl verify -CAfile ca.crt`.

### Which names the API server answers to

Its Subject Alternative Names: `127.0.0.1`, `10.32.0.1`, `kubernetes`, `kubernetes.default`, `kubernetes.default.svc`, `kubernetes.default.svc.cluster`, `kubernetes.svc.cluster.local`, `server.kubernetes.local`, `api-server.kubernetes.local`.

Tested with `openssl verify -verify_hostname` / `-verify_ip`:

| Client dials | Result |
|---|---|
| `server.kubernetes.local` | OK |
| `kubernetes.default.svc` | OK |
| `127.0.0.1` | OK |
| `node-0` | **verification failed** |
| `192.168.104.3` (the server's real IP) | **verification failed** |

**This is Session 02's question 5, proven live:** same certificate, same CA, same machine, but dialing the IP fails because it isn't in the certificate. TLS checks that the name you dialed is listed. That's why the kubeconfigs point at `server.kubernetes.local`.

Who uses each name:
- `127.0.0.1`: the admin kubeconfig, on the server itself
- `10.32.0.1` and `kubernetes.default.svc`: **pods inside the cluster**, via the built-in `kubernetes` Service. Your agent pods use these.
- **Lab slip:** `kubernetes.svc.cluster.local` was probably meant to be `kubernetes.default.svc.cluster.local`, the Service's fully qualified name, which isn't listed. Harmless in practice (clients use `kubernetes.default.svc`), but a pod dialing the full name would be rejected.

### node-0's certificate

`CN=system:node:node-0, O=system:nodes`, `CA:FALSE`, extended key usage **client and server authentication**, SAN `DNS:node-0, IP:127.0.0.1`.
- **One certificate, both directions:** the kubelet presents it when calling the API server, and when the API server calls into the kubelet for `kubectl logs` and `exec`.
- **The only name is the short hostname `node-0`,** so the API server must reach the node by that name. It works because of the shared hosts file from Session 02.
- **`CA:FALSE`:** a compromised node can impersonate only itself, and the Node Authorizer limits that to its own pods. That's why each node gets its own identity.

### Distribution

| Machine | Received |
|---|---|
| node-0 | `/var/lib/kubelet/ca.crt`, `kubelet.crt` (its own `node-0` identity), `kubelet.key` |
| node-1 | same, with the `node-1` identity |
| server | `~/ca.crt`, **`~/ca.key`**, `kube-api-server.crt/.key`, `service-accounts.crt/.key` |
| *(not yet)* | admin, kube-proxy, kube-scheduler, kube-controller-manager: wrapped into kubeconfigs in Lab 05 |

- **Each node gets only its own identity.** node-0 has nothing about node-1.
- **The server holds `ca.key`.** The controller manager uses it to sign rotated kubelet certificates. So the server is where all cluster trust lives: whoever steals that file can mint `system:masters`. In EKS and GKE, this key is exactly what you never see.

## Lab 05: kubeconfig files (done 2026-10-23)

A **kubeconfig** bundles three things a client needs:
1. **Where** the API server is (`server`)
2. **How to recognize it:** the CA certificate
3. **Who I am:** the client's certificate and key

Each is built with four `kubectl config` commands: `set-cluster` (1 and 2), `set-credentials` (3), then `set-context` and `use-context`. `--embed-certs=true` copies the certificates and key into the file.

| File | Server | User |
|---|---|---|
| `node-0.kubeconfig` | `https://server.kubernetes.local:6443` | `system:node:node-0` |
| `node-1.kubeconfig` | `https://server.kubernetes.local:6443` | `system:node:node-1` |
| `kube-proxy.kubeconfig` | `https://server.kubernetes.local:6443` | `system:kube-proxy` |
| `kube-controller-manager.kubeconfig` | `https://server.kubernetes.local:6443` | `system:kube-controller-manager` |
| `kube-scheduler.kubeconfig` | `https://server.kubernetes.local:6443` | `system:kube-scheduler` |
| `admin.kubeconfig` | `https://127.0.0.1:6443` | `admin` |

- **Five point at `server.kubernetes.local`,** a name in the API server's certificate. **`admin` points at `127.0.0.1`,** so it only works on the server itself (prediction 3).
- **The user name in a kubeconfig is only a label.** The API server reads the CN from the embedded certificate. Inside `kube-scheduler.kubeconfig` it's `CN=system:kube-scheduler`; rename the label and you're still the scheduler.
- **A kubeconfig is a credential.** The private key is embedded (`client-key-data`), so anyone who copies `admin.kubeconfig` is cluster admin for 10 years. kubectl wrote every file as `-rw-------`.
- **Production does it differently.** EKS and GKE kubeconfigs embed no key; they run a command (`aws eks get-token`, the GKE auth plugin) that fetches a short-lived token from your cloud login. Aim for the same with agents: short-lived, automatically issued credentials, never long-lived keys in files.

**Distribution:**

| Machine | Received |
|---|---|
| node-0, node-1 | `/var/lib/kubelet/kubeconfig` (its own node identity), `/var/lib/kube-proxy/kubeconfig` (shared) |
| server | `~/admin.kubeconfig`, `~/kube-controller-manager.kubeconfig`, `~/kube-scheduler.kubeconfig` |

**The asymmetry:** each kubelet has its own identity, but every node shares **one** kube-proxy identity. That's acceptable because kube-proxy only reads Services and endpoints. The kubelet writes pod and node status, so it gets per-node identity and the Node Authorizer. **Rule of thumb: the more an identity can change, the narrower it should be.** The same rule applies to agent credentials.

## Lab 06: Encrypting Secrets at rest (done 2026-10-23)

Without this, a Secret sits in etcd as plain base64. Anyone who reads etcd's disk or a backup gets every password and API key in the cluster.

The template, `configs/encryption-config.yaml`:

```yaml
kind: EncryptionConfiguration
apiVersion: apiserver.config.k8s.io/v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
```

Generated with `ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)` and `envsubst`, then copied to `server:~/encryption-config.yaml` (`-rw-------`).

- **Only Secrets are encrypted.** ConfigMaps and everything else stay readable in etcd, so credentials put in a ConfigMap aren't protected. Watch for this in reviews.
- **Provider order matters.** The first provider (`aescbc`, `key1`) encrypts new writes; all providers are tried on reads. `identity: {}` (plain text) is second so Secrets written before encryption was enabled can still be read. Key rotation works the same way: add `key2` on top, keep `key1`, rewrite every Secret.
- **The limit:** the key sits on the same server as etcd. It protects a stolen disk or backup, not a compromised server. Production uses the **KMS provider**, where the cloud's key service holds the key and it never touches the server's disk (EKS and GKE "envelope encryption"). Current Kubernetes docs steer away from `aescbc` toward KMS v2.
- **To verify in Session 04:** create a Secret and read its raw bytes from etcd with `etcdctl`. They should start with `k8s:enc:aescbc:v1:key1`.

## Prediction exercise *(to test in Session 04)*

Predict what happens, and why, when:
1. A client connects to the API server at `https://192.168.104.3:6443` instead of `server.kubernetes.local`.
2. node-0's kubelet starts with node-1's certificate.
3. Someone uses `admin.kubeconfig` from a machine other than the server.
4. The CA certificate expires.

## Check your understanding

1. The API server knows nothing about users. How does it decide who `admin` is, and what would someone need to impersonate `admin`?
2. Why does the server need `ca.key`, and what's the blast radius if the server is compromised compared with node-0?
3. A teammate proposes giving each engineer a client certificate in group `system:masters` "to keep it simple". What do you say, in two sentences?
4. **Agent platform:** an agent pod needs to call the Kubernetes API to create sandbox pods. Which identity should it use, and how do you keep it from doing anything else?

### My answers

_(Write your answers here.)_

---

**Next:** Lab 05 (kubeconfigs) and Lab 06 (encryption config), then the Session 04 control-plane labs.
