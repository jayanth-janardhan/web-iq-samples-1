# Web IQ Movie Chat

A small TypeScript starter for agent queries, linked citations, media evidence, and correlated logging. One Web IQ Auto integration retrieves reviews, movie facts, posters, trailers, and showtime information. React + [assistant-ui](https://www.assistant-ui.com/) provides the chat UI; Express owns retrieval and bounded session memory.

> [!WARNING]
> Review the [Web IQ Terms of Use](https://www.microsoft.com/en-us/webiq/webiq-legal) and [data-handling constraints](docs/integration.md#data-handling-and-usage-constraints) before making live requests. Retrieved passages stay server-side during the turn; API responses and browser diagnostics retain query/source metadata, not passage text. Queries and cited URLs are logged by default and can contain sensitive information. The sample has no UI/API authentication; do not expose it directly to an untrusted network.

## Quick start

You need Node.js **22.12+ on 22.x, or 24+**, a Web IQ Auto-enabled AppID mapped to a Microsoft Entra application with an RSA certificate, and an Azure OpenAI deployment/API key.

1. Edit `config.json` with your Azure OpenAI endpoint, deployment, and API version. The checked-in configuration contains the retrieval and session defaults; see the [configuration reference](docs/integration.md#configure).
2. Set credentials in PowerShell. The PEM must contain the registered public certificate and its private key; keep it outside the repository. For encrypted PEM or PFX conversion, follow the [certificate setup](docs/integration.md#web-iq-authentication).

   ```powershell
   $env:AZURE_TENANT_ID = "<directory-tenant-id>"
   $env:AZURE_CLIENT_ID = "<application-client-id>"
   $env:AZURE_CLIENT_CERTIFICATE_PATH = "C:\path\to\webiq.pem"
   $env:AZURE_OPENAI_API_KEY = "<your Azure OpenAI API key>"
   $env:MOVIE_CHAT_LOG_RETRIEVAL_DETAILS = "false"
   ```

3. From this directory, install and start:

   ```powershell
   npm install
   npm run dev
   ```

4. Open `http://localhost:5173`. Vite proxies `/api` to Express on port 3000.

The logging opt-out above omits queries and cited URL lists from server logs while preserving request/trace IDs, status, timing, and counts. It does not remove browser metadata or establish compliance. Never commit credentials or certificates.

## Try it

- `I liked Arrival and Ex Machina. What should I watch next?`
- `Compare Arrival reviews from rogerebert.com and variety.com.`
- `Show me the Titanic movie poster.`
- `Show me the trailer for Avengers: Doomsday.`

Citation pills open source links. **Debug** shows model/Web IQ queries, timings, and evidence metadata without retrieved passages. **New conversation** clears the browser transcript/preferences and attempts to delete the server session.

## Integration patterns

- **Plan:** an existing model call interprets the request, bounded history, and personalization intent; the server validates the structured query plan.
- **Retrieve:** call Auto with focused questions, bounded concurrency, 429/500 retry handling, and an eight-HTTP-call default turn budget.
- **Answer:** generate a concise, cited summary from evidence, with a bounded copying check. A transient follow-up failure can use earlier evidence with an explicit incomplete-retrieval notice; daily quota exhaustion returns an immediate error.
- **Observe:** correlate API requests, upstream attempts, and cited URLs without logging passage text. Source metadata remains available for citation/media rendering and diagnostics.

These are starter patterns, not guarantees of answer accuracy, consent enforcement, or legal compliance. Showtimes come from retrieved web pages, not a live ticketing feed; confirm them with the theater.

## Commands

```powershell
npm test
npm run typecheck
npm run build
npm start
```

After building, open `http://localhost:3000` for the compiled UI and API. A production build does not add authentication or change logging defaults.

## Detailed guidance

See the [integration reference](docs/integration.md) for certificate authentication, configuration, planner behavior, citation/media rendering, rate limits, diagnostics, data handling, and deployment responsibilities.
