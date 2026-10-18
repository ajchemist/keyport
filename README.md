# Keyport

[한국어](README.ko.md)

A self-hosted Tailscale auth key desk on Cloudflare. Select an account and tailnet on your phone, then issue and copy a single-use key valid for 24 hours.

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/ajchemist/keyport)

## Install in your browser

1. Click the button and connect **your own GitHub and Cloudflare accounts**. Review the names of the repository, Worker, and D1 database being created.
2. Set the build command to `bun run build` and the deploy command to `bun run deploy`. The repository uses `bun.lock` and declares Bun 1.4.2 in `packageManager`. D1 is provisioned and bound automatically; you do not need to copy database IDs. The deploy command also applies migrations.
3. Open the deployed Worker URL to launch the **setup wizard**. No local CLI is needed.
4. Follow the wizard to protect the entire Worker with Access and allow only your actual administrator emails. Generate an encryption key in your browser and secure the recovery file.
5. Save the five values prepared by the wizard as **Secrets** under Worker **Settings → Variables and Secrets**, then deploy. Sign in through Access to open the app.

Set the Preview command to `bunx wrangler preview` (or `bun run preview` with the latest source). Cloudflare may still display its own default, `npx wrangler preview`; repository scripts do not overwrite that dashboard field. Worker Previews are separate from `wrangler versions upload`. The empty `previews` block exposes setup instructions only: management remains blocked until separate test D1 bindings, Secrets, and Access are configured. Do not use production credentials for previews. [Preview configuration](https://developers.cloudflare.com/workers/previews/configuration/)

The button automates repository cloning, Workers Builds integration, D1 provisioning and binding, migrations, and deployment. **You complete Access policies and Secret configuration in the wizard.** The app does not store a Cloudflare account management token. End-to-end installation in a fresh Cloudflare account remains unverified. [Official deployment guide](https://developers.cloudflare.com/workers/platform/deploy-buttons/)

If you started setup before switching to Bun, refresh the commands above and ensure your cloned repository contains the latest source. Existing clones do not automatically receive upstream changes.

**Does the button require Git?** Yes. The official flow connects your GitHub/GitLab account, creates a copy of the source repository in your account, and connects it to Workers Builds. You do not need to create a repository beforehand. Manual deployment with Wrangler is a separate option that does not require a Git integration.

Only setup instructions are public before configuration. The management UI and APIs remain blocked until configuration is complete. Afterward, setup instructions also require authentication. The first visitor never becomes an administrator automatically.

## Use

- Create an account profile with a display name and expected owner, then register a tailnet identifier and **personal API access token (`tskey-api-`)**. Use a user-owned token, not OAuth credentials.
- Registration verifies access to the tailnet's key list. The expected owner is **administrator-supplied and unverified**. Successful listing does not prove the owner's identity or permission to issue keys. Token expiry dates are also administrator-supplied.
- Select an account and network, issue a key, check the selected connection, expected owner, and expiration, then copy and deliver it privately yourself.
- The server enforces single-use, preauthorized, non-ephemeral, untagged keys with a 24-hour lifetime. Tailnet Lock may require a separate signature. Keep existing security policies enabled.
- Register the same tailnet under different profiles with different users' tokens. All allowed administrators jointly manage every connection.
- Stored tokens are never shown again. Issued keys stay only in the current page's memory and are removed when cleared, switching connections, leaving or backgrounding the page, or reloading. Copied values may remain in the OS clipboard.
- Issuance is never retried automatically. Timeouts, incomplete responses, and `pending` audit records can mean the result is unknown. Check the Tailscale console and revoke unwanted keys before issuing another if needed.
- Personal API token expiry (up to 90 days), the issued key's 24-hour expiry, and device node key expiry are separate. Replace API tokens before they expire. [API authentication](https://tailscale.com/docs/reference/tailscale-api), [auth keys](https://tailscale.com/docs/features/access-control/auth-keys)

## Architecture and security

Astro static pages, UnoCSS, a TypeScript Worker, and one D1 database. No frontend runtime framework, KV, R2, Durable Objects, service worker, or analytics scripts.

Tokens use AES-256-GCM with random 96-bit nonces and the connection ID as additional authenticated data. The master key is a Worker Secret separate from D1. Requests go only to the fixed Tailscale API host; redirects are rejected. Tokens and issued keys never go into URLs or persistent browser storage. Upstream response bodies are excluded from server errors.

Configure Worker-wide Access protection. The server additionally verifies JWT signatures, issuer, audience, expiration, and allowed administrator emails. It rejects domains outside the configured single origin. Static Assets Workers do not receive native `ctx.access`, so the server verifies JWTs directly. `run_worker_first` routes asset requests through the Worker too. [Access documentation](https://developers.cloudflare.com/workers/configuration/cloudflare-access/)

Mutations require the correct Origin, JSON Content-Type, and a dedicated header. Cross-origin CORS is disabled. Atomic D1 updates enforce a five-second issuance interval per administrator and 30 seconds per connection. Every response uses `no-store`, including CDN cache controls. There is no offline issuance, secret request queue, or restoration of old keys after restart.

## Updates and operations

The [operations guide](docs/OPERATIONS.md) covers updates, rollback, token renewal, backup and recovery, encryption key rotation, and resource deletion. The [verification record](docs/VERIFICATION.md) distinguishes automated checks from outstanding live tests. Both documents currently use Korean. Domain terms live in [CONTEXT.md](CONTEXT.md); design decisions live in [ADRs](docs/adr/).

Upstream changes are not automatically merged into your cloned repository. Apply updates to the production branch connected to Workers Builds to trigger deployment. Preserve existing D1 binding IDs and Secrets.

Deleting a connection removes local credentials only. Existing auth keys and devices remain. Revoke external API tokens, unwanted keys, and devices separately in Tailscale.

## Local development

Bun 1.4.2 or later, plus Node 24 or later for Wrangler:

```sh
bun install --frozen-lockfile
bun run types
bun run check
bun test
bun run build
bun run db:local
bun run dev
```

Bun runs the test suite with `bun:test` and `bun:sqlite`, and runs Astro and TypeScript with `bunx --bun`. Wrangler uses `bunx wrangler`, which follows its Node shebang: forcing Wrangler onto Bun currently causes local requests to hang. Node 24 is retained solely for Wrangler; `.node-version` selects it in supported build environments. See the [upstream compatibility issue](https://github.com/cloudflare/workers-sdk/issues/15717). Cloudflare still runs the deployed app on Workers. Workers Builds detects the Bun lockfile; if selecting a Bun version explicitly, set the build variable `BUN_VERSION=1.4.2`.

Local development starts unconfigured and exposes only the setup wizard. There is no production authentication bypass. Tests use synthetic JWTs, temporary SQLite, and a mocked Tailscale API. Never put real tokens in test files or chat.

## Costs

As checked on October 19, 2026, a small administrator instance can start within Workers Free, D1 Free, and Access Free allowances. Free operation is not guaranteed.

| Resource | Free allowance / cost conditions |
| --- | --- |
| Workers | 100,000 requests/day and 10 ms CPU per invocation. Static requests also pass through the authentication Worker. Exceeding limits can cause failures; Paid adds a subscription and usage charges. |
| D1 | Five million rows read/day, 100,000 rows written/day, and 5 GB total storage. Free limits can restrict operations. Paid charges apply beyond included usage. |
| Access | Free for small teams. Check plan requirements for teams of 50 or more and additional features. |
| Builds | Build minutes, concurrency, and monthly allowances depend on the Workers Builds plan. |

[Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/), [D1 pricing](https://developers.cloudflare.com/d1/platform/pricing/), [Access pricing](https://www.cloudflare.com/sase/products/access/), [Builds limits and pricing](https://developers.cloudflare.com/workers/ci-cd/builds/limits-and-pricing/). Domain, IdP, and Tailscale charges are separate. Include other account usage when assessing costs.

## License and references

MIT © 2026 ajchemist. [tailscale-x/tailscale-cloudflare](https://github.com/tailscale-x/tailscale-cloudflare) was reviewed, but its current structure differs from the referenced Worker project and no license could be confirmed, so no code was reused. No code or secrets from the private onboarding project are included.
