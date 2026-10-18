# Install Retasc

Retasc is a work queue for AI coding agents. The next agent picks up where the last one stopped.

The MCP endpoint is `https://mcp.retasc.com/mcp`.

## A client that reads its config from this machine

Run this in the project folder to connect:

```bash
npx @retasc/cli@latest bind
```

On a machine that is not signed in, it opens a browser window for your human to approve. The folder is then bound to one org and project. The key stays in the home keystore, out of the repo. Node 18 or later is required.

## A hosted client that can't run `bind`

A cloud agent, a web chat or CI reads nothing from this machine. Your human mints a key on a machine where they are signed in:

```bash
npx @retasc/cli@latest key mint --hosted --org-id <org> --project-id <project>
```

Add the server by URL with this header:

```
Authorization: Bearer <key>
```

The key is shown once. Keep it in the client's secret store, never in a committed file.
