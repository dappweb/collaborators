# Collaborators

Collaborators connects GitHub issues with USDC-denominated bounties. Maintainers
post a bounty, contributors submit a pull request, and signed GitHub webhooks
track when a submitted solution is merged.

Visit [collaborators.build](https://collaborators.build) to use the hosted app.
This repository contains its Next.js application and API.

## Current workflow

### Contributors

1. Sign in with GitHub through Privy. Privy provides an embedded Solana wallet
   for users without one.
2. Browse **Active Bounties** in the dashboard and read the linked GitHub issue.
   Confirm the scope and acceptance criteria with the maintainer before starting.
3. Open a pull request in the bounty's repository.
4. Click **Submit Solution** on the bounty and enter the GitHub PR URL before
   it is merged. A linked GitHub account is required; each user can submit once
   per bounty and cannot submit to their own bounty.
5. After the maintainer merges the submitted PR, the repository's webhook marks
   the bounty solved and approves the matching submission.

### Maintainers

1. Sign in with GitHub and configure a repository webhook as described below.
   The repository must appear as connected before its issues are loaded.
2. Open **My Issues**, select an issue, and choose **Add Bounty**. Enter a positive
   USDC amount. Only one bounty can exist per GitHub issue.
3. Review the contributor's PR and merge the accepted solution.
4. Use **Solved Bounties** to find the solver's wallet address and arrange the
   agreed payment manually.

### What a solved bounty means

The current webhook updates PostgreSQL records and credits the solver's
`unclaimedTokens` balance. It does **not** send an on-chain USDC transfer or
escrow funds when a bounty is posted. Maintainers pay contributors manually;
the bounty amount and solved status are not proof of payment.

NFT minting and automatic SOL rewards from the older project are not part of
the active bounty workflow. Legacy components are retained in `archive/`
directories and are not the dashboard's payout mechanism.

## Stack and source map

- Next.js 15.3 / React 19 / TypeScript
- Tailwind CSS 4
- Privy for GitHub login and Solana wallet management
- Prisma 6 / PostgreSQL for users, bounties, submissions, and repository tracking
- GitHub APIs and signed repository webhooks

| Path | Purpose |
| --- | --- |
| [`src/app/dashboard/page.tsx`](src/app/dashboard/page.tsx) | Issues, active/created/solved bounties, and submission UI |
| [`src/app/PrivyProviders.tsx`](src/app/PrivyProviders.tsx) | Login methods, embedded wallet, and chain configuration |
| [`src/hooks/usePrivyAuth.ts`](src/hooks/usePrivyAuth.ts) | Authentication, linked GitHub account, and wallet state |
| [`src/app/api/bounties/`](src/app/api/bounties/) | Bounty CRUD, submissions, and solved bounty queries |
| [`src/app/api/github/`](src/app/api/github/) | GitHub issue APIs, webhook, and repository connection records |
| [`prisma/schema.prisma`](prisma/schema.prisma) | Database models and bounty/submission statuses |
| [`scripts/check-webhook.mjs`](scripts/check-webhook.mjs) | Local webhook integration smoke check |

## Run locally

### Prerequisites

- Node.js 20 or newer and pnpm 10
- A PostgreSQL database dedicated to local development
- A configured Privy app with GitHub login enabled

```bash
git clone https://github.com/andr-drgm/collaborators.git
cd collaborators
pnpm install --frozen-lockfile
```

Installation generates the Prisma client. Create `.env` in the repository root;
there is currently no checked-in `.env.example`. Replace the placeholders below:

```dotenv
DATABASE_URL="postgresql://USER:PASSWORD@localhost:5432/collaborators"
NEXT_PUBLIC_PRIVY_APP_ID="your-privy-app-id"
PRIVY_APP_SECRET="your-privy-app-secret"
GITHUB_WEBHOOK_SECRET="your-local-webhook-secret"
```

Using `.env` lets both Next.js and the Prisma CLI read the same database URL.
Environment files are git-ignored. Keep server secrets out of `NEXT_PUBLIC_*`
variables, which are visible in the browser.

| Variable | Used for |
| --- | --- |
| `DATABASE_URL` | Prisma's PostgreSQL connection |
| `NEXT_PUBLIC_PRIVY_APP_ID` | Public app ID for client and server |
| `PRIVY_APP_SECRET` | Server-side Privy authentication |
| `GITHUB_WEBHOOK_SECRET` | Required webhook verification secret |
| `GITHUB_TOKEN` | Optional server token for issue lookup/search/labels |
| `GITHUB_ACCESS_TOKEN` | Optional fallback for user issue and commit queries |
| `NEXT_PUBLIC_X_URL` | Optional footer social link |
| `NEXT_PUBLIC_PRIVACY_URL` | Optional footer privacy link |
| `NEXT_PUBLIC_TERMS_URL` | Optional footer terms link |

GitHub access tokens returned by Privy are used for authenticated GitHub calls.
Enable **Return OAuth tokens** in the Privy dashboard and allow
`http://localhost:3000` as a development origin. Configure GitHub OAuth in Privy
with callback `https://auth.privy.io/api/v1/oauth/github/callback`. The active
provider enables GitHub login only and creates Solana embedded wallets; its
default chain is devnet. Check the provider configuration before deploying.
The app requires a valid Privy app ID to render; the example placeholder is not
a working login configuration.

Apply the existing migrations to your local database, then start the app:

```bash
pnpm exec prisma migrate deploy
pnpm dev
```

Open [localhost:3000](http://localhost:3000). When changing the database schema,
use `pnpm exec prisma migrate dev` to create a new development migration.

## Repository webhooks

For each repository, the maintainer needs permission to manage its webhooks:

1. Sign in first so the app's database contains your GitHub username.
2. In the repository, open **Settings → Webhooks → Add webhook**.
3. Set the payload URL to `https://YOUR_DEPLOYMENT/api/github/webhook` and content
   type to `application/json`. For local testing, use an HTTPS tunnel to the
   local server; GitHub cannot deliver to `localhost` directly.
4. Enter the same secret as the server's `GITHUB_WEBHOOK_SECRET`.
5. Select **Issues** and **Pull requests**, keep the webhook active, and add it.
6. Check the initial `ping` delivery and the repository's connected status in
   the dashboard. If it was added before signing in, redeliver the ping.

Missing or invalid signatures are rejected. A merged PR is matched by its
repository, PR number, and author's linked GitHub username. Redelivering that
event does not credit the bounty twice. Closing an issue alone cancels the
active bounty; a matching merged PR event can still resolve it afterward.
Automatic bounty-label application is not implemented by the webhook helper.

## Checks and commands

| Command | Purpose |
| --- | --- |
| `pnpm dev` | Start the Turbopack development server |
| `pnpm lint` | Run the repository's Next.js lint script |
| `pnpm exec tsc --noEmit` | Check TypeScript without writing output |
| `pnpm exec prisma validate` | Validate the database schema |
| `pnpm build` | Generate Prisma client and create a production build |
| `pnpm start` | Serve an existing production build |

To exercise the webhook with synthetic records, use a running development
server and a **disposable local database** that has the migrations applied:

```bash
DATABASE_URL="postgresql://USER:PASSWORD@localhost:5432/collaborators_test" \
GITHUB_WEBHOOK_SECRET="the-same-local-secret-as-the-server" \
BASE_URL="http://localhost:3000" \
node scripts/check-webhook.mjs
```

Start the server with that same test database and secret. The script checks
invalid signatures, PR author matching, duplicate delivery, and issue/PR event
ordering. It creates temporary records and deletes them afterward, and refuses
non-local database URLs. No wallet transaction is performed.

## Deployment and troubleshooting

Set the environment variables on the deployment host and allow its origin in
Privy. `pnpm build` does not apply database migrations. The checked-in
[`vercel.json`](vercel.json) build command instead runs `prisma db push` before
`next build`; review that schema synchronization step before targeting a
database with existing data. A manual deployment can apply the checked-in
migrations with `pnpm exec prisma migrate deploy` before building.

- **Login fails:** check the Privy app ID, enabled GitHub method, allowed origin,
  and OAuth callback configuration.
- **GitHub issues are empty:** check **Return OAuth tokens**, repository access,
  and the connected webhook record. Some issue APIs use the optional server
  `GITHUB_TOKEN` rather than the linked user's token.
- **Webhook returns 401:** ensure the GitHub secret matches the server variable.
- **Merged PR stays pending:** check that the solution was submitted before the
  merge, the PR belongs to the bounty repository, and the submitter's linked
  GitHub username matches the PR author. Inspect GitHub's webhook delivery log.
- **Database errors:** verify the database URL and migrations, then regenerate
  the client with `pnpm exec prisma generate`.

## Contributing

Use [GitHub issues](https://github.com/andr-drgm/collaborators/issues) for bug
reports and proposals. Keep a PR focused, explain its behavior, and include the
checks you ran. For a bounty, agree on its acceptance criteria and register the
PR through **Submit Solution** as described above. Documentation-only changes
should verify the commands and source references they describe.
