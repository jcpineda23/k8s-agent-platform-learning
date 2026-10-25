# Session 03: The cluster's trust model

**Dates:** 2026-10-21 onward
**Plan position:** Phase 0, Week 3 · *Kubernetes the Hard Way* labs 04–06
**Status:** labs 04–06 done (2026-10-23). *Kubernetes the Hard Way* stopped here by decision (see the end of this file). Questions optional.

[← Back to index](../index.md) · [Previous: Session 02](02-week2-containers-and-hard-way.md)

---

## Before you start: key terms

No prior knowledge assumed beyond Sessions 01–02.

### Keys and certificates

- **Public-key cryptography:** each party has a pair of keys. The **private key** is kept secret; the **public key** is shared freely. Something signed with the private key can be verified by anyone holding the public key, and only the private key's owner could have produced it.
- **Digital signature:** proof that a specific private key vouched for a piece of data, and that the data hasn't changed since.
- **Certificate (X.509 certificate):** a small document that says "this public key belongs to this name", signed by someone others trust. Like an ID card: a photo (the public key), a name, and a stamp from an authority.
- **Certificate authority (CA):** the trusted signer of certificates. If you trust the CA, you trust every certificate it signed. Its own certificate is **self-signed**: nothing vouches for it; you trust it because you installed it.
- **CA private key (`ca.key`):** the key the CA signs with. Whoever holds it can create any identity, so it's the most sensitive file in a cluster.
- **Certificate signing request (CSR):** a request that says "please sign a certificate for my public key, with these names". You keep your private key; only the request goes to the CA.
- **Common Name (CN) and Organization (O):** fields in a certificate's subject. Kubernetes reads the CN as the **username** and the O as the **group**.
- **Subject Alternative Names (SANs):** the list of DNS names and IP addresses a certificate is valid for. A client connecting to a server checks that the name it dialed is in this list.
- **Expiry:** every certificate has a "not after" date. After it, nobody accepts it, and if the CA itself expires, the whole cluster stops trusting itself.
- **Revocation:** cancelling a certificate before it expires. Kubernetes has no way to revoke client certificates, which is why long-lived admin certificates are dangerous.

### Secure connections

- **TLS (Transport Layer Security):** the protocol that encrypts network connections and proves the server's identity. It's the "S" in HTTPS.
- **Mutual TLS (mTLS):** TLS where *both* sides present certificates, so the server also learns who the client is. Every Kubernetes component connection uses it.
- **Trust on first use (TOFU):** SSH's approach: trust a server's key the first time you see it and complain if it ever changes. A CA replaces this with trust arranged in advance.

### Who you are and what you may do

- **Authentication (authn):** proving who you are. In this lab: a client certificate.
- **Authorization (authz):** deciding what you're allowed to do once identified.
- **RBAC (role-based access control):** Kubernetes' main authorization system. A **Role** lists allowed actions ("create pods in namespace X"); a **RoleBinding** grants a Role to a user or group. **ClusterRole** and **ClusterRoleBinding** do the same cluster-wide.
- **Node Authorizer:** a special authorizer that limits each kubelet to its own node's pods and the data they use.
- **system:masters:** a built-in group that bypasses all authorization. Anyone in it is a superuser.
- **Service account:** an identity for *software* running in pods, rather than for people. A pod proves it with a **service account token**, a signed **JWT** (JSON Web Token: a small signed document of claims).
- **Workload identity:** the general idea of software proving who it is, to the cluster or to outside services such as cloud APIs, without long-lived passwords.

### Connecting to a cluster

- **kubeconfig:** a file that tells a client which API server to connect to, how to recognize it (the CA certificate), and who you are (your certificate and key, or a way to get a token). Because it can contain a private key, a kubeconfig is a credential.
- **Context:** a named combination of cluster and user inside a kubeconfig.
- **Short-lived token:** a credential that expires within minutes or hours. Managed Kubernetes kubeconfigs fetch these from your cloud login instead of embedding keys.

### Protecting stored data

- **Secret:** a Kubernetes object for sensitive data like passwords and API keys. By default it's stored in etcd only **base64-encoded**, which is a reversible text format, not encryption.
- **Encryption at rest:** encrypting data where it's stored (etcd's disk and backups), so a stolen disk reveals nothing.
- **Encryption provider:** how the API server encrypts. `aescbc` uses a key from a local file; `identity` means no encryption.
- **KMS (key management service) and envelope encryption:** the production approach. A cloud key service holds the master key, which never touches the server's disk; it encrypts the data keys that encrypt your Secrets.
- **Key rotation:** replacing an encryption key or certificate with a new one, without losing access to what the old one protected.

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

## Prediction exercise *(untested; Hard Way stopped before the API server ran)*

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

## Decision: stop *Kubernetes the Hard Way* here (2026-10-23)

**Question raised:** isn't this too much detail for an engineering manager?

**Answer:** partly yes. Typing `openssl` and `kubectl config` commands and fighting Lima's quirks is engineer-level mechanics. What matters for a manager is the *consequences*, because each maps to a decision you own or sign off on:

| What the labs showed | The decision it informs |
|---|---|
| Containers share the host kernel | Whether agent-generated code may run in plain containers, or needs gVisor or microVMs |
| `system:masters` bypasses permissions and can't be revoked | Access policy: why "just give the team admin" is a 10-year, unrevokable risk |
| Whoever holds `ca.key` controls the cluster | Build vs. buy, and the threat model |
| A kubeconfig is a credential; production uses short-lived tokens | How agents authenticate to the cluster and to tools |
| Only Secrets are encrypted, and only against disk theft | What you can honestly tell security reviewers and auditors |
| Certificates bind names; dialing by IP fails | Recognizing "x509: certificate is valid for X, not Y" during incidents |

**Decision:** stop after Lab 06 and move to Phase 1. The identity model, the most important lesson of the remaining labs, is already learned. Skipped: starting etcd and the control plane, worker nodes, pod networking and the smoke test.

**New approach from here:** questions ask what you'd *decide, ask or block*, not how things work; depth is spent where the Agent Platform's risk is (sandboxing, identity, multi-tenancy, GPU scheduling), shallow elsewhere; Claude runs mechanical steps.

The four predictions above stay untested; the check-your-understanding questions are optional.


## Resources

### Books
- *Production Kubernetes* (Rosso, Lander, Brand, Harris). Chapter 7, "Secret Management", and Chapter 10, "Identity": encryption at rest, KMS, user and workload identity, from a platform team's point of view. The best match for this session.
- *Kubernetes: Up and Running*, 3rd ed. Chapters: "ConfigMaps and Secrets" and "Role-Based Access Control".
- *Hacking Kubernetes* (Martin, Hausenblas). How attackers abuse exactly these weak points: stolen kubeconfigs, over-broad permissions, unencrypted Secrets.

### Articles and docs
- [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/): the authentication → authorization → admission sequence on one page. Start here.
- [PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/): every certificate a cluster needs, and why.
- [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/): client certificates, tokens and other methods.
- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) and [Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/).
- [Organizing Cluster Access Using kubeconfig Files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/).
- [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/): workload identity inside the cluster.
- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) and [Using a KMS provider](https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/).
- [What is an SSL certificate?](https://www.cloudflare.com/learning/ssl/what-is-an-ssl-certificate/) (Cloudflare) and [How It Works](https://letsencrypt.org/how-it-works/) (Let's Encrypt): plain-language introductions to certificates and CAs.

### Videos
- [TLS Handshake Explained](https://www.youtube.com/watch?v=86cQJ0MMses) (Computerphile). How a client and server use certificates to set up a secure connection.

---

**Next:** [Phase 1, Session 04](../index.md#phase-1-plan): extending Kubernetes with your own APIs.
