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
DNSStubListener=yes
```

Then:

```bash
systemctl restart systemd-resolved
resolvectl status | grep -A3 'Global'
```

Confirm the `Global` block now shows:

```
  resolv.conf mode: stub
       DNS Servers: 1.1.1.1 8.8.8.8
```

Restart Cosmos so it picks up the new resolv.conf, then retry the cert:

```bash
systemctl restart CosmosCloud.service
```

Now lego resolves `kosto.top` through the stub (`127.0.0.53`) → 1.1.1.1 → discovers **Cloudflare** as authoritative → finds the TXT it wrote there → self-check passes → Let's Encrypt validates against the same public view. Green. ✅

> [!warning] Two config gotchas that bit me
> - **`Domains=~.` needs a tilde**, not a hyphen. `~.` = "route *all* lookups to these servers." `-.` is exclusion syntax and does nothing useful here. One character; easy to miss.
> - **`DNSStubListener` must be `yes`.** With it set to `no`, nothing listens on `127.0.0.53`, which is exactly the address the resolv.conf symlink (and therefore Cosmos) points at. Turning the stub on is what flips `resolv.conf mode` from `uplink` to `stub`.

## The tradeoff (know where the lever is)

The OMV host itself now resolves `*.kosto.top` to **Cloudflare's public IP**, not `192.168.0.12`. That's fine because the services live on this box and are reachable locally by container name / IP, so the host doesn't need the internal view.

If some process *on OMV* ever needs the LAN answer, scope it instead of overriding globally: keep `DNS=1.1.1.1 8.8.8.8`, drop `Domains=~.`, and add a per-link route sending only `kosto.top` to Technitium.

**Clients are unaffected.** Phones/laptops still query Technitium via DHCP and keep the local `192.168.0.12` answers *and* ad-blocking. Only this one host changed.

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

## Persistence note

Cert **renewal depends on this host resolving publicly.** The `resolved.conf` change is persistent config, so renewals keep working, but if the OMV box is ever rebuilt or systemd-resolved is reset, the `Domains=~.` + upstream setting must be reapplied or renewal breaks again with the same split-horizon symptom.

## The one-line cause, for future me

> Split-horizon zone in Technitium → lego's ACME self-check followed the host resolver to internal Technitium → Technitium claimed authority over `kosto.top` and hid Cloudflare → fix by pointing the host's systemd-resolved at public upstreams so ACME sees the real Cloudflare delegation.

---

If this guide saved you some time, you can say thanks with a coffee:

<a href="https://ko-fi.com/dominoto"><img src="https://storage.ko-fi.com/cdn/kofi2.png?v=3" width="200" alt="Buy me a coffee at ko-fi.com"></a>