# 7. Web research

Goal: ask "what are people saying about X on Reddit, GitHub and forums" from a phone and get a
summary with sources, without any third-party search API key.

## Parts

- **SearXNG** in Docker (pinned image tag from 2026-09), on 127.0.0.1 only, JSON output enabled,
  rate limiter off (only the server talks to it), GitHub and Hacker News engines added. There is
  no Reddit engine any more; Reddit results arrive through the general engines.
- **Open WebUI web search** pointed at SearXNG's `/search?q=<query>`, 8 results, **embedding
  bypassed**: fetched pages go straight into the model's 128K context, capped at 12,000
  characters per page.
- **Forced search:** Open WebUI's default function-calling mode is set to **legacy**, so when
  the user switches the globe on, a search always runs. In "native" mode the model sometimes
  skipped searching and invented plausible sources.
- **Query writing and titles** use the thinking-off alias of the 35B (fast).

## Measured

A real question about a niche new tool: about 38K tokens of fetched pages, cited answer in
4.6 minutes.

## Gaps

- [Test with 3+ concurrent users doing research]
- [Local Deep Research or a similar multi-step research agent: planned, not built]
- [Whether the default engine set gets rate-limited by public engines under heavy use]
