# Movie Chat integration reference

- Follow the [quick start](../README.md) to run the sample; use this reference when adapting its integration patterns.
- Retrieval text stays request-local on the server. API responses and browser diagnostics retain source metadata, not passages.
- Queries and cited URLs can still be sensitive service Content. Review [data handling](#data-handling-and-usage-constraints) and [deployment responsibilities](#scope-and-deployment-limitations).

Paths and commands below are relative to the sample root, not this `docs` directory.

| Topic | Reference |
|---|---|
| Setup | [Configuration](#configure), [certificate authentication](#web-iq-authentication) |
| Agent integration | [Grounding](#how-grounding-works), [retries](#rate-limiting-and-retries), [planning modes](#query-planning-modes) |
| Presentation and operations | [Citations](#citation-rendering), [media](#images-and-video), [diagnostics](#diagnostics-and-logging) |

## Architecture

```text
React + assistant-ui
        |
        | unauthenticated, same-origin JSON
        v
Express TypeScript backend
        |-- in-memory session history and movie preferences
        |-- LLM query planner using checked-in Web IQ guidance
        |-- Entra certificate credential for Web IQ Auto (web, news, images, videos)
        |-- AZURE_OPENAI_API_KEY for grounded responses
        v
Azure OpenAI grounded response with citations
```

Session IDs are stored in browser `sessionStorage`. The backend keeps bounded conversation history and preferences in process memory until session deletion or inactivity expiry; restarting it clears that state. Retrieved passages are used only during the request, not stored in sessions or returned to the browser. The browser retains the current transcript, preferences, citation/media metadata, and call diagnostics while the page is open. The server session TTL does not expire those browser copies or collected logs.

The assistant-ui adapter sends only the new user message and session ID to the existing JSON API; Express remains the owner of bounded conversation history. The browser transcript is not restored on refresh. New conversation clears the UI, preferences, and session ID and attempts to delete the old backend session; if deletion fails, that session expires normally.

## Prerequisites

- Node.js 22.12 or later on the 22.x line, or Node.js 24+
- A Microsoft Entra app registration mapped to a Web IQ AppID enabled for Auto
- An RSA certificate registered on that application, with its certificate and private key available to the backend in PEM format
- An Azure OpenAI deployment
- An Azure OpenAI API key

## Configure

Edit `config.json`:

```json
{
  "webiq": {
    "base_url": "https://api.microsoft.ai/v3",
    "max_response_length": 18000,
    "max_sources": 30
  },
  "azure_openai": {
    "endpoint": "https://<your-resource-name>.openai.azure.com/",
    "deployment": "<your-deployment-name>",
    "api_version": "<your-api-version>"
  },
  "query_planning": {
    "mode": "iterative",
    "max_iterations": 5,
    "max_queries_per_iteration": 5,
    "max_total_queries": 8
  },
  "server": {
    "port": 3000,
    "session_ttl_minutes": 120,
    "max_history_messages": 12
  }
}
```

### Web IQ authentication

The backend follows the certificate-based app identity pattern used by [Market Briefing](../../../python/market-briefing/docs/authentication.md). It explicitly creates Azure Identity's `ClientCertificateCredential` and requests `https://api.microsoft.ai/.default`. The shared `getBearerTokenProvider` handles in-memory token caching and refresh; each Auto request uses the current token in its `Authorization: Bearer` header.

The application identity must resolve to one provisioned Web IQ AppID with Auto enabled. A token alone does not establish access: for a 401 or 403, check the application mapping and entitlements instead of repeatedly signing in.

Set the following in the same PowerShell session that runs the backend:

```powershell
$env:AZURE_TENANT_ID = "<directory-tenant-id>"
$env:AZURE_CLIENT_ID = "<application-client-id>"
$env:AZURE_CLIENT_CERTIFICATE_PATH = "C:\path\to\webiq.pem"
$env:AZURE_OPENAI_API_KEY = "<your Azure OpenAI API key>"
```

`AZURE_CLIENT_ID` is the application's client ID, not its object ID. The PEM must contain both the public certificate and its private key; a public-only CER file is not enough. For an encrypted PEM, also provide its existing private-key password without putting it in command history:

```powershell
$password = Read-Host "Existing PEM private-key password" -AsSecureString
$env:AZURE_CLIENT_CERTIFICATE_PASSWORD = (
  [System.Net.NetworkCredential]::new("", $password).Password
)
```

Omit `AZURE_CLIENT_CERTIFICATE_PASSWORD` only for an unencrypted PEM. After finishing all runs in this shell, clear the process-local password:

```powershell
Remove-Item Env:\AZURE_CLIENT_CERTIFICATE_PASSWORD -ErrorAction SilentlyContinue
```

Unlike the Python credential used by Market Briefing, the JavaScript credential accepts PEM, not PFX/PKCS12. If your existing certificate is in a PFX, use an approved conversion tool or OpenSSL to export the same certificate and private key to an encrypted PEM **outside the repository**:

```powershell
openssl pkcs12 -in "C:\path\to\webiq.pfx" -out "C:\path\to\webiq.pem" -clcerts
```

OpenSSL prompts for the existing PFX password and a password to encrypt the output private key; use the latter as `AZURE_CLIENT_CERTIFICATE_PASSWORD`. Do not add `-nodes` or print the PEM contents. The same registered public certificate can be reused; converting the file format does not create a new application identity.

There is no credential-chain fallback: Azure CLI sign-in, managed identity, `AZURE_CLIENT_SECRET`, and `WEBIQ_API_KEY` are not used for Web IQ. Missing required environment settings stop startup; certificate and token errors fail the request rather than switching identities. Azure OpenAI remains separately API-key authenticated.

Use your organization's approved certificate issuance and secret-delivery mechanisms. Restrict access to the private-key file, monitor expiration, and rotate the registered certificate and deployed private key together. Never commit certificates, private keys, passwords, tokens, or API keys, or put them in `config.json`, logs, or container images. The sample does not provision applications, renew certificates, or retrieve them from a vault. See the [JavaScript certificate credential reference](https://learn.microsoft.com/en-us/javascript/api/@azure/identity/clientcertificatecredential).

Replace resource-specific values in your local configuration with your own settings; the JSON above is the portable configuration template.

Alternatively, set `MOVIE_CHAT_CONFIG` to the path of a separate configuration file. For a development session, disable detailed retrieval logging **before starting the server**:

```powershell
$env:MOVIE_CHAT_CONFIG = "C:\path\to\movie-chat-config.json" # Optional
$env:MOVIE_CHAT_LOG_RETRIEVAL_DETAILS = "false"
```

The logging setting omits queries and cited URL lists, but does not remove browser metadata diagnostics or address the other data-handling constraints.

## Install and run

From this directory:

```powershell
npm install
npm run dev
```

Open:

```text
http://localhost:5173
```

Vite serves the UI and proxies `/api` requests to the Express backend at `http://localhost:3000`. If you change the backend port, update the proxy target in `client/vite.config.ts` too.

Use **Send** or **Enter** to submit; **Shift+Enter** inserts a newline. During a request, sending and starting a new conversation are disabled to avoid overlapping changes to session history. Editing, regeneration, and branching are intentionally omitted because the backend API is append-only.

The **Try asking** buttons fill the composer without sending. They demonstrate recommendations, showtimes, recent movie news, source-specific comparisons, video trailers, and poster images.

## Build and run the compiled sample

```powershell
npm run build
npm start
```

Open `http://localhost:3000`. Express serves the compiled UI and API from one process. This creates production build artifacts; it does **not** disable diagnostics, change logging defaults, or add authentication.

## Example conversations

- `I am in Seattle. What science-fiction movies are showing tonight?`
- `I liked Arrival and Ex Machina, but I do not like graphic horror. What should I watch next?`
- `How did critics review the latest Mission: Impossible movie?`
- `Tell me about movies directed by Greta Gerwig and which one best fits my preferences.`
- `Show me the trailer for Avengers: Doomsday.`
- `Show me the Titanic movie poster.`
- `What are the latest movie announcements this week?`
- `Compare Arrival reviews from rogerebert.com and variety.com.`
- `I usually trust reviews from RogerEbert.com. Refine those recommendations.`

The model is instructed to update only preferences the user explicitly states or confirms; the server normalizes the returned fields but does not independently verify their provenance. Current preferences appear in the left panel.

Stored preferences are supplied to the existing Azure OpenAI query planner alongside the current message and bounded conversation history. In that same model call, the planner returns a required boolean `use_stored_preferences` based on the request's meaning, including non-English phrasing, negation, and explicit overrides. It is instructed to use stored preferences only for requested personalization, not unrelated factual questions or general recommendations, and to default to false when unclear. Preferences stated directly in the current message remain part of that message and can affect its query.

The server validates the boolean and carries it through follow-up planning and answer generation; `debug.useStoredPreferences` exposes the decision. Follow-up planners receive stored preferences only when enabled. Opting out for a turn does not erase stored preferences. This replaces the English phrase-matching heuristic without adding a model call. Intent handling remains model-based, not a guaranteed consent or privacy boundary; use explicit application controls where consent is required. Web IQ receives only the generated questions, not the preference object or conversation history; personalized questions can contain relevant preference details.

A preference-only statement can produce an empty query plan. In that case the assistant records the preference and responds without making an unnecessary Web IQ call.

## How grounding works

For each chat turn, the backend:

1. Sends the request, stored preferences, current UTC date, and up to six recent messages (2,000 characters each) to an Azure OpenAI query planner using `server/prompts/webiq-query-guidance.txt`. In the same call, the planner decides whether the current request asks for personalized retrieval and produces the queries. Conversation context resolves references such as `Who directed it?`; it is not fresh factual evidence or permission to apply historical preferences.
2. Validates the planner's JSON output, deduplicates it, and makes a separate `POST https://api.microsoft.ai/v3/auto` request for each focused query, with at most two queries active at a time.
3. Reads `results[].evidence` and each result's own `citations`, then deduplicates and numbers the supporting sources. Text, image, and video evidence are supported; unsupported or malformed items and explicitly adult results are omitted.
   In the default iterative mode, the planner assesses remaining gaps after retrieval and can issue further targeted queries using entities discovered in the evidence. It stops when no useful follow-up is proposed, the configured pass limit is reached, or the total search budget is exhausted.
4. Sends recent conversation history, preferences, and Web IQ evidence to Azure OpenAI.
5. Requires an original, concise summary with Web IQ citations and updated preference state as JSON. Checks for excessive source copying, retries once when necessary, and withholds an answer that still fails.
6. Returns sources cited in the generated answer, plus all retained sources with media metadata. The UI displays at most three media cards.
7. Stores the bounded history and preference state in the in-memory session.

Retrieved web content is treated as evidence, not as model instructions.

Auto returns typed evidence rather than the raw `webResults` contract from `/search/web`. Requests use the planner-selected Auto mode and a bounded `maxResponseLength`. An Auto-enabled AppID is required; the sample does not silently fall back to Web Search if Auto is unavailable. The endpoint is `/v3/auto`.

### Rate limiting and retries

The client follows the public [Web IQ error-handling guidance](https://learn.microsoft.com/en-us/web-iq/reference/error-handling#rate-limiting-429):

- Retries HTTP 429 throttling and transient HTTP 500 failures with bounded exponential backoff and jitter, up to three HTTP attempts per query. It does not automatically retry authentication, validation, network, or malformed-response failures.
- Honors the response body's `retryAfter` duration (such as `"60s"`) or ISO 8601 UTC timestamp. It also accepts standard HTTP `Retry-After` seconds or dates. When multiple hints apply, it waits for the longest; it never shortens a service-provided delay. Missing or invalid hints use a one-second initial backoff for 429 or the documented 30-second fallback for 500.
- Shares a transient-throttling cooldown and a two-active-query cap across all chat requests using this client instance. Daily quota exhaustion (`AuthUserDailyUsageExceeded`) fails immediately: it is logged and returned as HTTP 429 with a safe message explaining that the app's daily quota has been reached. The chat UI displays that message. No retry, midnight timer, daily-quota cooldown, or automatic resubmission is scheduled. These are sample, per-process safeguards, not the AppID's service rate limit. Check your limit in the Web IQ portal; applications sharing an AppID across processes need coordinated rate limiting.
- Uses one 60-second timeout signal for a retrieval batch, including queueing, retry waits, and HTTP attempts, rather than resetting it on each retry. A retry hint beyond that deadline fails promptly instead of waiting indefinitely or retrying early. A failed batch aborts its remaining HTTP requests and retry waits. Token acquisition uses the credential provider's own cancellation/timeout behavior.
- Counts every HTTP attempt, including retries, against the existing per-message search budget (eight by default), reserving initial attempts for the other planned queries. Retries can therefore leave fewer calls for follow-up evidence. Exhausting retries or the budget fails the retrieval rather than silently increasing the budget or presenting a throttled request as an empty result.

If a follow-up batch fails with transient 429/500 errors, a retry/cooldown deadline, or a retrieval timeout after earlier evidence was collected, the server stops retrieval and answers only from that retained evidence. The model is instructed to identify unsupported parts, and the application appends a visible retrieval-incomplete notice even if the model omits one. `debug.retrievalIncomplete` and a correlated `chat_retrieval_incomplete` log expose this outcome; call totals include failed HTTP attempts. Successful results from the failing batch are not retained. The model's explanation is not a guarantee of complete gap detection.

Initial retrieval failures, follow-up failures without retained evidence, authentication/validation/response-contract failures, and daily quota exhaustion still fail the request. Daily quota exhaustion always keeps its immediate HTTP 429 response; it is never converted into a partial success.

Each HTTP attempt has its own safe `webiq_request` log, with attempt number, status, trace ID, and elapsed time. Successful turns include failed retry attempts in API call diagnostics and totals; their `attempt` and `status` fields distinguish retries from new queries. Retry waiting is included in the next attempt's duration. A request blocked locally before HTTP dispatch is logged but does not consume an HTTP call; dispatched attempts without a response still count. No upstream error body is logged.

### Retrieval planning

The existing planner chooses `auto_mode` per query without an additional routing model call. It uses `lite` for focused movie facts, posters, trailers, showtimes, and specific evidence gaps. For comprehensive research across related entities, it prefers one coherent `advanced` query covering the full task before decomposing it. For example, a comparison of worldwide grosses for the 2024 Best Picture nominees starts as one advanced query; related missing figures should be grouped into the minimum useful follow-up queries, not automatically one search per film.

The server validates the selection, defaults an omitted `auto_mode` to `lite`, and forwards it as Auto's request `mode`. It rejects unsupported values. Changing mode alone does not bypass query deduplication or justify another search. Advanced prioritizes comprehensiveness, not lower latency: it may reduce follow-up rounds, but faster end-to-end answers are not guaranteed. The separate `query_planning.mode` setting below controls the application's planning loop, not Auto's `lite`/`advanced` mode.

**API-contract limitation:** the current code also sends `contentType: "passage"` and `maxLength: 3000`, but neither field is listed in the [published Auto reference](https://learn.microsoft.com/en-us/web-iq/reference/auto). Do not rely on these fields to select passage content, enforce account entitlements, or limit server-side output. A successful response does not establish that an undocumented field was honored. Confirm request fields against the supported contract for your enabled account.

Independently of that request, the client code in `server/src/webIqClient.ts` bounds normalized text supplied to the model to 3,000 JavaScript string code units per source URL. Duplicate URLs merge distinct passages rather than discarding new follow-up facts: up to eight distinct passages share that budget, with the newest eight retained and each passage's originating query preserved. Long passages share the available space, so truncation can still omit facts. Both initial and iterative retrieval select sources round-robin across originating queries before applying `max_sources`, rather than allowing the first query to consume the entire source allowance. Evidence assessment receives the same bounded content as answer generation, without additional clipping, plus originating query metadata. This is a local context budget, **not** a ToU display allowance or evidence of non-full-document account behavior. It is separate from the configured `max_response_length` total-response budget.

Each turn makes an initial query-planning model call and an answer-generation model call. The default iterative flow also makes evidence-assessment calls after retrieval to decide whether more searches are necessary; preference-only turns with an empty query plan skip assessment and retrieval. A copying-check failure adds at most one answer-generation call, using the same evidence without another Auto request. Auto does not guarantee correctness, and clickable citations do not establish claim support. Use synthetic fixtures for automated tests; do not retain service Content as model-training or evaluation datasets, or use it for the evaluation or improvement purposes prohibited by Section 3.2.14 of the ToU.

### Answer-copying safeguard

The answer prompt directs the model to paraphrase concisely, avoid quotations and reconstruction of entire articles, respect stricter known source-owner limits, and cite sources for full details. Grounding input may contain more text than the generated answer is permitted to reproduce.

Before returning an answer, `server/src/movieModel.ts` checks the message against **all** supplied evidence, not just cited sources. It normalizes Unicode, case, punctuation, whitespace, and numeric citation markers, then finds shared spans using 40-character windows. Matching answer spans are counted once where they overlap and accumulated per document URL across separate excerpts and repeated quotations; fragment links share the same budget while retaining their original display URLs. More than 200 normalized matching characters from one document triggers a single rewrite request with a stronger no-copying instruction.

Only a passing answer and its preference updates reach session history. Rejected answers are not returned, stored in the session, or logged. If the rewrite also fails, the API returns a safe error explaining that the answer was withheld; it emits no `chat_citations` event. Successful diagnostics count and time both answer calls when a rewrite was needed.

**This is a simple heuristic, not a compliance guarantee.** It can miss collections of excerpts shorter than 40 normalized characters, edited or translated copying, and copying from text not in the supplied evidence; it can also flag coincidental shared phrasing. Normalized counts are not exact counts of original displayed characters. It neither discovers nor enforces unknown source-owner limits below 200, proves claim support, nor detects all substitutional paraphrases. Review these limitations for your intended use.

### Citation rendering

Answers display citations as compact, Copilot-style domain pills rather than numbered references. Multiple adjacent sources share one pill with a `+N` count. Hover, keyboard focus, or a tap opens a preview with each source's title and full clickable URL; there is no separate source list below the answer. Escape, Close, or clicking outside dismisses the preview.

Internally, the model still returns numeric source markers such as `[1]`. The UI resolves those markers against the backend's returned sources and replaces recognized markers with domain pills; the numbers are an implementation detail, not the intended citation display.

Answers also support Markdown lists, emphasis, tables, and code. Single line breaks are preserved with `remark-breaks`, so plain-text bullet lines do not run together.

Citation IDs are scoped to that answer, not the whole conversation. The UI maps IDs to the backend's returned `sources`, never to a URL invented by the model. Unknown IDs and invalid/non-HTTP(S) source URLs remain unlinked. Citation syntax inside code is not transformed. Raw HTML and model-generated images are disabled.

Clickable citations demonstrate traceability, not semantic proof: a matching ID does not verify that a source actually supports the claim.

Source-specific prompts rely on the planner preserving the requested domains and Auto honoring the issued query. The sample does not enforce a returned-source hostname allowlist. Add that validation if an application requires answers exclusively from particular publishers.

### Images and video

Up to three retrieved media cards appear with each answer. Image cards use the service-returned thumbnail or HTTPS image URL and link to the original image and hosting page. Video cards load an embedded player only after **Play video** is clicked. Only validated YouTube and Vimeo player URLs can become iframes; unsupported providers or videos that disallow embedding remain source links. Provider restrictions, browser policies, or removed videos can still prevent playback, so **Watch at source** is always available.

Validated player URLs are used as returned by Auto, without changing the domain or adding playback parameters. Standard YouTube embeds use `youtube.com` rather than being rewritten to privacy-enhanced mode; provider cookies and browser tracking policies can affect sign-in and playback.

Media comes from typed Auto evidence and its associated citations, never from Markdown image tags or HTML produced by the model. Model-authored images remain disabled. Image previews load from third-party hosts; starting a player contacts its provider and shares the app's origin as a referrer. The backend does not download, proxy, or persist media; browser and provider caching are outside that guarantee. Only text evidence and media metadata are supplied to the model, not the actual image or video. Displaying an image or trailer is not a grant of rights to reuse it.

### Diagnostics and logging

> [!WARNING]
> **Diagnostics retain queries and source metadata, not retrieved passages.** This reduces raw-content retention; it is not a compliance exemption for the remaining metadata, summaries, or logs.
>
> The [Web IQ Terms of Use](https://www.microsoft.com/en-us/webiq/webiq-legal) restrict Grounding Content use and display (Section 2.5 and Exhibit A). Exhibit A limits displayed Grounding Content from any single source to the lesser of 200 characters or the website owner's limit. Review source titles, URLs, other metadata, generated answers, and logs against the applicable terms; a collapsed debug panel does not change those obligations. Retain compliant source attribution alongside the generated answer.

Each assistant response includes a debug panel, collapsed by default, with separate model and Web IQ call counts:

- Model calls identify query planning, iterative evidence assessment, and answer generation. A copying-triggered rewrite appears as an additional answer-generation call with its own purpose and duration.
- Model and Web IQ entries show their actual elapsed time in milliseconds.
- Web IQ calls show every exact query, its purpose, and retrieval iteration. Their timing includes the complete Auto HTTP request; any work inside Auto is not counted as a separate application model call. Trace IDs remain in API diagnostics and server logs but are not displayed in the UI.
- Header totals show cumulative model time and cumulative Web IQ call time. Web IQ calls can run in parallel, so cumulative Web IQ time is not the same as end-to-end wall-clock latency.
- Preference-only turns show zero Web IQ calls while still showing their model-call stages.

Expand **Inspect evidence**, then an individual source, to see its source number, evidence type, title, clickable URL, originating queries, and update date when available. Source headings label **(Image)** and **(Video)** from typed media evidence, or **(Web)** and **(News)** from the citation's `contentSource`. Text without a recognized category is labeled **(Text)** rather than guessing from the URL. The response's `debug.evidence` describes the deduplicated evidence supplied to the answer model, including uncited sources. It preserves `id`, `title`, `url`, `query`, distinct originating `queries`, `contentSource`, `updatedAt`, and media metadata when available. Both `content` and `passages` are omitted; normalized image/video descriptions are also server-only. Evidence metadata stays scoped to each answer.

The server writes newline-delimited structured JSON to stdout (`info`) and stderr (`error`), suitable for collection by a hosting platform or company log pipeline. Each event includes a timestamp and level. The app request ID is carried through asynchronous retrieval calls and returned in the `x-request-id` response header, so parallel queries and overlapping chat requests remain distinguishable.

- `api_request`: one event per completed API request, with request ID, method, route template, status, and elapsed milliseconds. Successful chat requests also include model/Web IQ call counts and cumulative timings.
- `webiq_request`: one event per Auto HTTP attempt or locally blocked attempt, with the exact issued query, selected `autoMode`, attempt number, Web IQ `traceId`, HTTP status, outcome, duration, and result/source counts. Status and trace ID are `null` when unavailable. The mode also appears in API call diagnostics. These events never include returned URL lists and are emitted even when another query or later answer generation fails.
- `chat_citations`: one event after successful answer generation, with `citedUrlCount` and deduplicated `citedUrls` resolved from that answer's recognized source IDs. Unknown IDs, uncited evidence, and sources included only for media cards do not add URLs. An uncited answer logs an empty list; failed answer generation emits no citation event. The request ID correlates this event with the retrieval queries and trace IDs.
- `chat_retrieval_incomplete`: an error-level event when a transient follow-up failure falls back to earlier evidence, with iteration, safe error type, failed-batch call count, and retained-source count. No passage text or upstream error body is included.
- `api_error`: correlated application failures, with error type rather than raw SDK error details. Failed Auto calls emit their own error-level `webiq_request`; unavailable HTTP status or trace ID is `null`. Network failures and invalid response bodies are also logged.

Query text and **only URLs cited in the generated answer** are logged by default. Both can contain personal or confidential information, and URLs can include sensitive query parameters. **Cited URLs remain Grounding Content under the ToU; citation alone does not authorize persistent troubleshooting logs.** Confirm an applicable express permission before retaining them. Review consent, access controls, retention, and redaction requirements for customer query data as well. To omit query and URL fields while retaining trace IDs, status, timing, and counts, set:

```powershell
$env:MOVIE_CHAT_LOG_RETRIEVAL_DETAILS = "false"
```

The application does not deliberately log API keys, authorization headers, complete chat payloads, answers, evidence bodies, preferences, or session IDs. Issued queries can nevertheless reproduce an entire user prompt or explicitly supplied preferences, so this is not a redaction guarantee. The detail opt-out only affects server logs, not browser query/source metadata. Restricting access alone does not establish ToU compliance. No persistent log store, automatic retention policy, or external telemetry integration is included.

### Query-planning modes

- `"mode": "iterative"` is the default. It lets the planner inspect accumulated evidence and issue targeted follow-up Auto queries for unresolved gaps and multi-hop prerequisites, at the cost of more model calls and latency.
- `"mode": "initial"` is the lower-latency, single-pass alternative. It performs one planning pass and then answers from the Auto evidence without a separate assessment loop. It can issue multiple initial queries, but cannot discover intermediate entities and then search them in a later pass.

`max_iterations` includes the initial retrieval pass and is capped at 5 by configuration validation. `max_queries_per_iteration` is a maximum, not a target; the planner should return only the minimum focused query set required.

`max_total_queries` defaults to **8 Web IQ calls per user message** when omitted and accepts integers from 1 through 40. It is a server-enforced ceiling shared across all retrieval passes in both planning modes, not a target or a conversation-wide allowance. Each query consumes at least one call, whether `lite` or `advanced`, and every transport retry consumes another; internal work performed by Auto is not separately counted. The planner receives the remaining allowance and the maximum queries for the next pass, prioritizes queries, and is instructed to batch related gaps. The server trims oversized plans before dispatch and skips further assessment when the budget is exhausted. Existing per-pass and iteration limits still apply.

At the total limit, the answer uses only the available evidence. The answer model is instructed to identify unsupported parts and avoid implying complete research; the server always appends a visible search-limit notice to an accepted answer. Reaching the limit does not itself prove the answer is incomplete, and the notice does not independently verify which claims have support. Preference-only turns still make no Web IQ calls. This per-message ceiling is not a request-rate limit, concurrency limit, or end-to-end time limit.

Evidence assessment should stop at the precision requested. Cumulative totals without an explicit historical cutoff should use the latest reported cumulative figures, not interim nomination-day or opening-week snapshots; the award year identifies the nominee set, not a cutoff for earnings. This does not automatically require an audit reconciling every re-release or independent verification of every row. The guidance favors supported dated figures, appropriate approximations, and explicit uncertainty over repeated searches merely to reconcile reporting dates or rounding. Material unexplained conflicts, wrong measures, incomplete historical snapshots, and genuinely missing information can still justify follow-ups within the budget. These are model instructions, not a guaranteed semantic deduplication mechanism; the total call ceiling is enforced separately by the server.

The default intentionally demonstrates caller-controlled multi-hop retrieval, not a requirement for using Auto. A question about the highest-paid actors and each actor's highest-grossing movie should first keep the related task together in an advanced query; any remaining gaps can use actors named in that evidence for targeted follow-ups. Follow-up queries must be based on actual evidence rather than invented names; the model must distinguish actor earnings from net worth and movie box office, and acknowledge missing or ambiguous figures.

Auto already orchestrates retrieval. For a simpler integration, pass a coherent task with its related constraints to Auto and use `initial` mode when a single pass meets the application's needs. Keep external follow-up loops only where they add useful evidence rather than duplicating Auto's work.

## Data handling and usage constraints

This documentation records the current behavior; it does not certify legal compliance. The [Web IQ Terms of Use](https://www.microsoft.com/en-us/webiq/webiq-legal), applicable customer agreement, supported API documentation, and source-owner restrictions govern use. The following constraints apply to diagnostics, logging, generated answers, and host applications.

| Area | Current limitation | Integration responsibility |
|---|---|---|
| Evidence metadata | `debug.evidence` contains source/query metadata, including uncited sources, but no `content` or `passages`. The browser retains metadata with each answer. | Review permitted display and retention of titles, URLs, queries, dates, and media metadata. Removing passages does not exempt other service Content or generated summaries. |
| Cited URL retention | Default `chat_citations` logs contain only URLs cited in the answer, but those URLs are still service Content. | Disable detailed retrieval logging unless persistent retention is expressly permitted for your use. Citation alone does not establish that permission. Do not retain Content in log collectors, traces, exported responses, screenshots, or fixtures without an applicable express permission. |
| Summary display | The prompt and copied-span heuristic reduce obvious reproduction, but do not establish semantic support, detect every substitutional summary, or enforce stricter source-owner limits. | Keep attribution near supported claims and address the safeguard's documented limitations. The applicable limit is the lesser of 200 characters or the website owner's maximum, not 3,000. The heuristic is not proof of compliance. |
| Transient processing | Retrieved passages remain request-local on the server and are not stored in sessions or returned by the API. Browser metadata and summary history outlive the request. | Keep Content transient only as strictly necessary for an LLM-initiated Atomic Action; promptly delete it when that action completes and prevent reuse or later availability. Do not cache retrieval for later chat turns; review the remaining metadata/history against permitted exceptions. |
| Supported Auto contract | The code sends undocumented passage fields and uses caller-controlled planning around Auto. | Reconcile the request and query guidance with the supported contract for your enabled account. Do not assume HTTP 200 confirms optional fields or copy Web Search-specific controls into Auto. |
| Deployment and privacy | UI authorization is out of scope. No inbound chat API rate limiting, end-user privacy notice, or source-owner policy enforcement is included. | The host application must supply appropriate access/network controls, privacy notices, source-use review, and operational controls. Review third-party media rights and provider behavior. These responsibilities are not implemented by this sample. |

Section 2.5 permits only the specified LLM uses of Grounding Content, with Content displayed in the same UI as LLM output. Section 2.5.1 permits media display using provided URLs and media metadata for grounding, not actual images/videos for grounding or derivative media. Section 2.5.2 also prohibits summaries that substitute for the underlying Content.

Sections 3.2.14 and 3.2.15 restrict training, evaluation, service-improvement uses, and Content storage. Exhibit A's exceptions are limited: per-end-user LLM-summary chat history; qualifying internal work product containing summaries and associated attribution links solely for display; and strictly transient Atomic Action processing. They are not a general permission to save returned URLs, raw evidence, or debugging captures. The sample's generated chat history does not authorize retention of the accompanying raw evidence.

Exhibit A also requires a source-navigation experience and compliant source attribution, respecting applicable website-owner restrictions. Do not add non-Microsoft advertising near Responses or Microsoft attribution to returned Content unless the applicable terms permit it; identifying this repository as a Web IQ sample is not a substitute for source attribution or evidence of branding permission. Review applicable news-publisher obligations and privacy terms before deployment. Use this movie sample for its stated non-safety-critical purpose.

## Scope and deployment limitations

The sample focuses on server-side Web IQ integration and a useful demonstration UI: upstream credentials stay on the backend, retrieval is bounded and correlated with request/trace IDs, and returned evidence drives citations and media cards. These patterns are a starting point, not a complete production security boundary.

**UI authentication and authorization are deliberately out of scope.** No login screen, identity-provider integration, or user-to-session authorization is included. A company can integrate the sample behind its existing application or hosting access controls without adopting a new UI identity system from this project. Same-origin requests and random session IDs do not authenticate users.

| Area | Sample behavior | Host application's responsibility |
|---|---|---|
| Network access | The backend calls `listen(port)` without an explicit loopback host. A localhost URL does not guarantee loopback-only binding. | Keep local runs behind appropriate firewall controls. For hosting, supply HTTPS and restrict ingress to the intended audience; do not expose the unprotected API directly to an untrusted network. |
| Abuse and cost | Request/message sizes and retrieval budgets are bounded. The Web IQ client has a per-process concurrency cap, but no inbound API rate or concurrency limiter is included. Disabling Send in the UI is only a UX measure. | Limit request frequency, simultaneous work, and upstream usage for the deployment. |
| Credentials | Web IQ uses an explicit Entra certificate credential and server-side bearer tokens. Azure OpenAI reads `AZURE_OPENAI_API_KEY`. Neither is sent to the browser. | Provision and rotate the application certificate and API key securely, restrict upstream permissions and private-key file access, and avoid putting secrets in source control or client bundles. |
| Sessions | History/preferences are process-local and expire after inactivity. Session IDs are not tied to authenticated identities. | Apply the host's session-access policy. For multiple instances, choose intentional routing or an appropriate shared state design; do not persist raw retrieval evidence as session state. |
| Privacy and retention | Queries and cited URLs can be logged; the browser retains evidence metadata but no retrieved passages. No privacy notice or automatic log-retention policy is included. | Address the Content-handling constraints above, provide appropriate privacy notices, and establish permitted collection, access, redaction, and retention policies. |
| Publisher and media use | HTTP(S) links and safe Markdown rendering constrain rendering, not publisher rights or source eligibility. Third-party media loads from its provider. | Review source-owner restrictions, applicable news obligations, media rights, and third-party privacy behavior for the intended use. |

These are documented integration responsibilities, not a request to add deployment infrastructure or UI authorization to the sample.

## Commands

```powershell
npm run dev
npm run typecheck
npm test
npm run build
npm start
```
