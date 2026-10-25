# satpost

_a self-hosted Lightning address._

Get paid at `you@yourdomain.com`, straight into your own [phoenixd](https://github.com/ACINQ/phoenixd)
wallet.

- [LNbits](https://github.com/lnbits/lnbits) answers the address, with payer comments and nostr zaps.
- Only the address lookup and invoice creation are public, through Tailscale Funnel. Everything
  else stays on your tailnet.
- LNbits only has phoenixd's receive-only password, so nothing public can spend your money.
- You spend with the `satpost` command over SSH: pay invoices and Lightning addresses, or sweep
  on-chain. It's a small wrapper around phoenix-cli.

Best when you mostly receive, like donations or tips. Spending works too, from the command line.

## How it works

```
payer's wallet ──► https://yourdomain.com/.well-known/lnurlp/you      (a static file you host)
               ──► https://pay.<tailnet>.ts.net/lnurlp/api/v1/lnurl/cb/…  (Funnel: two paths only)

one network namespace, owned by the Tailscale container:
   LNbits   127.0.0.1:5000  ── receive-only password ──►  phoenixd 127.0.0.1:9740 ──► ACINQ
   tailnet only:  :8443 LNbits admin     :9740 phoenixd API (for a dashboard, if you allow one)

spending:  ssh host → satpost pay … / satpost sweep …
```

Nothing is published on the host, and the services only listen on localhost inside the
namespace. Tailscale still forwards tailnet traffic to any port in there, so your tailnet
policy matters: see [docs/security.md](docs/security.md).

## What it costs

phoenixd buys inbound liquidity automatically, in blocks of 2M sats by default
(`PHOENIXD_AUTO_LIQUIDITY`). A purchase costs 1% of the incoming payment plus the block, the
mining fee, and on the first purchase a 1,000 sat channel creation fee: about 22k sats in all.
After that, receiving is free until the block is used up. Sweeping out doesn't give the room
back; only spending over Lightning does.

**Fund the channel yourself before you publish the address.** Until a channel exists, payments
too small to cover the purchase are kept as fee credit, which is non-refundable. Sending is
0.4% + 4 sats per payment; on-chain sweeps cost only the mining fee you choose.

## Requirements

- A Linux host with Docker Compose (amd64 or arm64).
- A Tailscale tailnet with MagicDNS and HTTPS certificates on, and access to its policy file.
- A domain whose root serves static files (GitHub Pages works) for the address file.
- Paper, for the seed.

## Setup

1. **Tailscale policy.** Add the `tag:pay` owner, the Funnel attribute and the access rules
   from [docs/security.md](docs/security.md). Optionally create a single-use, pre-approved auth
   key tagged `tag:pay`.
2. **Configure.** `git clone` this repo on the host and put the command on your PATH:
   `ln -s "$PWD/bin/satpost" ~/.local/bin/satpost`. Run `satpost setup` once to create `.env`,
   edit it (`TS_HOSTNAME`, `TS_AUTHKEY`, `SATPOST_DOCKER`), then run `satpost setup` again. It
   generates LNbits' secrets, starts phoenixd, hands LNbits the receive-only password and starts
   LNbits. Re-running it is safe.
3. **Log the device in**, if you left `TS_AUTHKEY` empty: open the link in `satpost logs ts`
   and tag the device `tag:pay` in the admin console. A tagged device
   belongs to the tag rather than to you, so `autogroup:self` rules don't reach it, and its key
   no longer expires.
4. **Back up the seed yourself**, on paper, and keep it offline: `satpost seed`. Twelve words
   restore the wallet, channel included (ACINQ keeps an encrypted copy of the channel state for
   you). To use your own entropy instead, for example dice rolls on a SeedSigner, run
   `satpost restore --replace` before funding the channel, then `satpost setup`. It only replaces
   a wallet that has never had a channel or funds.
5. **LNbits.** Open `https://<TS_HOSTNAME>.<tailnet>.ts.net:8443`, finish the first install with
   `LNBITS_FIRST_INSTALL_TOKEN` from `.env`, with a strong, unique password. Under Extensions,
   enable Pay Links, then create a pay link with your username, your domain, comments allowed and
   zaps on. LNbits pre-fills the domain with the admin page's own address, so replace it with
   yours. LNbits fixes the description wallets show to "Payment to `<username>`".
6. **Fund the channel.** `satpost fees 35000` shows what opening it costs, and whether a 35k
   payment opens it or would only become fee credit. Pay `satpost invoice 35000 bootstrap` from
   another wallet, then check `satpost channels`.
7. **Open the payment paths.** Set `SATPOST_SERVE=serve.funnel` in `.env` and run
   `satpost up`. From outside your tailnet,
   `https://<TS_HOSTNAME>.<tailnet>.ts.net/.well-known/lnurlp/<username>` now answers.
8. **Publish the address file.** Save that response byte for byte as
   `https://yourdomain.com/.well-known/lnurlp/<username>`. On GitHub Pages the dot directory needs
   care: add `.nojekyll` when publishing from a branch, or set `include-hidden-files: true` on
   `actions/upload-pages-artifact` (v4 and later) when deploying from Actions. [address/](address/) has an hourly workflow that keeps the copy
   current and alerts you if the live response changes in a way it won't follow.
9. **Test** with two different wallets, and check that comments show up in LNbits.

## Everyday use

```
satpost balance                        balance and fee credit
satpost payments 20                    recent incoming payments
satpost pay <invoice>                  pay a Lightning invoice
satpost pay <user@domain> <sats>       pay a Lightning address
satpost sweep <address> <sats> <fee>   send on-chain; check the address on your signer first
satpost help                           everything else
```

[docs/operations.md](docs/operations.md) covers sweeping, liquidity, updates, backups and
recovering on a new host.

## Testing locally

`compose.test.yml` runs the same stack on testnet without Tailscale, with phoenixd's API on
`127.0.0.1:19740` and LNbits on `127.0.0.1:15000`. Its header lists the three `.env` lines it needs.

## License

MIT
