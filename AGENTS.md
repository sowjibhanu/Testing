<!-- BEGIN:nextjs-agent-rules -->
# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` before writing any code. Heed deprecation notices.
<!-- END:nextjs-agent-rules -->

<!-- BEGIN:logging-rules -->
# Structured Logging Only

Do NOT use `console.log`, `console.warn`, or `console.error` in the `forge-orchestrator` app. All logging must go through the structured logger at `apps/forge-orchestrator/src/logger.ts`:

```ts
import { log } from "./logger";

log.info("job done", { jobId: "abc123" });
log.error("job failed", { jobId: "abc123", error: "timeout" });
```

Every log line is emitted as a single JSON object with `ts`, `level`, `slug`, and `msg` fields. This is required for CloudWatch Insights filtering by deploy slug. Pass contextual data (jobId, error, etc.) as the second argument — never interpolate into the message string.
<!-- END:logging-rules -->

<!-- BEGIN:bedrock-proxy-rules -->
# Bedrock LLM Access

All LLM calls in the orchestrator MUST go through the abstraction in `apps/forge-orchestrator/src/llm-client.ts` — never create a `BedrockRuntimeClient` directly.

When `BEDROCK_PROXY_URL` is set in `.env`, LLM calls are routed through a proxy service (no AWS credentials needed locally). When unset, calls go directly to AWS Bedrock (requires IAM role or AWS credentials).

To add a new Bedrock call: import `converseWithLLM` from `./llm-client` and pass a `ConverseCommandInput` object. The response shape is identical to `ConverseCommandOutput`.

For local development or Devin sessions: set `BEDROCK_PROXY_URL` and `BEDROCK_PROXY_KEY` in `.env`. See `docs/local-dev.md` for details.
<!-- END:bedrock-proxy-rules -->
