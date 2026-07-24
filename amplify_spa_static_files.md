# AWS Amplify — static files 404 on an SPA

Debugging a static file (`sitemap.xml`) that returned the app's "Not Found" page in
production, while a sibling file in the same folder (`robots.txt`) served fine.

Stack: **Vite SPA + AWS Amplify Hosting + CloudFront + S3 + Route 53**.

This is a debugging log, not a runbook. The value is in the wrong turns — the
symptom pointed at four different systems before the actual one-line cause showed up.

**The answer up front:** Amplify's default SPA rewrite rule excludes a hardcoded list
of file extensions from being rewritten to `index.html`. `xml` is **not** in that list.
The file was in S3, intact, the whole time — the routing layer never let a request reach it.

---

## 1. The symptom

`https://example.com/sitemap.xml` rendered the SPA's client-side "Not Found" screen.
`https://example.com/robots.txt` worked perfectly. Both files live in `public/` and are
copied to `dist/` by the same build step.

1.1. **The browser lies here.** It looked like a 404, but it was an HTTP **200** serving
`index.html`, which then booted the router, found no matching route, and rendered its own
NotFound component. Always check the status code before trusting the page.

```bash
curl -sI https://example.com/sitemap.xml
# HTTP/1.1 200 OK
# Content-Type: text/html      <-- should have been application/xml
# Content-Length: 1550         <-- that's the size of index.html
```

---

## 2. Narrowing it down with curl

The whole investigation was four headers. No dashboards needed for the diagnosis.

2.1. **Compare the broken path against the homepage.** This is the highest-signal test:

```bash
curl -sI https://example.com/           | grep -iE "etag|content-length|content-type"
curl -sI https://example.com/sitemap.xml | grep -iE "etag|content-length|content-type"
```

Identical `ETag`, identical `Content-Length`, both `text/html` → **the server is handing
you `index.html` for that path.** Same bytes, so it's literally the same object.

2.2. **Test a path that definitely doesn't exist:**

```bash
curl -sI https://example.com/definitely-not-real-abc123.txt
# HTTP/1.1 404 Not Found
```

A **real 404** here is the key discriminator. It proves there is no blanket
"404 → index.html" catch-all, which means something is *selectively* rewriting only
certain paths. That narrows it to a rules-based rewrite, not a missing file.

2.3. **Probe every static file in `public/` at once** to find the pattern:

```bash
for p in robots.txt sitemap.xml favicon.svg manifest.webmanifest sw.js; do
  printf "%-22s " "/$p"
  curl -s -o /dev/null -w "%{http_code}  %{content_type}\n" "https://example.com/$p"
done
```

Everything served correctly **except** the `.xml`. That is the moment the answer is
visible: an extension-specific failure means an extension-based rule, not file contents.

2.4. **Identify the actual infrastructure** — don't assume it matches your CI:

```bash
curl -sI https://example.com/ | grep -iE "^server|^via|^x-cache|^x-amz|^x-vercel"
```

| Header | Means |
|--------|-------|
| `Server: AmazonS3` | content is coming out of an S3 bucket |
| `Via: ...cloudfront.net` | CloudFront is in front of it |
| `Server: Vercel` + `x-vercel-id` | Vercel |
| `Server: nginx` / `Apache` | a VM, not object storage |

---

## 3. Finding which AWS resource actually serves the domain

3.1. **Route 53 is the source of truth**, not the CloudFront list. Hosted zones → the zone →
find the **A record for the apex** (blank/root name) and read its alias target.

3.2. **The distribution serving the site may not appear in your CloudFront console.**
Amplify creates and manages its own distribution, and it is *hidden* from the normal
CloudFront distributions list. Route 53 will happily alias to a `dxxxxxxxx.cloudfront.net`
that you cannot find anywhere in that console — that is not (necessarily) a
cross-account setup, it's just Amplify.

3.3. **Buckets named after the domain are usually decoys.** A bucket like
`www.example.com` holding **zero objects** is the classic `www` → apex redirect: an empty
bucket with a static-website redirect rule, fronted by its own distribution.

```bash
curl -sI https://www.example.com/anything
# HTTP/1.1 301 Moved Permanently
# Location: https://example.com/anything
```

3.4. Checking whether the deploy is Amplify: console → **Amplify** → check the correct
**region** (top-right). Apps are region-scoped and simply won't be listed if you're
looking at the wrong one.

---

## 4. The actual cause

Amplify Hosting → app → **Hosting → Rewrites and redirects**. The default SPA rule:

```json
[
  {
    "source": "</^[^.]+$|\\.(?!(css|gif|ico|jpg|js|png|txt|svg|woff|woff2|ttf|map|json|webmanifest)$)([^.]+$)/>",
    "status": "200",
    "target": "/index.html"
  }
]
```

4.1. **Read it in plain English:** *"rewrite every request to `/index.html`, EXCEPT paths
ending in one of these extensions."* It is a negative lookahead over a hardcoded allowlist.

4.2. `txt` is in the list → `robots.txt` is excluded from the rewrite → the real file is served.
`xml` is **not** in the list → `sitemap.xml` gets rewritten → you get `index.html`.

4.3. Also missing by default and worth adding if used: `xml`, `pdf`, `csv`, `mp4`, `avif`,
`wasm`, `webp`.

---

## 5. The fix

5.1. **Rewrites and redirects → Manage redirects → open the JSON editor**, add `|xml` to
the extension group:

```json
[
  {
    "source": "</^[^.]+$|\\.(?!(css|gif|ico|jpg|js|png|txt|svg|woff|woff2|ttf|map|json|webmanifest|xml)$)([^.]+$)/>",
    "status": "200",
    "target": "/index.html"
  }
]
```

Keep the `\\.` double backslash exactly as-is — it's JSON-escaped.

5.2. **Saving the rule is not enough — the CDN still serves the old response.**

5.3. **Amplify has no manual cache-invalidation button.** Redeploy to purge it:
app → branch → **Redeploy this version**.

5.4. Verify, and confirm SPA routing still works:

```bash
curl -sI https://example.com/sitemap.xml | grep -iE "content-type|content-length|x-cache"
# Content-Type: text/xml
# Content-Length: 973
# X-Cache: Miss from cloudfront   <-- fresh from origin, cache was purged

curl -s -o /dev/null -w "%{http_code}\n" https://example.com/some/client/route
# 200  <-- deep links still rewrite to index.html correctly
```

---

## Gotchas

- **A "Not Found" page is not proof of a 404.** Check the status code. An SPA rendering its
  own NotFound on an HTTP 200 is a completely different bug from a real 404.

- **Identical `ETag` between two URLs means identical bytes.** It tells you the *same object*
  is being served — but not *why*. I read it as "the S3 object got overwritten with the
  homepage" and went hunting for a broken upload script. It actually meant "a rewrite rule is
  serving you index.html." Same evidence, opposite conclusion, hours apart.

- **When one file type works and another doesn't, stop looking at file contents.** `.txt`
  serving while `.xml` fails can only be a rule that discriminates by extension. That single
  observation would have gone straight to the answer.

- **Verify which platform actually serves production before debugging it.** CI can be green on
  one provider while a completely different one serves the domain. `Server:` and `Via:` headers
  settle it in one request. I burned real time tuning config for a platform that wasn't involved.

- **A service worker can produce identical symptoms.** If the app is a PWA, `navigateFallback`
  serves the app shell for navigation requests to anything not precached. Rule it in or out
  early: `curl` ignores service workers, so if curl reproduces the problem, the SW is innocent.

- **`s-maxage=31536000` (1 year) is common on static hosting.** Once a wrong response is
  cached, it is pinned until you invalidate. Query-string cache-busting (`?v=123`) does **not**
  work when the distribution is configured to ignore query strings — identical `Age` values
  across different query strings is the tell.

- **Newly added static files are the ones that expose this class of bug.** An existing file
  keeps working because it was already handled; a new file with an unlisted extension is what
  finally trips the rule.
