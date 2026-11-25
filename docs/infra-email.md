# Litrivo – infrastructure: web, e-mail, Kubernetes

Summary of a consultation on claude.ai (2026-11-25). Status: **planning**, nothing is deployed.
Legend: **[decided]** = the author's decision · **[recommended]** = proposal awaiting confirmation · **[open]** = unresolved.

## Domains and hostnames

Canonical domain `litrivo.com`; `.cz/.sk/.eu` → 301 to `.com` (1:1 subdomains). DNS in a single Cloudflare zone.

| Hostname | Purpose | Cloudflare proxy |
|---|---|---|
| `litrivo.com` | presentation website (`www` → 301 to the apex) | yes |
| `app.litrivo.com` | web version of the app (React / Next.js) | yes |
| `api.litrivo.com` | REST API for the native app (React Native, Android, later iOS) | yes |
| `mx.litrivo.com` | incoming mail (port 25) | **no – DNS only** |
| `smtp.litrivo.com` | sending from clients (465, 587) | **no – DNS only** |
| `imap.litrivo.com` | IMAP (993) | **no – DNS only** |
| `autoconfig.` / `autodiscover.litrivo.com` | automatic client configuration | yes (HTTPS) |
| `mail.litrivo.com` | webmail – optional, Outlook/Thunderbird preferred | yes |

Rancher will **not** be in the Litrivo zone → `kb.czghost.cz` [recommended], handled in a separate thread. Decide Rancher's `server-url` before installation (changing it later means re-registering the agents).

## Mail server

- **docker-mailserver** (Postfix + Dovecot + Rspamd) via the official Helm chart [decided]
  - Postfix: MX (25) and submission (465/587), Dovecot: IMAP (993), Rspamd: antispam + DKIM
  - No webmail and no web administration – accounts/aliases via the `setup` CLI or config files (versioned in Git)
- **Antispam:** Rspamd Bayes + neural module, trained on own samples; **GPT plugin with a local Ollama** [decided]
  - `type = "ollama"`, set `url` manually (verify the endpoint at deployment), raise `timeout` (~30 s on CPU)
  - `autolearn = false` at first, enable only after the verdicts have been verified
  - Ollama as a separate Deployment, Redis/Valkey with a PVC for Rspamd
  - Rule against "Google Play Console" phishing: display name vs. DMARC alignment (multimap/Lua) [open]
- **Migration from Cloudflare Email Routing** [recommended]: `support@litrivo.com` (the verified Google Play contact) currently runs on Cloudflare Email Routing, which stops working once the MX records point to the own server. Create the mailbox/alias in docker-mailserver and verify that it receives mail first, then switch MX and disable Email Routing, so the support address never goes dark
- **Secondary domain aliases** [decided]: `@litrivo.cz/.sk/.eu → @litrivo.com` in `postfix-virtual.cf`, MX for all domains; verify that non-existent addresses are rejected at `RCPT TO` (no backscatter)

## DNS – mail

```
_spf.litrivo.com  TXT "v=spf1 ip4:<outgoing IP> -all"
litrivo.com       TXT "v=spf1 include:_spf.litrivo.com -all"
litrivo.com       MX  10 mx.litrivo.com      (MX → A/AAAA, never CNAME)
```
- **Do not use `a`** in SPF (the apex is behind the Cloudflare proxy → Cloudflare IPs); limit of 10 DNS lookups
- DKIM + DMARC for `litrivo.com`; secondary domains that do not send: `v=spf1 -all`, DMARC `p=reject`
- **PTR** of the outgoing IP = `mx.litrivo.com` = HELO (set by the IP provider)

## Automatic client configuration [recommended]

- Thunderbird/KMail/Evolution: `https://autoconfig.litrivo.com/mail/config-v1.1.xml` (+ `/.well-known/autoconfig/…`), `clientConfig` 1.1 format
- Outlook (classic): POST `https://autodiscover.litrivo.com/autodiscover/autodiscover.xml` (schema `…/outlook/responseschema/2006a`) – needs a small script, not a static file
- Apple: `.mobileconfig` for download from the website
- SRV: `_imaps._tcp`, `_submissions._tcp`, `_submission._tcp`, `_autodiscover._tcp`
- Served by a small nginx in the cluster; plus a page with manual setup instructions

## Kubernetes – mail high availability

- docker-mailserver **cannot run multiple replicas** → `replicas: 1`, `Recreate` strategy [recommended]
- **Longhorn** – volume replication to 2–3 nodes; backups outside the cluster (replication ≠ backup)
- Shorter failover: `tolerationSeconds` for `node.kubernetes.io/unreachable` / `not-ready` → 30–60 s
- Longhorn setting `node-down-pod-deletion-policy` = `delete-both-statefulset-and-deployment-pod` [recommended]: without it the pod on a dead node stays `Terminating`, the RWO volume stays attached to that node and the replacement pod never starts, so `tolerationSeconds` alone does not complete the failover
- **MetalLB (L2)** or kube-vip – floating VIP, `externalTrafficPolicy: Local` (preserves the sender IP for Rspamd)
- Logical split of MX vs. SMTP/IMAP: two LoadBalancer Services pointing to the same pod, different IPs
- Physically split architecture (edge MX + backend) – postponed; an edge MX would need a persistent queue anyway
- No backup MX at a third party (spammers abuse them)
- **Outgoing IP ≠ Service IP** (SNAT via the node) → fixed egress IP, or a relay (`RELAY_HOST` in docker-mailserver)

## Hosting [open]

Candidate: 3 VMs at a friend's (Proxmox on a dedicated OVH server).
- OVH Additional IPs are bound to a vMAC → MetalLB L2 directly into the OVH network does not work
- **Preferred option** (requires the friend):
  1. internal bridge `vmbr1`, e.g. `10.0.0.0/24`, for the 3 VMs
  2. DNAT 25/465/587/993 from the public IP → VIP `10.0.0.10`
  3. SNAT outgoing traffic of the VMs to the same public IP (→ fixed egress IP, no relay needed)
  4. PTR of the public IP → `mx.litrivo.com` (OVH panel)
- **Fallback without the friend** (only if the VMs have their own public IPs): HAProxy DaemonSet with hostPort on every node + PROXY protocol into docker-mailserver, 3× A/MX record, outgoing via a relay
- Risks: one physical server = SPOF (acknowledge in the thesis), reputation of OVH IPs (check blacklists, UCEPROTECT), dependence on the friend
- Rancher: access only from an IP allowlist (+ node IPs for the agents), MFA via OIDC/GitHub, restrict the Kubernetes API (6443) as well
