# Litrivo – infrastructure: web, e-mail, Kubernetes

Summary of a consultation on claude.ai (2026-11-25, updated 2026-11-26). Status: **planning**, nothing is deployed.
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
| `notify.litrivo.com` | sending subdomain for automated app e-mails (SPF/DKIM/DMARC records only, no web) | n/a |

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
- **Incoming mail self-hosted, outgoing mail via a relay** [decided] – see *Outgoing mail* below

## Outgoing mail

Why a relay: the candidate hosting is OVH (AS16276). The candidate IP `51.89.16.5` is clean on its own (UCEPROTECT Level 1: not listed), but its `/16` is listed on UCEPROTECT Level 2 and the whole AS on Level 3 – a near-permanent state for OVH. UCEPROTECT L2/L3 carry little weight with the big mailbox providers, but OVH ranges are frequently blocked by **Microsoft** (Outlook.com/Hotmail/M365, error S3150), which cannot be checked in advance. Incoming mail is unaffected – blacklists judge the sender, not the recipient.

Two separate sending paths [recommended]:

| Path | Provider | Use | Notes |
|---|---|---|---|
| Personal mail (replies from `info@`, `support@` …) | **SMTP2GO** as Postfix smarthost | general-purpose relay | Free: 1,000/month, 200/day, 25/hour until the domain is verified, 5 days of history |
| Automated app e-mails | **Scaleway TEM** via REST API, directly from the backend (not through docker-mailserver) | transactional only | Free: 300/month, then €0.25 per 1,000, pay-as-you-go, billed monthly, EU-hosted, no daily cap |

- Scaleway TEM is **transactional only** (registration confirmation, password reset, new-device alerts, notifications); marketing is forbidden by its terms and it is not meant for person-to-person mail; allowed attachment types are restricted
- One provider for both is possible as a simplification; switching later only changes the relay settings in Postfix
- With a relay: no own PTR, no fixed egress IP and no IP reputation work needed for sending
- Docker-mailserver relay env variables: verify exact names for the deployed version

### Automated app e-mails [recommended]

- Sender on a subdomain, e.g. `noreply@notify.litrivo.com` → reputation of automated mail separated from personal mail
- Protect the registration form against abuse (attackers use sign-ups to send mail to third-party addresses):
  - CAPTCHA – Cloudflare Turnstile
  - rate limiting per IP and per e-mail address
  - short-lived confirmation links (e.g. 24 h), limited resends
- Process bounces via the provider's webhook; stop sending to addresses that bounced

## DNS – mail

```
_spf.litrivo.com  TXT "v=spf1 include:<SMTP2GO SPF> -all"
litrivo.com       TXT "v=spf1 include:_spf.litrivo.com -all"
litrivo.com       MX  10 mx.litrivo.com      (MX → A/AAAA, never CNAME)
notify.litrivo.com  SPF / DKIM / DMARC as required by Scaleway TEM
```
- **Do not use `a`** in SPF: the mechanism resolves the domain's A record, and with the apex behind the Cloudflare proxy that yields Cloudflare IPs (the TXT record itself is not proxied – the problem is the lookup). Limit of 10 DNS lookups across the whole include tree
- DKIM: the relay provider's DKIM record (usually a CNAME) after verifying the domain; DMARC passes via the provider's DKIM signature aligned to `litrivo.com`
- DMARC for `litrivo.com`; secondary domains that do not send: `v=spf1 -all`, DMARC `p=reject`
- DMARC reports to Cloudflare DMARC Management (same setup as `czghost.cz`, which already runs `p=reject` with 100 % pass)
- PTR of an own outgoing IP is **not needed** while all outgoing mail goes through relays

### Checking IP reputation

- Google Postmaster Tools **no longer shows IP or domain reputation** (removed in v2) and never allowed checking arbitrary ranges – only data for an own verified domain with enough Gmail volume
- Before committing to an IP: MXToolbox Blacklist Check, Spamhaus (check.spamhaus.org – the one that matters most), Cisco Talos, UCEPROTECT (L2/L3 = neighbours, low weight)
- Microsoft SNDS requires authorisation by the IP owner (WHOIS contact) → only via the friend / OVH
- After deployment: test message to mail-tester.com
- Paid UCEPROTECT "express delisting" / whitelisted.org: not worth it

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
- Outgoing IP ≠ Service IP (SNAT via the node) – **resolved by the relay**, egress IP no longer matters for sending

## Hosting [open]

Candidate: 3 VMs at a friend's (Proxmox on a dedicated OVH server). The friend has OVH public IP ranges and private networks in Proxmox already set up.
- OVH Additional IPs are bound to a vMAC → MetalLB L2 directly into the OVH network does not work
- **To clarify with the friend:**
  1. public IPs bridged with a vMAC per VM, or routed through the host? (routed → MetalLB can announce a public IP inside the private network, no NAT; bridged → DNAT to a private VIP)
  2. one shared L2 private network (bridge or Proxmox SDN VNet; VXLAN overlay is fine) for the 3 VMs
  3. a public IP for incoming mail (check blacklists first)
  4. port 25 inbound not blocked
- **Preferred option** (requires the friend):
  1. internal bridge `vmbr1`, e.g. `10.0.0.0/24`, for the 3 VMs
  2. DNAT 25/465/587/993 from the public IP → VIP `10.0.0.10`
  3. SNAT / PTR for outgoing mail – **not needed** with relays
- **Fallback without the friend** (only if the VMs have their own public IPs): HAProxy DaemonSet with hostPort on every node + PROXY protocol into docker-mailserver, 3× A/MX record
- **Alternative suggested by another friend ("SMTP proxy")** [open]: a small VPS with a clean IP as mail gateway, tunnelled (WireGuard) to the cluster – only relevant if the OVH setup does not work out
- Risks: one physical server = SPOF (acknowledge in the thesis), OVH IP reputation (handled by the relay for outgoing mail), dependence on the friend
- Rancher: access only from an IP allowlist (+ node IPs for the agents), MFA via OIDC/GitHub, restrict the Kubernetes API (6443) as well
- Claude Code on a separate dev VPS (not on the cluster nodes); no admin kubeconfig for production there – deploy via CI/CD or GitOps (Argo CD / Flux), read-only namespaced kubeconfig at most
