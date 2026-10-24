# Operations

Every command below runs on the host, from anywhere: `bin/satpost` finds its own directory.

## Sweeping to cold storage

1. In your coordinator (for example Sparrow with your hardware wallet loaded watch-only), get a
   fresh receive address.
2. Verify it on the signing device itself, for example with SeedSigner's address verification.
   A compromised server or laptop could swap the address; the device can't be fooled.
3. `bin/satpost balance`, then `bin/satpost sweep <address> <sats> <sat/vB>`. The command prints
   the address in groups of four to compare once more, and asks you to type `send`.

The sweep is a splice-out: the channel stays open and shrinks, and only the mining fee is paid.

## Liquidity

- `bin/satpost channels` shows capacity and inbound liquidity: how much more you can receive
  for free.
- When a payment doesn't fit, phoenixd buys another block (`PHOENIXD_AUTO_LIQUIDITY`, 2M sats by
  default) out of that payment. `bin/satpost fees <sats>` shows what a payment of that size would
  pay right now.
- Sweeping doesn't restore inbound liquidity; paying out over Lightning does.
- ACINQ may reclaim liquidity you haven't used a year after buying it. Your balance and the
  channel stay; the next payment that doesn't fit buys a new block.

## Updates

Versions are pinned in `.env`. Read the release notes, change the version, then:

```
docker compose pull && docker compose up -d
```

Watch the [phoenixd](https://github.com/ACINQ/phoenixd/releases) and
[LNbits](https://github.com/lnbits/lnbits/releases) releases on GitHub (Watch → Custom →
Releases). Update LNbits promptly: it's the part that faces the internet. Extension updates
are in LNbits' admin page.

## Backups

- **The seed** is the only thing that protects the money: 12 words on paper, made during setup.
- **Records** (payment history, payer comments, the pay link, the zap key) live in the
  `phoenixd-data` and `lnbits-data` volumes. If you want them kept, copy them somewhere safe,
  leaving out `seed.dat`:

```
docker compose exec -T phoenixd tar -C /phoenix/.phoenix --exclude=seed.dat -cz . > phoenixd-records.tar.gz
docker compose exec -T lnbits tar -C /app/data -cz . > lnbits-data.tar.gz
```

## Recovering on a new host

1. Repoint your address file at a custodial wallet's Lightning address for the meantime, so
   payments keep arriving.
2. On the new host, run `bin/satpost setup` once to create `.env` and edit it, then
   `bin/satpost restore` and type the 12 words (nothing is shown; it refuses to overwrite an
   existing seed). Then `bin/satpost setup`.
3. phoenixd reconnects to ACINQ and restores the channel and balance from its encrypted backup.
4. Recreate the LNbits pay link (or restore `lnbits-data`), then refresh the address file. Its
   callback URL changes with a new pay link.

Never run the same seed in two places at once: two phoenixd instances, or phoenixd and the
Phoenix app, cause force-closes.
