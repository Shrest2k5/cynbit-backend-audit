# Cynbit Technologies: Backend Technical Audit (Passive Review)

Evidence labels used below:
- OBSERVED: directly visible in public responses
- INFERRED: reasonable conclusion from observed data, not confirmed
- TO VERIFY: needs a browser DevTools check (Appendix A)

## Deliverables map

| Required deliverable | Where |
|---|---|
| Backend Technical Audit Report (architecture, API structure, payloads) | Part 1 |
| Security & Performance Findings (categorized) | Part 2 |
| Recommended Improvements (technical recommendations and code) | Part 3 |

---

# PART 1: Backend Technical Audit Report

## 1. Executive summary

Cynbit's marketing site is a server-rendered Next.js application with a content layer (blog, case studies, careers, diaries) and three lead-capture forms (contact, product demo, newsletter). Nothing observable from public content shows a critical or exploitable flaw. The findings are mostly hygiene, SEO/performance and hardening gaps, plus architectural opportunities for the products the site describes (TenantEase, Revio, Cynbit Studio), which handle sensitive data (tenant ID documents, reviews, executed user code).

Top items:
1. The sitemap is generated per request with a fresh `lastmod` for every static page, and uses a different host (`cynbit.in`) than the served host (`www.cynbit.in`).
2. Every page declares the same `og:url` and a `logo.svg` as `og:image`, which most social crawlers do not render.
3. robots.txt publishes `/api/`, `/admin/`, `/portal/`, which maps internal surfaces. This is only safe if those paths are properly authenticated.
4. Lead forms show no visible consent text or anti-abuse signal in the rendered markup (bot protection, rate limiting and validation need verifying).
5. Icon-font ligature names leak into link text and the DOM; marquee content is duplicated five times.

---

## 2. Observed architecture

| Aspect | Evidence | Label |
|---|---|---|
| Framework | `next-size-adjust` meta tag present on all pages (emitted by Next.js) | OBSERVED / strong inference |
| Rendering | Full content present in the HTML response, so SSR/SSG, not client-only | OBSERVED |
| Routing | Clean paths: `/services`, `/products`, `/careers/<slug>`, `/blog/<slug>`, `/case-studies/<slug>`, `/diaries/<slug>` | OBSERVED |
| Content model | ~25 blog posts, 15 case studies, 20 careers, 8 diaries. Blog/careers/diaries have `lastmod` values only milliseconds apart (e.g. 09:19:43.825 to .945), which suggests a single seed or import run, likely from a DB or CMS | INFERRED |
| Hidden surfaces | robots.txt (see 2.1) disallows `/api/`, `/admin/`, `/portal/` | OBSERVED |
| Orphan page | Footer links to `/navigation`, which is absent from the sitemap | OBSERVED |
| Forms | Contact (name, email, company, industry, message); product demo (name, work email, company, message, product selector); newsletter (email) | OBSERVED |
| Third parties | Google-hosted icon font (Material Symbols style ligatures), social links. No analytics or form vendor visible in static markup | OBSERVED / TO VERIFY |
| Canonical host | Site served on `www.cynbit.in`; sitemap, robots sitemap line and `og:url` use `cynbit.in` | OBSERVED |

Sketch of the likely request flow:

```
Browser -> CDN/edge (TO VERIFY) -> Next.js server (SSR/ISR)
                                    |-- /api/* route handlers (forms, newsletter)  [disallowed in robots]
                                    |-- content source (DB/CMS/static data)
                                    |-- email/CRM/queue for leads (TO VERIFY)
```

### 2.1 API structure and payloads (observed)

No application API endpoint was requested or discovered during passive review, so no live request or response body from a Cynbit API is claimed here. The payload-like data that is publicly observable:

| Surface | Method / format | Observed structure |
|---|---|---|
| HTML documents | GET, `text/html; charset=utf-8` | Full SSR HTML with a per-page `<title>`, description, Open Graph and Twitter meta block |
| `/robots.txt` | GET, `text/plain` | `User-agent: *`, `Allow: /`, `Crawl-delay: 1`, `Sitemap: https://cynbit.in/sitemap.xml`, `Disallow: /api/`, `/admin/`, `/portal/` |
| `/sitemap.xml` | GET, `application/xml` | sitemaps.org `urlset`, ~110 `<url>` entries, each with `loc`, `lastmod` (ISO 8601 with ms), `changefreq`, `priority` (1.0 home, 0.8 core pages, 0.6 blog and case studies, 0.5 careers and diaries) |
| Contact form (client side) | Not observed on the wire | Fields: Full Name, Email Address, Company (optional), Industry (optional), Your Message |
| Product demo form (client side) | Not observed on the wire | Fields: Full Name, Work Email, Company Name, How can we help, product selector (Studio, TenantEase, Revio) |
| Newsletter form (client side) | Not observed on the wire | Single email input in the footer |

Likely API shape (INFERRED from the framework and robots.txt, not confirmed): JSON `POST` handlers under `/api/` for the three forms, returning a status and a message. The endpoint paths, request schema, status codes and error format must be read from the site's JavaScript bundle in the browser (Appendix A). Record them in the table below once confirmed:

| Endpoint | Method | Request body | Success response | Error response | Auth |
|---|---|---|---|---|---|
| (fill from DevTools) | | | | | |

---

# PART 2: Security & Performance Findings

Findings are grouped by category. Severity reflects visible evidence only.

## 3. Findings

### Security

| ID | Finding | Severity | Evidence | Label |
|---|---|---|---|---|
| S-1 | robots.txt enumerates `/api/`, `/admin/`, `/portal/`. Robots is not an access control; it advertises where privileged surfaces live. | Low / Info | robots.txt | OBSERVED |
| S-2 | Three lead forms with no visible consent or privacy notice at submit, and no visible bot challenge. Risk: spam, email bombing of the inbox via the form, and DPDP Act 2023 consent gaps for collected personal data. | Low-Medium | Rendered form markup | OBSERVED (UI) / TO VERIFY (server) |
| S-3 | Newsletter claims "25k+" subscribers. Without double opt-in, a public subscribe endpoint can be abused to subscribe third-party emails. | Low | Footer on every page | TO VERIFY |
| S-4 | Several contact emails and three phone numbers are published in plain text on every page (scraping and phishing fuel). | Info | Footer | OBSERVED |
| S-5 | Security headers (CSP, HSTS, X-Frame-Options/frame-ancestors, Referrer-Policy, Permissions-Policy), cookie flags and CORS policy are not visible via page content. | Unknown | n/a | TO VERIFY |

### Performance / SEO

| ID | Finding | Severity | Evidence | Label |
|---|---|---|---|---|
| P-1 | `sitemap.xml` sets `lastmod` of all static pages to the generation timestamp (identical, to the millisecond). Crawlers learn to ignore `lastmod`, and the route is likely dynamic and uncached. | Low | sitemap.xml | OBSERVED |
| P-2 | Host mismatch: pages served on `www`, sitemap/og/robots reference apex. Needs one canonical host with a 301 and matching `rel=canonical`. | Low-Medium | sitemap, meta tags | OBSERVED |
| P-3 | `og:url` is `https://cynbit.in` on every page (home, contact, products, careers), so shared links all canonicalize to the homepage. | Low | meta tags on 4 pages | OBSERVED |
| P-4 | `og:image` and `twitter:image` are `logo.svg`. Facebook, LinkedIn and X do not render SVG previews. | Low | meta tags | OBSERVED |
| P-5 | Careers page reuses the homepage `meta-keywords`; keywords meta is ignored by search engines anyway. | Info | careers meta | OBSERVED |
| P-6 | Client-logo marquee repeats its content five times in the DOM; nav is rendered twice (desktop and mobile). Extra DOM weight and duplicate content for screen readers. | Low | Rendered text | OBSERVED |
| P-7 | Icon ligature names ("history_edu Our Story", "expand_more") are part of link text. Before the font loads, or for text-only crawlers and screen readers, users see raw words. Also adds an external font request on the critical path. | Low | Rendered text | OBSERVED |
| P-8 | Stale `lastmod` on some case studies (2024-08) next to freshly seeded ones, while the page header advertises 2026 content. Content freshness signals are inconsistent. | Info | sitemap | OBSERVED |

### Data handling / validation / auth (architectural opportunities)

| ID | Finding | Severity | Label |
|---|---|---|---|
| D-1 | Marketing claims ("Efficiency 95%", "99.9% uptime", "98% retention", "1000+ certified") appear to be hardcoded. A claim tied to a live status endpoint is more defensible. | Info | INFERRED |
| D-2 | TenantEase stores tenant ID verification data. Needs encryption at rest, signed-URL object storage, strict tenant isolation, audit logging. | Design | Product claim on /products |
| D-3 | Revio promises role hierarchy (Admin / Manager / Advisor) and auto-posted AI replies to public reviews. Needs server-side RBAC checks per resource and a human approval gate, since review text is untrusted input to an LLM (prompt injection). | Design | Product claim |
| D-4 | Cynbit Studio evaluates user code ("Execution", "AI Code Evaluation"). Needs an isolated sandbox (no network, resource limits, short TTL) and a queue so execution never blocks the API. | Design | Product claim |
| D-5 | Careers has no visible application form; "Drop a portfolio" routes to the generic contact form. Applicant PII (and possibly resumes) would share an unscoped lead pipeline. | Low | OBSERVED |

### Data validation and error handling (summary of the above)

- Validation: no server-side schema, length limits, consent flag or honeypot is confirmable (S-2, S-3, D-5). Fix in 4.1.
- Error handling: the error format for failed submissions is unknown (TO VERIFY). Recommend a consistent `application/problem+json` response with 415/422/429 codes (4.1).
- Authentication: no login or protected area is exposed on the marketing site. `/admin/` and `/portal/` exist per robots.txt and were deliberately not probed. Product authorization is covered in D-2, D-3.

---

# PART 3: Recommended Improvements

## 4. Recommended improvements (with code)

### 4.1 Hardened lead-form endpoint (S-2, S-3, validation, error handling)

Schema validation, honeypot, bot check, rate limit, RFC 9457-style errors, and async hand-off so the request never waits on email/CRM.

```ts
// app/api/contact/route.ts
import { NextResponse } from "next/server";
import { z } from "zod";
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(5, "10 m"),
  prefix: "rl:contact",
});

const ContactSchema = z.object({
  fullName: z.string().trim().min(2).max(100),
  email: z.string().trim().toLowerCase().email().max(254),
  company: z.string().trim().max(120).optional(),
  industry: z.string().trim().max(80).optional(),
  message: z.string().trim().min(10).max(2000),
  consent: z.literal(true),                 // DPDP-style explicit consent
  website: z.string().max(0).optional(),    // honeypot: must stay empty
  turnstileToken: z.string().min(1),
});

const problem = (status: number, title: string, extra = {}) =>
  NextResponse.json({ type: "about:blank", title, status, ...extra }, {
    status,
    headers: { "Content-Type": "application/problem+json" },
  });

export async function POST(req: Request) {
  if (!req.headers.get("content-type")?.includes("application/json"))
    return problem(415, "Unsupported media type");

  const ip = req.headers.get("x-forwarded-for")?.split(",")[0]?.trim() ?? "unknown";
  const { success, reset } = await ratelimit.limit(ip);
  if (!success)
    return problem(429, "Too many requests", {
      retryAfter: Math.ceil((reset - Date.now()) / 1000),
    });

  const parsed = ContactSchema.safeParse(await req.json().catch(() => null));
  if (!parsed.success)
    return problem(422, "Validation failed", { errors: parsed.error.flatten().fieldErrors });

  const data = parsed.data;
  if (data.website) return NextResponse.json({ ok: true }); // silently drop bots

  const verify = await fetch("https://challenges.cloudflare.com/turnstile/v0/siteverify", {
    method: "POST",
    body: new URLSearchParams({
      secret: process.env.TURNSTILE_SECRET!,
      response: data.turnstileToken,
      remoteip: ip,
    }),
  }).then((r) => r.json());
  if (!verify.success) return problem(400, "Bot check failed");

  // Hand off to a queue; never send email/CRM calls inline.
  await enqueueLead({ ...data, receivedAt: new Date().toISOString() });
  return NextResponse.json({ ok: true }, { status: 202 });
}
```

Newsletter: add double opt-in (store `pending`, email a signed, expiring token, mark `confirmed` on click).

### 4.2 Security headers (S-5)

```js
// next.config.js
const csp = [
  "default-src 'self'",
  "script-src 'self' 'unsafe-inline' https://challenges.cloudflare.com", // replace unsafe-inline with nonces
  "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
  "font-src 'self' https://fonts.gstatic.com",
  "img-src 'self' data: https:",
  "frame-src https://challenges.cloudflare.com",
  "frame-ancestors 'none'",
  "base-uri 'self'",
  "form-action 'self'",
].join("; ");

module.exports = {
  async headers() {
    return [{
      source: "/(.*)",
      headers: [
        { key: "Content-Security-Policy", value: csp },
        { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" },
        { key: "X-Content-Type-Options", value: "nosniff" },
        { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
        { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
      ],
    }];
  },
  async redirects() {
    return [{
      source: "/:path*",
      has: [{ type: "host", value: "cynbit.in" }],
      destination: "https://www.cynbit.in/:path*",   // pick one canonical host
      permanent: true,
    }];
  },
};
```

### 4.3 Truthful, cacheable sitemap (P-1, P-2, P-8)

```ts
// app/sitemap.ts
import type { MetadataRoute } from "next";
import { getAllPosts, getAllCaseStudies } from "@/lib/content";

export const revalidate = 3600; // cache for an hour instead of regenerating per request
const BASE = "https://www.cynbit.in";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const [posts, cases] = await Promise.all([getAllPosts(), getAllCaseStudies()]);
  const staticPages = ["", "/story", "/team", "/services", "/products", "/contact"]
    .map((p) => ({ url: `${BASE}${p}`, lastModified: BUILD_TIME })); // fixed at build, not new Date()
  return [
    ...staticPages,
    ...posts.map((p) => ({ url: `${BASE}/blog/${p.slug}`, lastModified: p.updatedAt })),
    ...cases.map((c) => ({ url: `${BASE}/case-studies/${c.slug}`, lastModified: c.updatedAt })),
  ];
}
```

### 4.4 Per-page canonical and social metadata (P-3, P-4, P-5)

```ts
// app/blog/[slug]/page.tsx
export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await getPost(params.slug);
  const url = `https://www.cynbit.in/blog/${post.slug}`;
  return {
    title: post.title,
    description: post.excerpt,
    alternates: { canonical: url },
    openGraph: { url, title: post.title, images: [{ url: `${url}/opengraph-image`, width: 1200, height: 630 }] },
    twitter: { card: "summary_large_image" },
  };
}
```

Use a 1200x630 PNG/JPG (Next's `opengraph-image.tsx` can generate one), not an SVG.

### 4.5 Icons and DOM weight (P-6, P-7)

Replace ligature icon fonts with inline SVG (or `lucide-react`), and mark decorative icons hidden:

```tsx
<Link href="/story"><HistoryIcon aria-hidden="true" /> Our Story</Link>
```

Render the marquee list once and duplicate with CSS (or `aria-hidden` on the clone), and render one nav with responsive CSS instead of two.

### 4.6 Multi-tenant isolation for TenantEase / Revio (D-2, D-3)

Enforce tenancy in the database, not only in application code:

```sql
ALTER TABLE tenants_profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY org_isolation ON tenants_profiles
  USING (org_id = current_setting('app.current_org')::uuid);

-- per request, after authentication:
-- SET LOCAL app.current_org = '<org uuid from verified JWT claim>';
```

Store ID documents in private object storage; serve via short-lived signed URLs; encrypt sensitive columns (e.g. envelope encryption with a KMS key); write an append-only audit log for every read of an ID document.

RBAC check on every resource handler (never trust a role sent by the client):

```ts
function authorize(user: User, action: "reply:approve" | "reply:draft", loc: Location) {
  const allowed = {
    admin: ["reply:approve", "reply:draft"],
    manager: ["reply:approve", "reply:draft"],
    advisor: ["reply:draft"],
  }[user.role] ?? [];
  if (!allowed.includes(action) || !user.locationIds.includes(loc.id)) throw new ForbiddenError();
}
```

Revio AI replies: treat review text as untrusted. Wrap it in a delimited data block, force structured JSON output, run a policy check, and require approval before auto-posting negative or high-risk reviews (crisis-detection items always go to a human).

### 4.7 Code execution sandbox for Cynbit Studio (D-4)

```
API -> queue (BullMQ/SQS) -> worker -> ephemeral sandbox container
docker run --rm --network=none --read-only --cpus=0.5 --memory=128m \
  --pids-limit=64 --cap-drop=ALL --security-opt=no-new-privileges \
  --user 65534 runner-image
# hard wall-clock timeout (e.g. 5s), output size cap, per-user rate limit
```

The API returns a job id immediately; the client polls or subscribes (SSE) for the result, so slow submissions never tie up web workers.

### 4.8 Edge and caching posture (performance)

- Cache HTML at the CDN with `s-maxage` plus `stale-while-revalidate`; use ISR for blog/case-study pages and on-demand revalidation when content changes.
- Serve static assets with `Cache-Control: public, max-age=31536000, immutable` (hashed filenames).
- Self-host the icon/font assets and preload only the weights actually used.
- Provide a real `/api/health` (not disclosed in robots) and a status page to back the uptime claims.

### 4.9 robots.txt (S-1)

Keep `/api/`, `/admin/`, `/portal/` protected by authentication, put admin behind SSO/IP allowlist or a separate subdomain, and send `X-Robots-Tag: noindex` from those routes. Do not rely on robots.txt to hide anything.

---

## 5. Prioritized roadmap

1. Week 1: canonical host redirect, per-page `og:url`, PNG og image, real sitemap `lastmod`, security headers, Turnstile and rate limit on forms.
2. Week 2: double opt-in newsletter, consent checkbox, dedicated careers application endpoint with file validation (type, size, AV scan).
3. Week 3: replace icon font, de-duplicate DOM, CDN caching/ISR.
4. Ongoing (product): RLS multi-tenancy, RBAC tests, audit logging, sandboxed execution, human-in-the-loop for AI replies.

---

## Appendix A. Items to confirm yourself in the browser (passive)

Use DevTools while browsing normally. Do not touch `/api/`, `/admin/`, `/portal/`, and do not submit junk or repeated form data.

| Check | How | Result |
|---|---|---|
| Response headers on `/` | DevTools > Network > document > Headers (or securityheaders.com) | |
| TLS and HSTS | SSL Labs or browser padlock details | |
| `www` vs apex redirect | Open `http://cynbit.in` and `https://cynbit.in`, note the final URL | |
| Cache headers on HTML and static assets | Network tab: `Cache-Control`, `Age`, `x-vercel-cache`/CDN headers | |
| Server/framework disclosure | `Server`, `X-Powered-By` headers | |
| Form endpoint and payload shape | Sources tab: search JS bundles for `fetch(` / `/api/` to see the URL and body schema; do not send test traffic | |
| Client-side validation vs server | Read the bundle's validation rules (e.g. zod schema) | |
| Cookies and flags | Application tab: `Secure`, `HttpOnly`, `SameSite` | |
| Third-party requests | Network tab filtered by third-party domain | |
| Lighthouse (Performance, SEO, Best Practices) | DevTools > Lighthouse, mobile preset | |
| Core Web Vitals / payload size | Lighthouse + Network tab totals | |

Add the measured values in the Result column before submitting; they turn the TO VERIFY items into evidence-backed findings.

---

## Appendix B. Scope and limitations

This review used only publicly retrievable pages and files. Response headers, network payloads, cookies and TLS configuration were not observable through this method and are listed in Appendix A for confirmation. No vulnerability was exploited and no non-public resource was accessed. Severity ratings are conservative estimates based on visible evidence only.
