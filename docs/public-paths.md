# Serving the payment paths

Wallets need two LNbits paths from the internet: `/.well-known/lnurlp/<username>`, which
describes the address, and `/lnurlp/api/v1/lnurl/cb/<id>`, which hands out invoices. satpost
serves them one of two ways. Either way they reach only LNbits, which only has phoenixd's
receive-only password, so nothing public can spend.

## Tailscale Funnel

The default, and the simplest: set `SATPOST_SERVE=serve.funnel` in `.env`, give `tag:pay` the
`funnel` attribute in your tailnet policy ([security.md](security.md)), and run `satpost up`.
The paths answer at `https://<TS_HOSTNAME>.<tailnet>.ts.net`.

## Your own reverse proxy or tunnel

`compose.proxy.yml` adds two small nginx containers, hardened like the rest:

- **proxy**, in its own network namespace, answers GET (and CORS preflights) on the two paths
  and 404 on everything else. It can only reach the forwarder, so never phoenixd.
- **forwarder**, in the shared namespace, listens only on an internal network that nothing but
  the proxy is on, and passes those connections to LNbits, which stays on 127.0.0.1.

To switch:

1. Put your reverse proxy or tunnel container on a Docker network of its own, add these lines
   to `.env` and run `satpost up`:

   ```
   COMPOSE_FILE=compose.yml:compose.proxy.yml
   SATPOST_PROXY_NETWORK=<that network>
   ```

2. Route `https://<your host>` to `http://satpost-web:8080`. Check from outside your tailnet
   that `https://<your host>/.well-known/lnurlp/<username>` answers, with a `callback` on your
   host, and that the callback returns an invoice for an `?amount=` in millisats.
3. Republish your address file from the new URL. With the workflow in [address/](../address/),
   change `LNADDRESS_SOURCE` and `LNADDRESS_CALLBACK_HOST` together.
4. Close Funnel: set `SATPOST_SERVE=serve`, run `satpost up`, and remove the `funnel`
   attribute from `tag:pay`.

Behind Cloudflare, wallets call these paths from code, not from a browser, so nothing may
challenge them. Turn Bot Fight Mode off for the zone (the Free plan can't exempt a single
host), add a configuration rule for the host that turns Browser Integrity Check off, and put
no Access application and no Under Attack mode in front of it. A cache rule that bypasses the
cache for the host makes sure no invoice is ever served twice.

## Trade-offs

|                      | Tailscale Funnel                                                                                    | Your proxy or tunnel                                                                                                                     |
| -------------------- | --------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| You need             | Funnel on your tailnet                                                                              | a domain, and a proxy or tunnel on a Docker network                                                                                      |
| Who reads requests   | only your server: TLS ends there, and Tailscale relays encrypted bytes                              | your provider too: it ends TLS, so it sees amounts, comments, zap requests and payers' addresses, and could hand out invoices of its own |
| The public host      | `<TS_HOSTNAME>.<tailnet>.ts.net`, which names your tailnet                                          | your own                                                                                                                                 |
| Speed and uptime     | relayed through Tailscale's servers, which can add seconds to a request; Funnel has its own outages | your provider's network                                                                                                                  |
| Extra parts to watch | none                                                                                                | two nginx containers and an internal network                                                                                             |
| Bot protection       | nothing in the way                                                                                  | has to stay off for these paths                                                                                                          |

If the same provider runs your domain's DNS, it could already point your address elsewhere;
ending TLS adds that it can read the requests.
