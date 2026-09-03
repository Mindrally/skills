---
name: netlify-development
description: "Best practices for building and deploying Netlify sites, covering serverless and edge functions, Netlify Blobs storage, the Image CDN, and build configuration. Use when writing Netlify Functions or Edge Functions, configuring netlify.toml or redirects/headers, managing environment variables across deploy contexts, working with Netlify Blobs, or setting up local development and deploy previews."
---

# Netlify Development Best Practices

## Overview

This skill provides comprehensive guidelines for building and deploying projects on Netlify, covering serverless functions, edge functions, background functions, scheduled functions, Netlify Blobs, Image CDN, and deployment configuration.

## Core Principles

- Use in-code configuration via exported `config` objects (preferred over netlify.toml)
- Never add version numbers to imported Netlify packages (use `@netlify/functions`, never `@netlify/functions@1.2.3`)
- Only add CORS headers when explicitly required
- Leverage appropriate function types for different use cases
- Use Netlify Blobs for state and data storage
- The `.netlify` folder is generated tooling output, not user code — keep it in `.gitignore`
- Never put serverless or edge functions inside the publish/public directory
- Don't change the default functions or edge-functions directory unless explicitly asked to

## Function Types Overview

| Type | Use Case | Timeout | Path Convention |
|------|----------|---------|-----------------|
| Serverless | Standard API endpoints | 10s (26s Pro) | `/.netlify/functions/name` |
| Edge | Request/response modification | 50ms CPU | Custom paths |
| Background | Long-running async tasks | 15 minutes | `-background` suffix |
| Scheduled | Cron-based tasks | 30s | Configured schedule |

Function and edge-function source directories default to `netlify/functions` and `netlify/edge-functions`, but can be overridden in `netlify.toml`:
```toml
[functions]
  directory = "my_functions"

[build]
  edge_functions = "my-edge-functions"
```
`netlify.toml` settings override UI build-settings configuration.

## Serverless Functions

### Basic Structure
```typescript
// netlify/functions/hello.mts
import type { Context } from '@netlify/functions';

export default async (request: Request, context: Context) => {
  try {
    // Validate request
    if (request.method !== 'POST') {
      return new Response('Method Not Allowed', { status: 405 });
    }

    const body = await request.json();

    // Business logic
    const result = await processData(body);

    return Response.json(result);
  } catch (error) {
    console.error('Function error:', error);
    return Response.json({ error: 'Internal Server Error' }, { status: 500 });
  }
};

export const config = {
  path: '/api/hello',
};
```

### Configuration Options
```typescript
export const config = {
  // Custom path (instead of /.netlify/functions/name)
  path: '/api/users',

  // HTTP methods (optional, allows all by default)
  method: ['GET', 'POST'],

  // Rate limiting
  rateLimit: {
    windowSize: 60,
    windowLimit: 100,
  },
};
```

### Path Conventions
- Default path: `/.netlify/functions/{function_name}`
- Custom paths via config completely replace the default
- Use custom paths for cleaner API URLs

### The Context Argument

Both serverless and edge functions receive a `context` object as their second argument:

```typescript
{
  account: { id: string };                 // Netlify team account ID
  cookies: {
    get: (name: string) => string | undefined;
    set: (options: { name: string; value: string; path?: string; domain?: string; secure?: boolean; httpOnly?: boolean; expires?: Date }) => void;
    delete: (nameOrOptions: string | { name: string; path?: string; domain?: string }) => void;
  };
  deploy: { context: string; id: string; published: boolean };
  geo: {
    city: string;
    country: { code: string; name: string };
    latitude: number;
    longitude: number;
    subdivision: { code: string; name: string };
    timezone: string;
    postalCode: string;
    ip: string;
  };
  params: Record<string, string>;          // route params from the path config
  requestId: string;
  server: { region: string };
  site: { id: string; name: string; url: string };
}
```

### The `Netlify` Global Object

Available in global scope on all serverless and edge functions — prefer `Netlify.env.*` over `process.env` for reading/writing environment variables at runtime:

```typescript
{
  context: object | null; // same as the function's second arg; null outside a handler
  env: {
    get: (name: string) => string | undefined;
    has: (name: string) => boolean;
    set: (name: string, value: string) => void;
    delete: (name: string) => void;
    toObject: () => Record<string, string>;
  };
}
```

## Edge Functions

### Use Cases
- Modify requests before they reach the origin
- Modify responses before returning to users
- Geolocation-based personalization
- A/B testing
- Authentication at the edge

### Implementation
```typescript
// netlify/edge-functions/geo-redirect.ts
import type { Context } from '@netlify/edge-functions';

export default async (request: Request, context: Context) => {
  const country = context.geo.country?.code || 'US';

  // Redirect based on country
  if (country === 'DE') {
    return Response.redirect(new URL('/de', request.url));
  }

  // Continue to origin
  return context.next();
};

export const config = {
  path: '/*',
  excludedPath: ['/api/*', '/_next/*'],
};
```

### Response Modification
```typescript
export default async (request: Request, context: Context) => {
  // Get response from origin
  const response = await context.next();

  // Modify headers
  response.headers.set('X-Custom-Header', 'value');

  // Transform HTML
  const html = await response.text();
  const modifiedHtml = html.replace('</body>', '<script>...</script></body>');

  return new Response(modifiedHtml, {
    status: response.status,
    headers: response.headers,
  });
};
```

### Runtime and Module Support

Edge functions run on Deno, not Node.js, and can pull in code from several ecosystems:
- Node.js built-ins: `import { randomBytes } from 'node:crypto'`
- Deno modules: URL imports, e.g. `import React from 'https://esm.sh/react'`
- npm packages (beta): install via `npm install` and import by name — packages with native binaries (e.g., Prisma) or dynamic requires may not work
- Import maps let you reference third-party modules by short name instead of full URL; declare the map file path via `[functions] deno_import_map = "./import_map.json"` in `netlify.toml`

Available Web APIs are more limited than Node.js: `fetch`/`Request`/`Response`/`URL`, `TextEncoder`/`TextDecoder` (and stream variants), Web Crypto (`randomUUID`, `getRandomValues`, `SubtleCrypto`), `WebSocket`, `setTimeout`/`setInterval`, the Streams API, and `URLPattern`.

### Extra Config Options

Beyond `path` and `excludedPath`, edge function config supports:
```typescript
{
  pattern?: RegExp | RegExp[];        // regex alternative to `path`
  excludedPattern?: RegExp | RegExp[];
  method?: string | string[];
  onError?: 'continue' | 'fail' | 'fallback';
  cache?: 'manual';                   // opt in to edge response caching
}
```

### Ordering Multiple Edge Functions

When several edge functions can match the same path, declare them in `netlify.toml` for explicit top-to-bottom ordering (cached functions always run last regardless of position):
```toml
[[edge_functions]]
  path = "/admin"
  function = "auth"

[[edge_functions]]
  path = "/admin"
  function = "injector"
  cache = "manual"
```
Execution order: config-based functions run before framework-generated ones, which run before user-defined inline functions; non-cached functions run before cached ones; multiple inline functions run alphabetically by filename. Use `context.next()` to continue the chain rather than re-triggering functions with `fetch()`/`URL()` (which starts a new request chain).

### Limits and Incompatibilities

- 20 MB compressed code size, 512 MB memory per deployment, 50ms CPU time per request, 40s response-header timeout
- Not compatible with split testing, `_headers`/`netlify.toml` custom headers, or prerendering on paths an edge function serves
- Can only rewrite requests to same-site URLs — use `fetch()` for external content
- Not included in Netlify's HIPAA-compliant hosting offering

## Background Functions

### Key Characteristics
- 15-minute timeout (wall clock time)
- Immediately return 202 status code
- Return values are ignored
- Must have `-background` suffix

### Implementation
```typescript
// netlify/functions/process-video-background.mts
import type { Context } from '@netlify/functions';
import { getStore } from '@netlify/blobs';

export default async (request: Request, context: Context) => {
  const { videoId } = await request.json();

  // Long-running processing
  const result = await processVideo(videoId);

  // Store result for later retrieval
  const store = getStore('processed-videos');
  await store.setJSON(videoId, result);

  // Return value is ignored
  return new Response('Processing complete');
};

export const config = {
  path: '/api/process-video',
};
```

### Retrieving Background Results
```typescript
// netlify/functions/get-video-status.mts
import { getStore } from '@netlify/blobs';

export default async (request: Request, context: Context) => {
  const url = new URL(request.url);
  const videoId = url.searchParams.get('id');

  const store = getStore('processed-videos');
  const result = await store.get(videoId, { type: 'json' });

  if (!result) {
    return Response.json({ status: 'processing' });
  }

  return Response.json({ status: 'complete', data: result });
};
```

## Scheduled Functions

### Key Characteristics
- CRON expressions are evaluated in UTC, with a 1-minute minimum interval
- 30-second execution limit; the function does not return a response body (the return value is ignored)
- The request body is a JSON-encoded object with a `next_run` field — the ISO-8601 timestamp of the next scheduled invocation
- Scheduled functions only run on published deploys — they do **not** run on deploy previews or branch deploys
- Test locally by running `netlify dev` alongside `netlify functions:invoke <name>`
- Schedules can also live in `netlify.toml` for consistency with other config, but in-code `config.schedule` is preferred:
  ```toml
  [functions."daily-cleanup"]
    schedule = "@daily"
  ```

### Configuration
```typescript
// netlify/functions/daily-cleanup.mts
import type { Context } from '@netlify/functions';

export default async (request: Request, context: Context) => {
  console.log('Running daily cleanup...');

  // Cleanup logic
  await cleanupOldRecords();

  return new Response('Cleanup complete');
};

export const config = {
  schedule: '@daily', // or '0 0 * * *' for midnight UTC
};
```

### Schedule Patterns
```typescript
// Common patterns
export const config = {
  schedule: '@hourly',     // Every hour
  schedule: '@daily',      // Every day at midnight
  schedule: '@weekly',     // Every week
  schedule: '*/15 * * * *', // Every 15 minutes
  schedule: '0 9 * * 1-5',  // 9 AM on weekdays
};
```

## Netlify Blobs

Prefer Blobs over standing up a new database unless the data needs relational structure or search — Blobs require no provisioning and share the same API across all compute types. Limits: store names ≤64 bytes, object keys ≤600 bytes, max object size 5GB.

### Consistency and Storage Scopes

- By default, stores are **eventually consistent** — fast reads, with writes propagating within about 60 seconds. Pass `{ consistency: 'strong' }` to `getStore` for immediate read-after-write visibility at the cost of slower reads.
- There's no built-in concurrency control — the last write wins. Add your own locking if concurrent writers must not clobber each other.
- Deploy-specific stores (`getDeployStore`) sync with a deploy and are removed when that deploy is deleted; global stores (`getStore`) persist across all branches and are never auto-cleaned. Build plugins and file-based uploads must write to deploy-specific stores.
- When writing to a global store, guard against non-production data leaking in by branching on `context.deploy.context`:
  ```typescript
  function getBlobStore(...args: Parameters<typeof getStore>) {
    return context.deploy.context === 'production'
      ? getStore(...args)
      : getDeployStore(...args);
  }
  ```

### Basic Usage
```typescript
import { getStore } from '@netlify/blobs';

// Get a store
const store = getStore('my-store');

// Store data
await store.set('key', 'string value');
await store.setJSON('json-key', { foo: 'bar' });

// Retrieve data
const value = await store.get('key');
const jsonValue = await store.get('json-key', { type: 'json' });

// Delete data
await store.delete('key');

// List keys
const { blobs } = await store.list();
```

### Binary Data
```typescript
import { getStore } from '@netlify/blobs';

const store = getStore('files');

// Store binary data
const arrayBuffer = await file.arrayBuffer();
await store.set('uploads/file.pdf', arrayBuffer, {
  metadata: { contentType: 'application/pdf' },
});

// Retrieve binary data
const blob = await store.get('uploads/file.pdf', { type: 'blob' });
```

### Deploy-specific vs Site-wide
```typescript
// Site-wide store (persists across deploys)
const siteStore = getStore({
  name: 'user-data',
  siteID: context.site.id,
});

// Deploy-specific store (scoped to deployment)
const deployStore = getStore({
  name: 'cache',
  deployID: context.deploy.id,
});
```

## Netlify Image CDN

### Usage
```html
<!-- Basic optimization -->
<img src="/.netlify/images?url=/images/hero.jpg&w=800&q=80" alt="Hero">

<!-- With fit and format -->
<img src="/.netlify/images?url=/images/hero.jpg&w=400&h=300&fit=cover&fm=webp" alt="Hero">
```

### Parameters
- `url`: Source image path (required)
- `w`: Width in pixels
- `h`: Height in pixels
- `q`: Quality (1-100)
- `fit`: cover, contain, fill
- `fm`: Format (webp, avif, auto)

### Programmatic Usage
```typescript
function getOptimizedImageUrl(src: string, options: ImageOptions) {
  const params = new URLSearchParams({
    url: src,
    w: String(options.width),
    q: String(options.quality || 80),
    fm: 'auto',
  });

  return `/.netlify/images?${params}`;
}
```

### Remote Images, Custom Routes, and Caching

- External image domains must be allowlisted before they can be transformed, and only need absolute URLs:
  ```toml
  [images]
    remote_images = ["https://externalexample.com/.*"]
  ```
- URI-encode the `url` parameter value when it points at a remote image.
- To expose transformations at a friendlier URL than `/.netlify/images`, define a redirect (params can stay dynamic or be pinned):
  ```toml
  [[redirects]]
    from = "/transform-my-images/*"
    to = "/.netlify/images?url=/:splat&w=50&h=50"
    status = 200
  ```
- Custom headers (e.g., `Cache-Control`) can only be applied to images hosted on the same domain, via `[[headers]]`/`_headers`, and only when explicitly requested.
- Transformed and source images are cached at the edge; a new deploy invalidates the cache so images are reprocessed if the source changed.
- Framework integrations pick up Image CDN automatically or with minimal config: Angular's `NgOptimizedImage`, Astro's `<Image />`, Nuxt's `nuxt/image`, Next.js via `remotePatterns` in `next.config.js`, and Gatsby via `NETLIFY_IMAGE_CDN=true`.

## Environment Variables

### Access in Functions
```typescript
export default async (request: Request, context: Context) => {
  // Access environment variables
  const apiKey = Netlify.env.get('API_KEY');
  const dbUrl = process.env.DATABASE_URL;

  if (!apiKey) {
    console.error('API_KEY not configured');
    return Response.json({ error: 'Configuration error' }, { status: 500 });
  }

  // Use variables
};
```

### Context Variables
```typescript
export default async (request: Request, context: Context) => {
  // Available context
  const { site, deploy, geo, ip, requestId } = context;

  console.log('Site ID:', site.id);
  console.log('Deploy ID:', deploy.id);
  console.log('Country:', geo.country?.code);
  console.log('Request ID:', requestId);
};
```

### Managing Variables

Variables can be set via the Netlify UI (*Site configuration > Environment variables*), the CLI, the API, or `netlify.toml` — precedence is `netlify.toml` > UI/CLI/API, and site-specific variables win over shared ones. The CLI requires the site to be linked first (`netlify link`):
```bash
netlify env:set API_KEY "value"
netlify env:set API_KEY "secret-value" --secret
netlify env:unset API_KEY
netlify env:import .env
netlify env:list --plain --context production
```
Netlify builds don't read `.env` files directly — import them with `env:import` instead. Only export a production-context `.env.local` after confirming it's gitignored, and never commit it.

`netlify.toml` can scope variables per deploy context (`production`, `deploy-preview`, `branch-deploy`, `dev`, or a specific branch name):
```toml
[context.production.environment]
  NODE_VERSION = "20"

[context.deploy-preview.environment]
  NOT_PRIVATE_ITEM = "not so secret"
```
Avoid committing sensitive values this way — prefer the UI or CLI for secrets.

## Build Configuration

### netlify.toml
```toml
[build]
  command = "npm run build"
  publish = "dist"
  functions = "netlify/functions"

[build.environment]
  NODE_VERSION = "20"

[[redirects]]
  from = "/api/*"
  to = "/.netlify/functions/:splat"
  status = 200

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"

[functions]
  node_bundler = "esbuild"

[dev]
  command = "npm run dev"
  port = 3000
  targetPort = 5173
```

## File-based Uploads

### Direct Upload to Functions
```typescript
// netlify/functions/upload.mts
import { getStore } from '@netlify/blobs';

export default async (request: Request, context: Context) => {
  const formData = await request.formData();
  const file = formData.get('file') as File;

  if (!file) {
    return Response.json({ error: 'No file provided' }, { status: 400 });
  }

  const store = getStore('uploads');
  const key = `${Date.now()}-${file.name}`;

  await store.set(key, await file.arrayBuffer(), {
    metadata: {
      contentType: file.type,
      originalName: file.name,
    },
  });

  return Response.json({ key, message: 'Upload successful' });
};
```

## Site Management

Check whether a site is already linked by looking for `.netlify/state.json` with a populated `siteId` — if it's missing, `netlify init` walks through connecting the project to Netlify (including setting up the repo if needed); if the site already exists on Netlify, `netlify link` just prompts for credentials to connect the local folder to it.

### Creating and Linking Sites
```bash
# Initialize new site
netlify init

# Link existing site
netlify link

# Deploy manually
netlify deploy

# Deploy to production
netlify deploy --prod
```

## Local Development

### Netlify Dev
```bash
# Start local development server
netlify dev

# With specific port
netlify dev --port 8888

# With live reload
netlify dev --live
```

### Testing Functions Locally
```bash
# Invoke function directly
netlify functions:invoke hello --payload '{"name": "World"}'

# Serve functions only
netlify functions:serve
```

## Error Handling Best Practices

### Structured Error Responses
```typescript
interface ErrorResponse {
  error: string;
  code: string;
  details?: unknown;
}

function errorResponse(status: number, error: ErrorResponse): Response {
  return Response.json(error, { status });
}

export default async (request: Request, context: Context) => {
  try {
    // Validation
    const body = await request.json();
    if (!body.email) {
      return errorResponse(400, {
        error: 'Email is required',
        code: 'MISSING_EMAIL',
      });
    }

    // Business logic
    const result = await processRequest(body);
    return Response.json(result);

  } catch (error) {
    console.error('Function error:', error);
    return errorResponse(500, {
      error: 'Internal server error',
      code: 'INTERNAL_ERROR',
    });
  }
};
```

## Security Guidelines

### Input Validation
```typescript
import { z } from 'zod';

const RequestSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
});

export default async (request: Request, context: Context) => {
  const body = await request.json();

  const result = RequestSchema.safeParse(body);
  if (!result.success) {
    return Response.json(
      { error: 'Validation failed', details: result.error.issues },
      { status: 400 }
    );
  }

  // Use validated data
  const { email, name } = result.data;
};
```

### Authentication
```typescript
async function verifyToken(request: Request): Promise<User | null> {
  const auth = request.headers.get('Authorization');
  if (!auth?.startsWith('Bearer ')) {
    return null;
  }

  const token = auth.slice(7);
  // Verify token logic
  return verifyJWT(token);
}

export default async (request: Request, context: Context) => {
  const user = await verifyToken(request);
  if (!user) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Authenticated request handling
};
```

## Common Pitfalls to Avoid

1. Adding version numbers to `@netlify/functions` imports
2. Adding CORS headers when not explicitly needed
3. Using wrong function type for the use case
4. Forgetting `-background` suffix for background functions
5. Not using Blobs for persistent storage in background functions
6. Ignoring the 15-minute timeout for background functions
7. Not validating input in serverless functions
8. Hardcoding environment variables
9. Not handling errors appropriately at the edge
10. Using serverless functions for tasks better suited to edge functions
11. Committing the `.netlify` directory instead of gitignoring it
12. Adding redirects or custom headers to `netlify.toml`/`_redirects`/`_headers` on a new site unless the user asked for them
13. Writing non-production data into a global-scope Blobs store instead of a deploy-specific one
