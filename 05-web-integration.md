# Web Integration Patterns

> §1–6: HTTP fetch, web search, rate limiting, private IP blocking, content transformation,
> and citation grounding.
>
> Sources: Gemini CLI (`web-fetch.ts`, `web-search.ts`), Claurst (`web_fetch.rs`, `web_search.rs`)

---

## §1 — Web Fetch Tool Design

A production web-fetch tool needs more than `fetch(url)`. It must:

- **Block SSRF** — prevent agents from fetching internal network addresses
- **Rate-limit per hostname** — avoid hammering a single server
- **Convert HTML to text** — models don't benefit from raw HTML
- **Handle binary content** — images/PDFs via base64 inline data
- **Cite sources** — link grounding metadata to output

```typescript
interface WebFetchInput {
  url: string;
  method?: 'GET' | 'POST';
  headers?: Record<string, string>;
  body?: string;
}

interface WebFetchResult {
  content: string;          // text or base64
  contentType: string;
  citations?: Citation[];
  truncated: boolean;
}
```

---

## §2 — SSRF Prevention (Private IP Blocking)

Never let an agent fetch internal services. Resolve hostnames and check before connecting.

```typescript
import { promises as dns } from 'dns';

// CIDR ranges to block
const BLOCKED_CIDRS = [
  '127.0.0.0/8',    // loopback
  '10.0.0.0/8',     // private class A
  '172.16.0.0/12',  // private class B
  '192.168.0.0/16', // private class C
  '169.254.0.0/16', // link-local (cloud metadata)
  '::1/128',        // IPv6 loopback
  'fc00::/7',       // IPv6 unique local
];

function isPrivateIP(ip: string): boolean {
  // IPv4 fast-path checks
  const parts = ip.split('.').map(Number);
  if (parts.length === 4) {
    if (parts[0] === 127) return true;
    if (parts[0] === 10) return true;
    if (parts[0] === 172 && parts[1] >= 16 && parts[1] <= 31) return true;
    if (parts[0] === 192 && parts[1] === 168) return true;
    if (parts[0] === 169 && parts[1] === 254) return true;
  }
  // IPv6
  if (ip === '::1' || ip.startsWith('fc') || ip.startsWith('fd')) return true;
  return false;
}

async function validateUrl(url: string): Promise<void> {
  const parsed = new URL(url);

  if (!['http:', 'https:'].includes(parsed.protocol)) {
    throw new Error(`Blocked protocol: ${parsed.protocol}`);
  }

  // Resolve hostname and check all resolved IPs
  const addresses = await dns.resolve(parsed.hostname).catch(() => [parsed.hostname]);
  for (const addr of addresses) {
    if (isPrivateIP(addr)) {
      throw new Error(`Blocked: ${parsed.hostname} resolves to private IP ${addr}`);
    }
  }
}
```

---

## §3 — Per-Hostname Rate Limiting

```typescript
import { LRUCache } from 'lru-cache';

interface RateWindow {
  count: number;
  windowStart: number;
}

const RATE_LIMIT_REQUESTS = 10;
const RATE_LIMIT_WINDOW_MS = 60_000;  // 60 seconds

const hostnameWindows = new LRUCache<string, RateWindow>({
  max: 500,
  ttl: RATE_LIMIT_WINDOW_MS,
});

function checkRateLimit(hostname: string): void {
  const now = Date.now();
  const window = hostnameWindows.get(hostname);

  if (!window || now - window.windowStart > RATE_LIMIT_WINDOW_MS) {
    hostnameWindows.set(hostname, { count: 1, windowStart: now });
    return;
  }

  window.count++;
  if (window.count > RATE_LIMIT_REQUESTS) {
    const retryAfter = Math.ceil(
      (window.windowStart + RATE_LIMIT_WINDOW_MS - now) / 1000,
    );
    throw new Error(
      `Rate limit: ${hostname} exceeded ${RATE_LIMIT_REQUESTS} req/min. ` +
      `Retry after ${retryAfter}s.`,
    );
  }
}
```

---

## §4 — Content Transformation Pipeline

### HTML to Markdown

```typescript
import { convert as htmlToText } from 'html-to-text';

function htmlToMarkdown(html: string, url: string): string {
  return htmlToText(html, {
    wordwrap: 120,
    selectors: [
      { selector: 'a', options: { hideLinkHrefIfSameAsText: true, baseUrl: url } },
      { selector: 'img', format: 'skip' },  // skip decorative images
      { selector: 'nav', format: 'skip' },
      { selector: 'footer', format: 'skip' },
      { selector: 'script', format: 'skip' },
      { selector: 'style', format: 'skip' },
      { selector: 'h1', options: { uppercase: false } },
    ],
    limits: { maxInputLength: 2_000_000 },
  });
}

// Content routing by MIME type
async function transformContent(
  body: Buffer,
  contentType: string,
  url: string,
): Promise<{ content: string; isBase64: boolean }> {
  const mime = contentType.split(';')[0].trim().toLowerCase();

  if (mime === 'text/html') {
    return { content: htmlToMarkdown(body.toString('utf-8'), url), isBase64: false };
  }

  if (mime === 'application/json') {
    try {
      const parsed = JSON.parse(body.toString('utf-8'));
      return { content: JSON.stringify(parsed, null, 2), isBase64: false };
    } catch {
      return { content: body.toString('utf-8'), isBase64: false };
    }
  }

  if (mime.startsWith('text/') || mime === 'application/xml') {
    return { content: body.toString('utf-8'), isBase64: false };
  }

  if (mime.startsWith('image/') || mime === 'application/pdf') {
    // Return as base64 for multimodal models
    return { content: body.toString('base64'), isBase64: true };
  }

  // Unknown binary: partial hex dump
  const hex = body.slice(0, 256).toString('hex');
  return { content: `[Binary content, ${body.length} bytes]\nFirst 256 bytes: ${hex}`, isBase64: false };
}
```

### GitHub URL Normalization

```typescript
// Convert GitHub blob URLs to raw content for direct download
function normalizeGitHubUrl(url: string): string {
  const ghBlob = /^https?:\/\/github\.com\/([^/]+\/[^/]+)\/blob\/(.+)$/;
  const match = url.match(ghBlob);
  if (match) {
    return `https://raw.githubusercontent.com/${match[1]}/${match[2]}`;
  }
  return url;
}
```

---

## §5 — Full Web Fetch Implementation

```typescript
const FETCH_TIMEOUT_MS = 10_000;
const MAX_RESPONSE_BYTES = 2_000_000;   // 2 MB
const OUTPUT_MAX_CHARS = 100_000;       // chars injected into LLM context

export async function webFetch(input: WebFetchInput): Promise<WebFetchResult> {
  const url = normalizeGitHubUrl(input.url);

  // 1. Validate
  await validateUrl(url);
  const hostname = new URL(url).hostname;
  checkRateLimit(hostname);

  // 2. Fetch with timeout and size limit
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), FETCH_TIMEOUT_MS);

  let response: Response;
  try {
    response = await fetch(url, {
      method: input.method ?? 'GET',
      headers: {
        'User-Agent': 'Mozilla/5.0 (compatible; AgentBot/1.0)',
        ...input.headers,
      },
      signal: controller.signal,
      redirect: 'follow',
    });
  } catch (e) {
    clearTimeout(timer);
    if ((e as Error).name === 'AbortError') throw new Error('Web fetch timed out after 10s');
    throw e;
  } finally {
    clearTimeout(timer);
  }

  if (!response.ok) {
    throw new Error(`HTTP ${response.status} ${response.statusText} for ${url}`);
  }

  // 3. Read body with size limit
  const contentType = response.headers.get('content-type') ?? 'text/plain';
  const reader = response.body!.getReader();
  const chunks: Uint8Array[] = [];
  let totalBytes = 0;

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    totalBytes += value.length;
    chunks.push(value);
    if (totalBytes > MAX_RESPONSE_BYTES) {
      await reader.cancel();
      break;
    }
  }

  const body = Buffer.concat(chunks);
  const truncated = totalBytes > MAX_RESPONSE_BYTES;

  // 4. Transform
  const { content, isBase64 } = await transformContent(body, contentType, url);

  // 5. Truncate for LLM context
  const finalContent = content.length > OUTPUT_MAX_CHARS
    ? content.slice(0, OUTPUT_MAX_CHARS) + '\n\n[Content truncated]'
    : content;

  return { content: finalContent, contentType, truncated };
}
```

---

## §6 — Web Search with Citation Grounding

### Search Tool

```typescript
interface SearchResult {
  title: string;
  url: string;
  snippet: string;
  rank: number;
}

// Grounding metadata: citation index → URL + title
interface GroundingCitation {
  index: number;    // [1], [2], etc.
  url: string;
  title: string;
}

// Insert citation markers into text at correct byte positions
// Note: positions are byte offsets, not char positions — use Buffer for accuracy
function insertCitations(text: string, markers: GroundingChunk[]): string {
  const buf = Buffer.from(text, 'utf-8');
  let result = '';
  let lastPos = 0;

  // Sort markers by start position descending to avoid offset drift
  const sorted = [...markers].sort((a, b) => b.startIndex - a.startIndex);

  for (const marker of sorted) {
    result = `[${marker.index}]${buf.slice(marker.endIndex).toString('utf-8')}${result}`;
    lastPos = marker.startIndex;
  }

  return buf.slice(0, lastPos).toString('utf-8') + result;
}

// Format citations as a references section
function formatCitationFooter(citations: GroundingCitation[]): string {
  if (citations.length === 0) return '';
  const lines = citations.map(c => `[${c.index}] ${c.title} — ${c.url}`);
  return '\n\n---\n**Sources:**\n' + lines.join('\n');
}
```

### Search Provider Abstraction

```typescript
interface SearchProvider {
  name: string;
  search(query: string, maxResults?: number): Promise<SearchResult[]>;
}

// Brave Search
class BraveSearchProvider implements SearchProvider {
  name = 'brave';
  constructor(private apiKey: string) {}

  async search(query: string, maxResults = 10): Promise<SearchResult[]> {
    const url = `https://api.search.brave.com/res/v1/web/search?q=${encodeURIComponent(query)}&count=${maxResults}`;
    const resp = await fetch(url, {
      headers: { 'Accept': 'application/json', 'X-Subscription-Token': this.apiKey },
    });
    const data = await resp.json();
    return (data.web?.results ?? []).map((r: BraveResult, i: number) => ({
      title: r.title,
      url: r.url,
      snippet: r.description,
      rank: i + 1,
    }));
  }
}

// Fallback chain: primary → secondary → empty
class CompositeSearchProvider implements SearchProvider {
  name = 'composite';
  constructor(private providers: SearchProvider[]) {}

  async search(query: string, maxResults = 10): Promise<SearchResult[]> {
    for (const provider of this.providers) {
      try {
        const results = await provider.search(query, maxResults);
        if (results.length > 0) return results;
      } catch (e) {
        // try next provider
      }
    }
    return [];
  }
}
```

---

## §7 — LRU Response Cache

```typescript
interface CacheEntry {
  content: string;
  contentType: string;
  fetchedAt: number;
}

const responseCache = new LRUCache<string, CacheEntry>({
  max: 200,
  ttl: 15 * 60 * 1000,  // 15 minutes
});

async function cachedFetch(input: WebFetchInput): Promise<WebFetchResult> {
  const cacheKey = `${input.method ?? 'GET'}:${input.url}`;
  const cached = responseCache.get(cacheKey);

  if (cached) {
    return {
      content: cached.content,
      contentType: cached.contentType,
      truncated: false,
    };
  }

  const result = await webFetch(input);
  responseCache.set(cacheKey, {
    content: result.content,
    contentType: result.contentType,
    fetchedAt: Date.now(),
  });

  return result;
}
```

---

## Integration Checklist

- [ ] Resolve hostname to IP and block private/loopback ranges (SSRF prevention)
- [ ] Rate-limit per hostname (10 req/60s default; use LRU window cache)
- [ ] Timeout fetch at 10s; cap response body at 2 MB
- [ ] Transform HTML → markdown; JSON → pretty-printed; binary → base64
- [ ] Normalize GitHub blob URLs → raw.githubusercontent.com
- [ ] Truncate content to 100 KB before injection into LLM context
- [ ] Insert citation markers at byte positions (not char positions) to avoid UTF-8 drift
- [ ] Add `---\nSources:` footer with numbered citations
- [ ] Cache responses for 15 minutes (LRU, max 200 entries)
- [ ] Block non-HTTP(S) protocols (file://, ftp://, data://)
- [ ] Use fallback search provider chain when primary is unavailable
