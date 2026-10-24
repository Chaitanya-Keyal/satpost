# Security

Nothing public can spend. The internet only reaches two LNbits endpoints, and LNbits only has
phoenixd's receive-only password. The password that can spend never leaves phoenixd's container;
you use it through `satpost` over SSH.

## What can reach what

| Exposure | What | If it's compromised |
|---|---|---|
| Internet (Funnel) | `/.well-known/lnurlp/` and `/lnurlp/api/v1/lnurl/cb/` on port 443, answered by LNbits | invoice spam, your payment history readable, and new payments redirected to someone else's invoices until you notice. No spending |
| Tailnet, by policy | LNbits admin on 8443 (its login plus two-factor); phoenixd's API on 9740, for a dashboard you allow | needs one of your devices plus the app's credentials |
| Host only | phoenixd's full-access password and the seed, in phoenixd's volume | needs root on the host, or a user in the docker group |

Nothing is published on the host. phoenixd and LNbits bind `127.0.0.1` inside the shared
namespace, so other users of the machine can't connect to them through the container's bridge
address (both images bind `0.0.0.0` by default; compose.yml overrides it).

## The receive-only password

LNbits gets phoenixd's limited-access password. Checked against phoenixd 0.9.3, it is refused
with HTTP 401 by every call that moves money out: `payinvoice`, `payoffer`, `paylnaddress`,
`lnurlpay`, `lnurlauth`, `sendtoaddress`, `closechannel` and `bumpfee`. It still works for
everything LNbits needs to receive: `getinfo`, `getbalance`, `createinvoice`,
`payments/incoming` and the websocket. (`export` is also allowed, contrary to phoenixd's API
docs; it only writes a CSV inside the container.)

Never give LNbits `PHOENIXD_DATA_DIR`: with it, LNbits reads the seed from phoenixd's data
directory.

## Your tailnet policy is required

The Tailscale container runs in userspace mode, which forwards tailnet connections for *any*
port that serve doesn't handle to that port in the namespace, such as LNbits' plain HTTP port
5000. Binding loopback keeps the services off the host, not off the tailnet. Only your policy decides which
devices reach which ports, so no rule may reach `tag:pay` except the ones below. A policy that
allows everything to everything has to narrow first, for example to `autogroup:self`, which
covers your own devices but not tagged ones.

```jsonc
{
  "tagOwners": { "tag:pay": ["autogroup:admin"] },
  // Optional: a device allowed to read phoenixd's API, such as a satbook dashboard. If that
  // device is tagged, use its tag as the source below instead of a host entry.
  "hosts": { "ledger": "100.64.0.10" },
  "nodeAttrs": [{ "target": ["tag:pay"], "attr": ["funnel"] }],
  "grants": [
    // Your own devices keep reaching each other. Tagged devices aren't in autogroup:self.
    { "src": ["autogroup:member"], "dst": ["autogroup:self"], "ip": ["*"] },
    // The LNbits admin page.
    { "src": ["autogroup:member"], "dst": ["tag:pay"], "ip": ["tcp:8443"] },
    // Optional: the dashboard.
    { "src": ["ledger"], "dst": ["tag:pay"], "ip": ["tcp:9740", "tcp:8443"] }
  ]
}
```

Adapt this to your existing policy. Narrow the admin rule to your phone and laptop if other
people or automated accounts use your devices.

## Where the secrets live

| Secret | Where | Backup |
|---|---|---|
| phoenixd seed | phoenixd volume, `seed.dat` | 12 words on paper, offline. Never in cloud backups |
| phoenixd full-access password | phoenixd volume, `phoenix.conf` | none needed; regenerated if lost |
| phoenixd receive-only password | phoenixd volume, `.env`, LNbits' database | none needed |
| LNbits first-install token, auth secret | `.env` | none needed |
| LNbits admin login and two-factor | your password manager | yes |
| Zap signing key | LNbits' database | optional; a new one only needs the address file refreshed |
| Tailscale auth key | `.env`; single-use, so spent after the first login | none needed |

`satpost` never puts a secret on a command line, where every user on the host could read
it; values go to `.env` through shell builtins only. LNbits prints the first-install token to
its container log on a fresh install, which only root and the docker group can read.

## Known gaps

- **Redirected payments.** Anyone who takes over the internet-facing path can hand out their
  own invoices. Keep LNbits updated and watch that payments keep arriving. A BIP353 record with
  a BOLT12 offer, signed in DNS, removes the live server from the path; it isn't set up here yet.
- **Zap request signatures.** LNURLp 1.3.2 checks a zap request's structure but not its
  signature, so a forged request still gets an invoice. Nostr clients that verify zap receipts
  discard the result.
- **ACINQ** sees your payments and is your only channel peer.
