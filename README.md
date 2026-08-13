# mcp-ngrok-frontdoor

Drop-in TypeScript quickstart for testing an in-development MCP server against multiple AI model providers at once. Put an ngrok Cloud Endpoint in front of your existing stdio-based MCP server to centralize secret management, inspect traffic, replay requests, and test responses across model providers.

There are two pieces:

- **`http-transport.ts`**: a stateless Streamable HTTP transport with no
  dependency on any particular server's tools. Copy it into your server's
  `src/` directory.
- **`traffic-policy.yaml`**: the ngrok cloud endpoint traffic policy that authenticates
  model providers (one bearer token per provider, via ngrok Vaults/Secrets) and forwards
  authenticated traffic to your server.

## Requirements

- `@modelcontextprotocol/sdk` ^1.30.0+
- a Pay-as-you-go ngrok account
- the ngrok agent installed in your dev environment alongside the MCP server

## Steps

1. Copy `http-transport.ts` into your server's `src/` directory.

2. In your server entrypoint, add an HTTP branch next to your existing stdio
   startup:

   ```ts
   import { runHttp } from "./http-transport.js";

   // Reuse whatever function you currently pass to `server.connect(new StdioServerTransport())`.
   function buildServer() {
     const server = new McpServer({ name: "my-server", version: "1.0.0" });
     registerMyTools(server);
     return server;
   }

   const httpPort = process.env.MCP_HTTP_PORT ? Number(process.env.MCP_HTTP_PORT) : undefined;
   if (httpPort) {
     await runHttp(buildServer, httpPort);
   } else {
     // ...your existing stdio startup...
   }
   ```

3. Copy `traffic-policy.yaml`. The architecture it sets up is:

   ```
   MCP client --Bearer token--> Cloud Endpoint (managed by traffic policy, public URL)
     --forward-internal--> Agent Endpoint (bound to https://mcp.internal)
       --HTTP--> your MCP server (MCP_HTTP_PORT=<port>)
   ```

   a. Create a vault and then one secret per model provider you want to allow. These are tokens you generate yourself (not the provider's API key), so you can tell their responses apart in your backend:
      ```sh
      ngrok api vaults create --name "mcp-callers" --description "Tokens for dev MCP"
      ngrok api secrets create --name "claude-key" --value "$(openssl rand -hex 32)" --vault-id "$VAULT_ID"
      ngrok api secrets create --name "openai-key" --value "$(openssl rand -hex 32)" --vault-id "$VAULT_ID"
      ```

   b. Start your MCP server in HTTP mode, then bind an internal agent endpoint to it:
      ```sh
      MCP_HTTP_PORT=<port> node dist/index.js
      ngrok http <port> --url https://mcp.internal --host-header=rewrite
      ```

   c. In the ngrok dashboard, create a cloud endpoint with a public URL. Paste `traffic-policy.yaml` into its traffic policy editor.

   To add a model provider later, copy one of the rules in the traffic policy, add a new secret to the vault, and update the `x-mcp-caller` tag value.

4. On the provider side, add each key you generated as a custom header on the provider's MCP connector, and point the provider to`https://<your-cloud-endpoint-domain>/mcp`. For example:
   - Claude: [Third-party connectors with remote MCP](https://claude.com/docs/connectors/custom/remote-mcp)
   - OpenAI: [MCP and Connectors](https://developers.openai.com/api/docs/guides/tools-connectors-mcp)

5. Profit! Happy hacking.

## Notes

- `buildServer` must return a **new** `McpServer` instance each call. This is the MCP SDK's
  documented stateless pattern that ensures that concurrent requests from different model providers never share session state.
- The local ngrok agent tunnel needs `--host-header=rewrite` (or you need to pass
  `allowedHosts` to `runHttp`) — traffic arriving via `forward-internal`
  carries `Host: <your-service>.internal`, which the SDK's built-in DNS
  rebinding protection rejects by default.
- This only adds an HTTP path; it doesn't remove or change stdio behavior, so
  existing local clients (Claude Desktop, etc.) continue to work as usual.
