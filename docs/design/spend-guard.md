# Spend guard (parked design)

**Not built.** Parked in October 2026: sends are rare enough that `satpost pay` and `satpost sweep`
over SSH do the job. This records the design agreed with a dashboard (satbook), in case spending
from a dashboard is ever wanted.

## Goal

Let a dashboard send from the node, from a phone or a laptop, without the dashboard ever holding
phoenixd's full-access password, and with limits that still hold if the dashboard is compromised.

## What it would be

A small service in the satpost stack, in the same network namespace as phoenixd. It listens on
loopback, and serve publishes it to the tailnet on its own port, never through Funnel. It holds
the full-access password from `.env`; the dashboard only holds a token for the guard. LNbits stays
receive-only.

## Rules the guard enforces

- **Sweeps go only to your cold wallet.** The guard holds a watch-only xpub for a dedicated account
  on the cold wallet's seed, for example `m/84'/0'/1'`, which SeedSigner exports with Custom
  Derivation. It hands out that account's addresses in order and refuses any other destination.
  With a dedicated account, nothing else hands out those addresses. Add the account to your
  coordinator so sweeps show up there.
- **Lightning payments have a daily cap**, over a rolling 24 hours and including fees. Anything
  larger goes through the CLI.
- **Every spend needs a TOTP code** from an authenticator app, enrolled on the host and never
  through the dashboard. A code approves one preview and can't be reused, and repeated failures
  lock the guard for a while.
- **Fee ceilings:** a maximum feerate, and a maximum fee relative to the amount, so a hijacked
  dashboard can't burn the balance on mining fees.
- **A high daily cap on sweeps**, as a backstop.
- **An audit log** of every request, refusals included.

Even with the dashboard compromised, the worst case is sweeps to your own cold wallet and Lightning
payments up to the daily cap, each still needing a code from your phone. A compromised dashboard
could show you one preview and submit another with your code; the caps and the xpub restriction
are what bound that.

## API

JSON over HTTPS on the tailnet, with the token in a header.

- `GET /limits` → `dailyCapSat`, `usedTodaySat`, `resetsAt`.
- `POST /sweep/address` → the next unused cold address, with its index, derivation path and script
  type, to check on the signing device.
- `POST /preview` with `kind` set to `sweep` or `pay` → `previewId`, the destination, `amountSat`,
  an estimated `feeSat` and a `maxFeeSat`, the feerate for a sweep, `capRemainingSat` and
  `expiresAt`. For a pay, the guard decodes the invoice or resolves the address itself, so the
  preview shows its own reading of the payee, amount and description.
- `POST /execute` with `previewId` and `totp` → `paymentId` or `txId`, `amountSat` and the actual
  `feeSat`. A preview is single-use and short-lived, and executing it twice never pays twice.
- `GET /audit?since=` → log entries, refusals included.

Errors carry a code: `totp_invalid`, `totp_reused`, `cap_exceeded`, `preview_expired`,
`not_from_xpub` or `insufficient_funds`.

A pay's preview shows the guard's upper bound on the fee. A sweep's shows the feerate times the
estimated size as an approximation, with the balance after it. phoenixd 0.9.3 doesn't expose its
own splice-out estimate, so the guard computes one. The real fee always comes from phoenixd's
outgoing record.

## Dashboard side

- The spend screens exist only when `SPEND_GUARD_URL` and `SPEND_GUARD_TOKEN_FILE` are set.
- The ledger records spends from the synced phoenixd records only, never from a preview or an
  execute. A sweep becomes a transfer to the cold wallet with the actual mining fee and txid; a pay
  becomes a spend, or a transfer when the payee is another wallet of yours.
- Each sweep address the guard hands out is saved to the cold wallet's addresses, so chain lookups
  recognize the sweep.

## Network

One grant: the dashboard's tag to `tag:pay` on the guard's port, with policy tests that deny that
port to everything else, your own devices and the host included.

## Alternatives considered

- **The dashboard holds the full-access password.** Simplest, but every limit is then only as
  strong as the dashboard.
- **LNbits' NWC extension with a budget.** LNbits answers the internet, so it stays receive-only.
- **Alby Hub on top of phoenixd.** Popular, and gives apps such as Nostr clients budgets over NWC,
  but it has no xpub-locked sweeps or per-spend codes, and it's a second full wallet app. Worth
  evaluating if zaps from a Nostr client are ever wanted.

## If it's revived

- Decide the cold wallet account, the caps, and whether synced spends are recorded automatically or
  reviewed.
- Build sweeps first, then Lightning payments, and NWC only if it's wanted.
- Python with embit, the library SeedSigner uses, for address derivation; RFC 6238 for the codes;
  phoenixd's own API for payments and splice-outs.
- Test on testnet with `compose.test.yml` before mainnet, and have whoever maintains the host
  review it, since it adds spending power to the machine.
