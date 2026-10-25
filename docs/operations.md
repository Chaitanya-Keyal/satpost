# Operations

Every command below runs on the host. `satpost` finds its own folder, so once it's on your PATH
(see the README) it works from anywhere.

## Sweeping to cold storage

1. In your coordinator (for example Sparrow with your hardware wallet loaded watch-only), get a
   fresh receive address.
2. Verify it on the signing device itself, for example with SeedSigner's address verification.
   A compromised server or laptop could swap the address; the device can't be fooled.
3. `satpost balance`, then `satpost sweep <address> <sats> <sat/vB>`. The command prints
   the address in groups of four to compare once more, and asks you to type `send`.

The sweep is a splice-out: the channel stays open and shrinks, and only the mining fee is paid.

- The fee comes out of your balance and phoenixd can't quote it in advance, so leave a little
  behind: about 1,000 sats at a few sat/vB. If phoenixd answers with an insufficient-funds
  failure, try a bit less. Whatever stays goes out with the next sweep.
- If a sweep sits unconfirmed too long, `satpost bumpfee <sat/vB>` raises its feerate and pays
  the difference from your balance. The channel keeps working in the meantime.
- Don't close the channel to get everything out. Closing gives up the inbound liquidity you paid
  for, and the next payment that doesn't fit buys it again.

## Liquidity

- `satpost channels` shows capacity and inbound liquidity: how much more you can receive
  for free.
- When a payment doesn't fit, phoenixd buys another block (`PHOENIXD_AUTO_LIQUIDITY`, 2M sats by
  default) out of that payment. `satpost fees <sats>` shows what a payment of that size would
  pay right now.
- Sweeping doesn't restore inbound liquidity; paying out over Lightning does.
- ACINQ may reclaim liquidity you haven't used a year after buying it. Your balance and the
  channel stay; the next payment that doesn't fit buys a new block.

## Updates

Versions are pinned in `.env` by tag and digest, so a re-pushed tag can't change what runs.
Read the release notes, then set the new tag and its digest, which
`docker buildx imagetools inspect <image>:<tag>` prints as `Digest:`, for example
`PHOENIXD_VERSION=0.9.4@sha256:…`. Then run `satpost update`.

Watch the [phoenixd](https://github.com/ACINQ/phoenixd/releases) and
[LNbits](https://github.com/lnbits/lnbits/releases) releases on GitHub (Watch → Custom →
Releases). Update LNbits promptly: two of its endpoints are public. LNbits installs its
extensions itself, so the LNURLp version isn't pinned by the image; update it from LNbits'
admin page when LNbits offers a new one.

## Backups

- **The seed** is the wallet's only backup: 12 (or 24) words on paper. It restores
  the channel too, because ACINQ keeps an encrypted copy of its latest state.
- **Never back up phoenixd's database to restore later.** Channel state goes stale with the
  next payment, and restoring an old copy can publish a revoked state, which loses the channel.
- **Payment history**, for your records: `satpost export history.csv`.
- **LNbits' data** (the pay link, payer comments, the zap key and the receive-only password),
  if you want it kept. Keep it private and encrypt it if it leaves the server. From the
  satpost folder:

```
(umask 077; docker compose exec -T lnbits tar -C /app/data -cz . > lnbits-data.tar.gz)
```

LNbits' own logs, in that data and in `docker compose logs lnbits`, record its wallet page's
invoice key. Redact them before you share them.

## Recovering on a new host

1. Repoint your address file at a custodial wallet's Lightning address for the meantime, so
   payments keep arriving.
2. On the new host, run `satpost setup` once to create `.env` and edit it, then
   `satpost restore` and type the words (nothing is shown; it refuses to overwrite an existing
   seed). Then `satpost setup`.
3. phoenixd reconnects to ACINQ and restores the channel and balance from its encrypted backup.
4. Recreate the LNbits pay link (or restore LNbits' data), then refresh the address file. Its
   callback URL changes with a new pay link.

Never run the same seed in two places at once: two phoenixd instances, or phoenixd and the
Phoenix app, cause force-closes.
