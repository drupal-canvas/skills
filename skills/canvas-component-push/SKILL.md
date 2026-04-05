---
name: canvas-component-push
description:
  Push validated Canvas component changes to Drupal Canvas and recover from
  common push failures. Use after component work is complete and validated.
  Handles dependency-related push failures that require retry.
---

## Push to Canvas

Before pushing, confirm the user has Drupal Canvas CLI installed and configured
for their target site.

### Setup gate

Before running any pull or push command, confirm the user has a
`CANVAS_ACCESS_TOKEN` and the site URL. These are passed inline — no `.env`
setup required:

```bash
CANVAS_ACCESS_TOKEN=your-bearer-token npx canvas pull --site-url https://your-site.com
CANVAS_ACCESS_TOKEN=your-bearer-token npx canvas push --site-url https://your-site.com --yes
```

If the user has not yet provided a token and site URL, **stop** and ask them to
supply both before continuing.

**Alternative — OAuth client credentials:**

If the user does not have a pre-issued token, they can use OAuth instead. Check
that a `.env` file exists in the project root with these values set:

- `CANVAS_SITE_URL`
- `CANVAS_CLIENT_ID`
- `CANVAS_CLIENT_SECRET`

If `.env` is missing or any required value is missing, **stop** and ask the
user to complete setup first. Point them to the official docs:

- Drupal Canvas OAuth module setup:
  <https://git.drupalcode.org/project/canvas/-/tree/1.x/modules/canvas_oauth#2-setup>
- Drupal Canvas CLI package/docs:
  <https://www.npmjs.com/package/@drupal-canvas/cli>

Continue only after the user confirms setup is complete.

## Run pull then push

Always run `canvas pull` before `canvas push` to sync remote components
locally. Pushing without pulling first risks overwriting or losing components
that exist on the remote but are not present in the local working tree.

### Step 1: Pull

Pull the latest components from Canvas before making any push. Make sure to use
the right package manager. For example, if using npm:

```bash
# With CANVAS_ACCESS_TOKEN
CANVAS_ACCESS_TOKEN=your-bearer-token npx canvas pull --site-url https://your-site.com

# Without CANVAS_ACCESS_TOKEN (OAuth via .env)
npx canvas pull
```

### Step 2: Push

When component work is complete and validated, ask the user if they would like
to push. `canvas push --yes` pushes all current changes; it does not support
selecting specific components. If there are unrelated or unvalidated Canvas
changes in the working tree, stop and ask the user how they want to proceed.

```bash
# With CANVAS_ACCESS_TOKEN
CANVAS_ACCESS_TOKEN=your-bearer-token npx canvas push --site-url https://your-site.com --yes

# Without CANVAS_ACCESS_TOKEN (OAuth via .env)
npx canvas push --yes
```

## Handling push failures

Default behavior: **always retry failed pushes** unless the error is clearly a
connection/setup failure.

Retry pushes when the failure indicates the Canvas app connection is already
working (for example, dependency/order-related component errors). Do **not**
retry connection/setup failures.

### Connection/setup failures: Stop, do not retry

If push fails with authentication, authorization, or network/connection errors,
stop and ask the user to complete or verify setup first. This includes errors
like invalid credentials, unauthorized/forbidden responses, DNS issues,
connection refused, host unreachable, request timeout before reaching Canvas, or
TLS/SSL handshake/certificate failures.

Point the user to the official setup docs:

- Drupal Canvas OAuth module setup:
  <https://git.drupalcode.org/project/canvas/-/tree/1.x/modules/canvas_oauth#2-setup>
- Drupal Canvas CLI package/docs:
  <https://www.npmjs.com/package/@drupal-canvas/cli>

Ask them to verify their auth setup and retry only after they confirm it is
corrected.

- OAuth: verify `.env` values `CANVAS_SITE_URL`, `CANVAS_CLIENT_ID`,
  `CANVAS_CLIENT_SECRET`
- Bearer token: verify the inline `CANVAS_ACCESS_TOKEN` value and
  `--site-url` argument are correct

### Dependency-related failures

When pushing multiple new components where one component depends on another
(e.g., `hero` imports `heading`), the push may fail with a message indicating
that a component doesn't exist. This happens when a component that includes
another gets pushed before its dependency.

**This is expected behavior.** Simply retry the push command. On subsequent
attempts, the dependencies that were successfully pushed in the previous run
will already exist, allowing the dependent components to push successfully.

Example scenario:

1. First push attempt: `hero` fails because `heading` doesn't exist yet, but
   `heading` pushes successfully.
2. Second push attempt: `hero` now succeeds because `heading` exists.

If pushes continue to fail after multiple retries, check that all required
dependency components are part of the current local changes or already exist in
Canvas.
