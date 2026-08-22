---
{"dg-publish":true,"dg-path":"Resources/Split-horizon DNS breaks Let's Encrypt renewal in Cosmos Cloud.md","permalink":"/resources/split-horizon-dns-breaks-let-s-encrypt-renewal-in-cosmos-cloud/","title":"Split-horizon DNS breaks Let's Encrypt renewal in Cosmos (and how to fix it)","created":"2026-08-18","updated":"2026-08-22","dg-note-properties":{"type":["[[Notes/Article]]"],"topics":["[[Notes/Homelab]]"],"title":"Split-horizon DNS breaks Let's Encrypt renewal in Cosmos (and how to fix it)","created":"2026-08-18","modified":"2026-08-22"}}
---


**Topics:** ==[[Notes/Homelab\|Homelab]]==

> [!summary]
> `kosto.top` is publicly delegated to **Cloudflare**, but **Technitium** also runs a *primary* zone for it internally (split-horizon). That internal authority shadowed the ACME challenge lookup, so Cosmos/lego kept checking the wrong DNS server during cert renewal. **Fix:** make the OMV host resolve *public* names through public DNS (systemd-resolved upstreams), so lego sees Cloudflare (the real public authority) instead of internal Technitium. Split-horizon and ad-blocking for clients stay fully intact.

## The setup (mental model)

Two views of the same domain, decided by *who you ask*:

- **Public / outside world / Let's Encrypt** → `kosto.top` is delegated to Cloudflare (`kate.ns.cloudflare.com`, `augustus.ns.cloudflare.com`). Names resolve to Cloudflare anycast IPs and traffic comes in via the Cloudflare tunnel.
- **Inside the LAN** → clients query Technitium (handed out by DHCP). Technitium holds a **Primary zone** for `kosto.top`:
  - `@  A  192.168.0.12`
  - `*  A  192.168.0.12`
  - So any `something.kosto.top` resolves straight to the NAS's LAN IP: lower latency, no round-trip through Cloudflare, and it survives an internet/tunnel outage.

This is the intended, useful part. It's also the source of the problem below.

## What broke

Cosmos issues wildcard certs via **DNS-01** using the **Cloudflare** provider. The write half works perfectly: the `_acme-challenge.kosto.top` TXT record lands on Cloudflare. The failure is in **lego's propagation self-check** (lego = the ACME library Cosmos wraps).

The chain:

1. lego writes the TXT record to **Cloudflare** ✅
2. lego asks the host's resolver *"who is authoritative for `kosto.top`?"*
3. The host resolver forwards to **Technitium**, which, being a primary zone for `kosto.top` internally, answers *"me, `dns.kosto.top` → 192.168.0.12"* instead of returning Cloudflare's delegation.
4. lego queries **internal Technitium** for a TXT record that only exists on **Cloudflare** → never finds it → `time limit exceeded`.

> [!note] The record was always fine
> The TXT record was correctly on Cloudflare the whole time. Proof, run *during* a renewal attempt from any machine:
> ```
> dig _acme-challenge.kosto.top TXT @1.1.1.1 +short
> ```
> If two quoted values come back, the write works and the problem is purely lego resolving against the wrong server.

## Dead ends (don't repeat these)

- **Delegating `_acme-challenge` back to Cloudflare via NS records in Technitium.** Technitium returns a *referral* that lego doesn't chase, so the self-check just stalls on `authoritative nameservers: NS dns.kosto.top`. Remove these NS records if they were added.
- **Bind-mounting a private `resolv.conf` into just the Cosmos service** (`BindReadOnlyPaths=/etc/resolv.cosmos.conf:/etc/resolv.conf`). Doesn't hold, because `/etc/resolv.conf` is a symlink into `/run/systemd/resolve/`, and systemd-resolved actively rewrites those files. The Cosmos process kept reading resolved's "No DNS servers known" file.

## The fix that worked

Give **systemd-resolved itself** real upstream DNS on the OMV host. The host was running with *no global upstream* (`No DNS servers known`), leaning entirely on Technitium via per-link DHCP, which is exactly what needed to change.

Edit `/etc/systemd/resolved.conf` (or a drop-in under `/etc/systemd/resolved.conf.d/`):

```ini
[Resolve]
DNS=1.1.1.1 8.8.8.8
Domains=~.
DNSStubListener=no
```

Then:

```bash
systemctl restart systemd-resolved
resolvectl status | grep -A3 'Global'
```

Confirm the `Global` block now shows:

```
  resolv.conf mode: uplink
Current DNS Server: 1.1.1.1
```

And that `/etc/resolv.conf` lists the upstreams directly:

```bash
grep nameserver /etc/resolv.conf
# nameserver 1.1.1.1
# nameserver 8.8.8.8
```

Restart Cosmos so it picks up the new resolv.conf, then retry the cert:

```bash
systemctl restart CosmosCloud.service
```

Now lego resolves `kosto.top` via 1.1.1.1 → discovers **Cloudflare** as authoritative → finds the TXT it wrote there → self-check passes → Let's Encrypt validates against the same public view. Green. ✅

> [!warning] Config gotchas
> - **`Domains=~.` needs a tilde**, not a hyphen. `~.` = "route *all* lookups to these servers." `-.` is exclusion syntax and does nothing useful here. One character; easy to miss. **`DNS=` plus `Domains=~.` are the only two settings the cert fix actually needs.**
> - **Leave `DNSStubListener=no`.** It was already `no` on this box; don't "fix" it. With the stub off, `resolv.conf` runs in **uplink** mode and lists `1.1.1.1`/`8.8.8.8` directly, which is exactly what Cosmos reads. Setting it to `yes` also technically works (stub mode → `127.0.0.53` → same upstreams), but there's no reason to change it.
> - ⚠️ **Correction to the first version of this note:** it claimed the stub listener steals port 53 from Technitium. **That is wrong.** Technitium runs as a *bridged* container in its own network namespace, so its internal `:53` cannot conflict with the host's `127.0.0.53:53`. They coexist fine. The real port-53 outage had a completely different cause — see below.

## Second incident: Technitium container loses published port 53

Unrelated to the cert work, but it hit a few days later and *looked* like the same class of problem, so it's worth recording separately.

**Symptom:** every `*.kosto.top` name stops resolving on LAN clients. Browser shows `DNS_PROBE_POSSIBLE` / `ERR_FAILED`. `Resolve-DnsName` without `-Server` falls through to the **public** view and returns Cloudflare's SOA for `kosto.top` with no A record. Meanwhile `docker ps` shows the container **"Up"** and the web UI on `:5380` works fine, so it looks perfectly healthy.

**Cause:** the container had been recreated at some point *without its port mappings*. Technitium was listening on `:53` inside its own namespace, but nothing mapped that to `192.168.0.12:53`.

**Diagnose:**

```bash
dig immich.kosto.top @192.168.0.12 +short   # connection refused (NOT timeout)
ss -tulnp | grep -w ':53'                   # no listener at all
docker logs --tail 30 Dns-server            # hostname is a container ID → bridge networking
docker ps -a | grep -i technitium           # PORTS column empty → no published ports
```

**Fix:** repull the image and recreate the container with ports published. Verify afterwards:

```bash
docker inspect Dns-server --format '{{json .HostConfig.PortBindings}}'
```

Known-good bindings for this box:

```json
{"53/tcp":[{"HostPort":"53"}],
 "53/udp":[{"HostPort":"53"}],
 "5380/tcp":[{"HostPort":"5380"}],
 "8053/tcp":[{"HostPort":"8053"}]}
```

> [!tip] Read the error precisely
> - **`connection refused`** = reachable, but *nothing is listening* → port not published / service not bound.
> - **`timeout`** = nothing answered at all → host unreachable, firewall, or wrong IP.
> - **`DNS_PROBE_POSSIBLE` / falls through to public SOA** = the client never got an answer from Technitium and resolved publicly instead.
>
> These three point at genuinely different causes. Don't lump them together.

## Client-side gotcha: browser DoH bypasses Technitium

If a name resolves fine in PowerShell but the **browser** times out or can't find it, suspect **Secure DNS (DNS-over-HTTPS)** in the browser. Brave with "Use secure DNS" on (even set to *OS default*) can resolve via public DoH, skipping Technitium entirely — so it gets the **public** record instead of `192.168.0.12`, and internal-only services appear dead.

**Fix:** `brave://settings/security` → turn **Use secure DNS** off. On a LAN where you *want* split-horizon and ad-blocking, browser DoH works against you; its privacy benefit is about hiding lookups from your ISP, and your lookups already stop at your own Technitium box.

If you want encrypted DNS anyway, set the provider to **Custom** and point it at Technitium's own DoH endpoint (e.g. `https://dns.kosto.top/dns-query`, which the wildcard cert already covers) rather than "OS default".

**Always flush both caches after any DNS change**, since browsers cache *negative* results too:

```powershell
ipconfig /flushdns
# plus: brave://net-internals/#dns → Clear host cache
```

## The tradeoff (know where the lever is)

The OMV host itself now resolves `*.kosto.top` to **Cloudflare's public IP**, not `192.168.0.12`. That's fine because the services live on this box and are reachable locally by container name / IP, so the host doesn't need the internal view.

If some process *on OMV* ever needs the LAN answer, scope it instead of overriding globally: keep `DNS=1.1.1.1 8.8.8.8`, drop `Domains=~.`, and add a per-link route sending only `kosto.top` to Technitium.

**Clients are unaffected.** Phones/laptops still query Technitium via DHCP and keep the local `192.168.0.12` answers *and* ad-blocking. Only this one host changed. The two coexist by design: **the host bypasses Technitium (needed for ACME), while LAN clients use it (split-horizon + ad-blocking).**

## Verify Technitium still does its job

Run these **from a LAN client** (e.g. Windows PowerShell), *not* from OMV (OMV now resolves publicly, so it's no longer a valid test point).

```powershell
# Internal split-horizon answer: expect 192.168.0.12
Resolve-DnsName immich.kosto.top -Type A -Server 192.168.0.12 -DnsOnly

# Public answer: expect a Cloudflare IP (104.x / 172.6x.x)
Resolve-DnsName immich.kosto.top -Type A -Server 1.1.1.1 -DnsOnly

# Default lookup (uses DHCP-assigned resolver): expect 192.168.0.12
Resolve-DnsName immich.kosto.top -Type A -DnsOnly

# Ad-blocking: expect NXDOMAIN / 0.0.0.0, not a real IP
Resolve-DnsName doubleclick.net -Type A -Server 192.168.0.12 -DnsOnly
```

Healthy result:

| Query | Expected | Meaning |
|---|---|---|
| `-Server 192.168.0.12` | `192.168.0.12` | Internal view works |
| `-Server 1.1.1.1` | Cloudflare IP | Public view stays separate |
| default (no `-Server`) | `192.168.0.12` | Clients actually use Technitium |
| `doubleclick.net` | NXDOMAIN | Ad-blocking live |

Bonus tell: different **TTLs** from the two sources (e.g. `3600` from Technitium vs `300` from Cloudflare) confirm you're genuinely hitting two different authorities, not a cache crossover. `nslookup` showing `Server: UnKnown / Address: 192.168.0.12` is normal; it just means there's no PTR record for the resolver's own IP (cosmetic; add a reverse zone if it bothers you).

## Full health check (host side)

```bash
resolvectl status | grep -A3 'Global'                                   # DNS Servers: 1.1.1.1 8.8.8.8
grep nameserver /etc/resolv.conf                                        # 1.1.1.1 / 8.8.8.8 (uplink mode)
dig immich.kosto.top @192.168.0.12 +short                               # 192.168.0.12
docker inspect Dns-server --format '{{json .HostConfig.PortBindings}}'  # 53/tcp, 53/udp, 5380, 8053
```

All four green = cert renewal path healthy *and* split-horizon healthy.

## Persistence note

Cert **renewal depends on this host resolving publicly.** The `resolved.conf` change is persistent config, so renewals keep working, but if the OMV box is ever rebuilt or systemd-resolved is reset, the `Domains=~.` + upstream setting must be reapplied or renewal breaks again with the same split-horizon symptom.

Likewise, **LAN DNS depends on the Technitium container keeping its published ports.** A container recreate that drops `53:53` silently kills every internal name while still looking "Up".

## The one-line cause, for future me

> Split-horizon zone in Technitium → lego's ACME self-check followed the host resolver to internal Technitium → Technitium claimed authority over `kosto.top` and hid Cloudflare → fix by pointing the host's systemd-resolved at public upstreams so ACME sees the real Cloudflare delegation.

> And if internal names die later: check `ss -tulnp | grep -w ':53'` before blaming DNS config. *Connection refused* means the Technitium container lost its published ports, not that anything is misconfigured.

---

If this guide saved you some time, you can say thanks with a coffee:

<a href="https://ko-fi.com/dominoto"><img src="https://storage.ko-fi.com/cdn/kofi2.png?v=3" width="200" alt="Buy me a coffee at ko-fi.com"></a>