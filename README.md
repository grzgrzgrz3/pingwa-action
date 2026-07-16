# pingwa-action — WhatsApp notify & approve for GitHub Actions

Send a WhatsApp notification from any workflow, or **pause the workflow until
you tap a button on your phone**. Runs on [pingwa](https://pingwa.dev) — the
official Meta Cloud API, no linked device, no Twilio, no WABA of your own.

Two modes:

- **`notify`** — fire-and-forget ping (`POST /v1/notify`).
- **`ask`** — send a question with up to 3 buttons and long-poll for the tap
  (`POST /v1/ask`). The step blocks until you answer or the timeout hits.

## Get a key

Send **join** to the pingwa WhatsApp number (QR + number at
[pingwa.dev](https://pingwa.dev)) — the API key comes straight back in the
chat. Store it as a repository secret named `PINGWA_KEY`:
*Settings → Secrets and variables → Actions → New repository secret*.

## Deploy gate: approve the prod deploy from your phone

The gate step asks on WhatsApp and waits (up to 90 s). The deploy step runs
**only** when you tapped the first button (`b0`). No reply → the gate step
fails → the deploy never runs. Fail-safe: silence ships nothing.

```yaml
name: deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Ask before shipping
        id: gate
        uses: grzgrzgrz3/pingwa-action@v1
        with:
          api-key: ${{ secrets.PINGWA_KEY }}
          mode: ask
          message: "Deploy ${{ github.repository }} @ ${{ github.sha }} to prod?"
          buttons: "ship it,abort"

      - name: Deploy
        if: steps.gate.outputs.button-id == 'b0'
        run: ./deploy.sh
```

Button ids are **positional**: `b0` is the first label you passed ("ship it"
above), `b1` the second. Tapping "abort" leaves the gate step green with
`button-id=b1` — the deploy step is simply skipped and the job succeeds.

If the ask lands outside your 24h reply window (likely, for deploys that
arrive out of the blue), the buttons arrive as a numbered list instead of tap
buttons — replying **1** (or the button's text, **ship it**) works the same
and still maps to `b0`.

## Done-ping: notify when the job finishes

`if: always()` makes the ping fire on success *and* failure;
`${{ job.status }}` puts the verdict in the message.

```yaml
      - name: Ping when done
        if: always()
        uses: grzgrzgrz3/pingwa-action@v1
        with:
          api-key: ${{ secrets.PINGWA_KEY }}
          message: "${{ github.workflow }} on ${{ github.repository }}: ${{ job.status }}"
```

## Inputs & outputs

| Input | Default | Notes |
|---|---|---|
| `api-key` | — (required) | pingwa API key, from a repo secret |
| `message` | — (required) | text to send (notify) or question to ask (ask) |
| `mode` | `notify` | `notify` \| `ask` |
| `buttons` | `yes,no` | comma-separated labels for ask mode, max 3 |
| `timeout` | `90` | seconds to wait for the reply (90 = server max) |
| `base-url` | `https://pingwa.dev` | API base URL |

| Output | Notes |
|---|---|
| `button-id` | ask mode: positional id of the tapped button (`b0` = first) |

## Behavior

| What happened | HTTP | Step result |
|---|---|---|
| You tapped a button in time | 200 | success — `button-id` output set (`b0`, `b1`, …) |
| No reply within the timeout | 408 | step **fails** — anything gated behind it is blocked (fail-safe) |
| Any other error (bad key, rate limit, …) | 4xx/5xx | step fails |

The 408 still means the question was **delivered** — it stays on your phone,
and a late tap is retrievable via `GET /v1/messages/<id>/reply` if a later
job wants a second look.

## Pinning

`@v1` follows the latest v1.x release. For strict supply-chain hygiene, pin
the full commit SHA instead:

```yaml
uses: grzgrzgrz3/pingwa-action@<commit-sha>
```

## What's inside

A composite action: one bash step, `curl` + `jq` only (both preinstalled on
GitHub-hosted runners), ~40 lines, zero third-party dependencies. Read
[`action.yml`](action.yml) — that's the whole thing.

## Links

- [pingwa.dev](https://pingwa.dev) — the service (send **join** to start)
- [pingwa.dev/deploy](https://pingwa.dev/deploy) — the deploy-gate recipe page
- [pingwa.dev/llms.txt](https://pingwa.dev/llms.txt) — full API docs, agent-readable
- [github.com/grzgrzgrz3/pingwa-client](https://github.com/grzgrzgrz3/pingwa-client) — Python client, CLI + MCP server

## License

[MIT](LICENSE) © Grzegorz Grzywacz
