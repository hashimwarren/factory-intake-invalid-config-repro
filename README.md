# Factory GitHub intake `invalid_config` reproduction

Minimal reproduction for a Mastra Factory bug: saving a GitHub repository as an intake source fails when GitHub is configured but Linear is not.

This repository is an untouched `create-factory` scaffold, apart from this README, the package name/description, and the screenshot below. It contains no application-specific business logic, custom UI, or credentials.

## Expected behavior

Selecting a connected GitHub repository under **Settings → General → Intake sources** should save successfully when Linear is not configured.

## Actual behavior

The save fails with an `invalid_config` toast. The request to `PUT /web/intake/config` receives:

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{"error":"invalid_config"}
```

![Factory settings showing the invalid_config error](docs/invalid-config.png)

## Versions and environment

- OS: Windows
- Node.js: 24.18.0
- npm: use the version bundled with Node.js 24
- `mastra`: `1.20.2-alpha.4`
- `@mastra/factory`: `0.2.1-alpha.3`
- GitHub integration: configured
- Linear integration: intentionally not configured

The package lock is committed so the reproduction installs the exact versions that produced the error.

## Prerequisites

- Node.js 22.19 or newer
- PostgreSQL with the `pgvector` extension
- A WorkOS project
- A GitHub App
- A GitHub repository that the App can access

Do not configure a Linear OAuth application for this reproduction.

### Required callback URLs

- WorkOS redirect: `http://localhost:4111/auth/callback`
- GitHub App callback: `http://localhost:4111/auth/github/callback`

The GitHub App needs:

- Contents: read and write
- Issues: read and write
- Pull requests: read and write
- Metadata: read-only

## Setup

1. Install the locked dependencies:

   ```bash
   npm ci
   ```

2. Copy `.env.example` to `.env`.

3. Set the following values in `.env`:

   ```dotenv
   MASTRACODE_PUBLIC_URL=http://localhost:4111
   DATABASE_URL=postgres://user:pass@localhost:54329/mastracode_web

   WORKOS_API_KEY=your-workos-api-key
   WORKOS_CLIENT_ID=your-workos-client-id
   WORKOS_COOKIE_PASSWORD=a-random-secret-at-least-32-characters-long

   GITHUB_APP_ID=your-github-app-id
   GITHUB_APP_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----"
   GITHUB_APP_CLIENT_ID=your-github-client-id
   GITHUB_APP_CLIENT_SECRET=your-github-client-secret
   GITHUB_APP_SLUG=your-github-app-slug
   ```

   Leave `LINEAR_CLIENT_ID` and `LINEAR_CLIENT_SECRET` unset.

4. Start the included local PostgreSQL service, or use another PostgreSQL database with `pgvector`:

   ```bash
   npm run db:up
   ```

5. Start Factory:

   ```bash
   npm run dev
   ```

6. Open <http://localhost:4111>.

## Steps to reproduce

1. Sign in through WorkOS.
2. Connect and install the configured GitHub App.
3. Select a GitHub repository and create a Factory for it.
4. Open **Settings → General → Intake sources**.
5. Enable **GitHub repositories**.
6. Select the connected repository.
7. Observe the `invalid_config` toast.

The browser sends a body shaped like this:

```json
{
  "github": {
    "enabled": true,
    "sourceIds": ["owner/repository"]
  },
  "linear": {
    "enabled": false,
    "sourceIds": null
  }
}
```

## Why the request fails

The Factory API returns an intake configuration map containing only integrations registered by the deployment. With GitHub configured and Linear absent, that map contains only `github`.

The generated UI's `normalizeIntakeConfig` function in `src/web/ui/domains/factory/services/intake.ts` adds a disabled `linear` entry to create a fixed client-side shape. `saveIntakeConfig` then serializes the entire fixed object.

The server rejects the request because `linear` is not in its registered integration list, even though the submitted Linear entry is disabled:

```ts
if (!config || Object.keys(config).some(integrationId => !integrationIds.includes(integrationId))) {
  return c.json({ error: 'invalid_config' }, 400);
}
```

Either of these behaviors would resolve the mismatch:

- The client submits only integration keys returned by the server.
- The server ignores disabled entries for integrations that are not registered.

## Verification

Static checks:

```bash
npm run check
```

The final UI reproduction requires the WorkOS and GitHub App credentials described above. No LLM provider or API key is required to trigger this bug.
