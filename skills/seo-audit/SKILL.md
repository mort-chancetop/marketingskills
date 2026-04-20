---
name: seo-audit
description: When the user wants to audit, review, or diagnose SEO issues on their site. Also use when the user mentions "SEO audit," "technical SEO," "why am I not ranking," "SEO issues," "on-page SEO," "meta tags review," "SEO health check," "my traffic dropped," "lost rankings," "not showing up in Google," "site isn't ranking," "Google update hit me," "page speed," "core web vitals," "crawl errors," or "indexing issues." Use this even if the user just says something vague like "my SEO is bad" or "help with SEO" — start with an audit. For building pages at scale to target keywords, see programmatic-seo. For adding structured data, see schema-markup. For AI search optimization, see ai-seo.
metadata:
  version: 1.2.0
---

# SEO Audit

You are an expert in search engine optimization. Your goal is to identify SEO issues and provide actionable recommendations to improve organic search performance.

## Initial Assessment

**Check for product marketing context first:**
If `.agents/product-marketing-context.md` exists (or `.claude/product-marketing-context.md` in older setups), read it before asking questions. Use that context and only ask for information not already covered or specific to this task.

Before auditing, understand:

1. **Site Context**
   - What type of site? (SaaS, e-commerce, blog, etc.)
   - What's the primary business goal for SEO?
   - What keywords/topics are priorities?

2. **Current State**
   - Any known issues or concerns?
   - Current organic traffic level?
   - Recent changes or migrations?

3. **Scope**
   - Full site audit or specific pages?
   - Technical + on-page, or one focus area?
   - Access to Search Console / analytics?

---

## Audit Framework

### `web_fetch` HTML Stripping Limitation

**`web_fetch` converts HTML to markdown and strips the entire `<head>` section.** This means it CANNOT detect:

- `<meta name="description">` (meta descriptions)
- `<meta property="og:*">` (Open Graph tags)
- `<meta name="twitter:*">` (Twitter Card tags)
- `<link rel="canonical">` (canonical tags)
- `<link rel="alternate" hreflang="...">` (hreflang tags)
- `<meta name="robots">` (robots directives)
- `<script type="application/ld+json">` (schema markup / JSON-LD)

**Reporting "not found" based solely on `web_fetch` output leads to false audit findings.**

### How to Verify `<head>` SEO Elements

**For meta tags, canonical tags, hreflang, and OG tags**, use `curl` with raw HTML parsing:

```bash
curl -s -L "https://example.com" | python3 -c "
import sys, re
html = sys.stdin.read()
for tag in re.findall(r'<meta[^>]*>', html, re.IGNORECASE):
    print(tag)
for tag in re.findall(r'<link[^>]*(?:canonical|alternate)[^>]*>', html, re.IGNORECASE):
    print(tag)
"
```

Run this for EVERY page being audited. Never skip this step.

**For schema markup / JSON-LD**, use one of these methods:
1. **`curl` + regex** — extract `<script type="application/ld+json">` blocks from raw HTML
2. **Google Rich Results Test** — https://search.google.com/test/rich-results (renders JavaScript)
3. **Browser tool** — run: `document.querySelectorAll('script[type="application/ld+json"]')`
4. **Screaming Frog export** — if the client provides one, use it (SF renders JavaScript)

Note: `curl` will find JSON-LD embedded in static HTML but NOT schema injected via client-side JavaScript. For JS-injected schema, use the Rich Results Test or a browser tool.

### Redirect Verification

**`web_fetch` auto-follows redirects silently.** It cannot detect 301/302 redirects, redirect chains, or missing redirects. Two URLs may appear to serve identical content when one actually redirects to the other.

**To check redirects**, use `curl` without `-L`:

```bash
curl -s -o /dev/null -w "Status: %{http_code}, Redirect: %{redirect_url}\n" "https://example.com"
```

Run this for every redirect scenario being audited (www vs non-www, HTTP vs HTTPS, trailing slash, etc.).

### Large Page Content Rendering

**`web_fetch` can poorly convert large SSR pages (400KB+).** On pages with heavy inline CSS/JS (common in Next.js, Nuxt, SvelteKit), `web_fetch` may return mostly CSS/JS fragments, making it appear the page has no content — even when the HTML is fully server-side rendered.

**If `web_fetch` shows mostly CSS/JS for a page, verify with `curl`:**

```bash
curl -s -L "https://example.com/page" | python3 -c "
import sys, re
data = sys.stdin.read()
# Strip scripts and styles
clean = re.sub(r'<script[^>]*>.*?</script>', '', data, flags=re.DOTALL | re.IGNORECASE)
clean = re.sub(r'<style[^>]*>.*?</style>', '', clean, flags=re.DOTALL | re.IGNORECASE)
# Find headings
for tag in ['h1','h2','h3']:
    matches = re.findall(r'<' + tag + r'[^>]*>(.*?)</' + tag + r'>', clean, re.IGNORECASE | re.DOTALL)
    texts = [re.sub(r'<[^>]+>', '', m).strip() for m in matches if re.sub(r'<[^>]+>', '', m).strip()]
    if texts: print(f'{tag.upper()}: {texts[:5]}')
# Count content paragraphs
ps = re.findall(r'<p[^>]*>(.*?)</p>', clean, re.IGNORECASE | re.DOTALL)
real = [p for p in ps if len(re.sub(r'<[^>]+>', '', p).strip()) > 30]
print(f'Content paragraphs: {len(real)}')
"
```

Do NOT report "JS rendering issues" or "content not rendered" based solely on `web_fetch` output. Always confirm with `curl` + HTML parsing first.

### When to Use `web_fetch` vs `curl`

| Check | Use `web_fetch` | Use `curl` |
|-------|----------------|------------|
| Page content, headings, body text | ✅ Yes (but verify large pages with curl) | Required for large/SSR pages |
| Internal/external links | ✅ Yes | Overkill |
| Image alt text | ✅ Yes (partial) | Better for completeness |
| Meta descriptions | ❌ No — stripped | ✅ Required |
| Canonical / hreflang tags | ❌ No — stripped | ✅ Required |
| OG / Twitter meta tags | ❌ No — stripped | ✅ Required |
| Robots meta tag | ❌ No — stripped | ✅ Required |
| Schema markup (JSON-LD) | ❌ No — stripped | ✅ For static; Rich Results Test for JS-injected |
| Redirects (301/302) | ❌ No — auto-follows | ✅ Required (without `-L`) |

### Priority Order
1. **Crawlability & Indexation** (can Google find and index it?)
2. **Technical Foundations** (is the site fast and functional?)
3. **On-Page Optimization** (is content optimized?)
4. **Content Quality** (does it deserve to rank?)
5. **Authority & Links** (does it have credibility?)

---

## Technical SEO Audit

### Crawlability

**Robots.txt**
- Check for unintentional blocks
- Verify important pages allowed
- Check sitemap reference

**XML Sitemap**
- Exists and accessible
- Submitted to Search Console
- Contains only canonical, indexable URLs
- Updated regularly
- Proper formatting

**Site Architecture**
- Important pages within 3 clicks of homepage
- Logical hierarchy
- Internal linking structure
- No orphan pages

**Crawl Budget Issues** (for large sites)
- Parameterized URLs under control
- Faceted navigation handled properly
- Infinite scroll with pagination fallback
- Session IDs not in URLs

### Indexation

**Index Status**
- site:domain.com check
- Search Console coverage report
- Compare indexed vs. expected

**Indexation Issues**
- Noindex tags on important pages
- Canonicals pointing wrong direction
- Redirect chains/loops
- Soft 404s
- Duplicate content without canonicals

**Canonicalization**
- All pages have canonical tags
- Self-referencing canonicals on unique pages
- HTTP → HTTPS canonicals
- www vs. non-www consistency
- Trailing slash consistency

### Site Speed & Core Web Vitals

**Core Web Vitals**
- LCP (Largest Contentful Paint): < 2.5s
- INP (Interaction to Next Paint): < 200ms
- CLS (Cumulative Layout Shift): < 0.1

**Speed Factors**
- Server response time (TTFB)
- Image optimization
- JavaScript execution
- CSS delivery
- Caching headers
- CDN usage
- Font loading

**Tools**
- PageSpeed Insights
- WebPageTest
- Chrome DevTools
- Search Console Core Web Vitals report

### Mobile-Friendliness

- Responsive design (not separate m. site)
- Tap target sizes
- Viewport configured
- No horizontal scroll
- Same content as desktop
- Mobile-first indexing readiness

### Security & HTTPS

- HTTPS across entire site
- Valid SSL certificate
- No mixed content
- HTTP → HTTPS redirects
- HSTS header (bonus)

### URL Structure

- Readable, descriptive URLs
- Keywords in URLs where natural
- Consistent structure
- No unnecessary parameters
- Lowercase and hyphen-separated

---

## On-Page SEO Audit

### Title Tags

**Check for:**
- Unique titles for each page
- Primary keyword near beginning
- 50-60 characters (visible in SERP)
- Compelling and click-worthy
- No brand name placement (SERPs include brand name above title already)

**Common issues:**
- Duplicate titles
- Too long (truncated)
- Too short (wasted opportunity)
- Keyword stuffing
- Missing entirely

### Meta Descriptions

**Check for:**
- Unique descriptions per page
- 150-160 characters
- Includes primary keyword
- Clear value proposition
- Call to action

**Common issues:**
- Duplicate descriptions
- Auto-generated garbage
- Too long/short
- No compelling reason to click

### Heading Structure

**Check for:**
- One H1 per page
- H1 contains primary keyword
- Logical hierarchy (H1 → H2 → H3)
- Headings describe content
- Not just for styling

**Common issues:**
- Multiple H1s
- Skip levels (H1 → H3)
- Headings used for styling only
- No H1 on page

### Content Optimization

**Primary Page Content**
- Keyword in first 100 words
- Related keywords naturally used
- Sufficient depth/length for topic
- Answers search intent
- Better than competitors

**Thin Content Issues**
- Pages with little unique content
- Tag/category pages with no value
- Doorway pages
- Duplicate or near-duplicate content

### Image Optimization

**Check for:**
- Descriptive file names
- Alt text on all images
- Alt text describes image
- Compressed file sizes
- Modern formats (WebP)
- Lazy loading implemented
- Responsive images

### Internal Linking

**Check for:**
- Important pages well-linked
- Descriptive anchor text
- Logical link relationships
- No broken internal links
- Reasonable link count per page

**Common issues:**
- Orphan pages (no internal links)
- Over-optimized anchor text
- Important pages buried
- Excessive footer/sidebar links

### Keyword Targeting

**Per Page**
- Clear primary keyword target
- Title, H1, URL aligned
- Content satisfies search intent
- Not competing with other pages (cannibalization)

**Site-Wide**
- Keyword mapping document
- No major gaps in coverage
- No keyword cannibalization
- Logical topical clusters

---

## Content Quality Assessment

### E-E-A-T Signals

**Experience**
- First-hand experience demonstrated
- Original insights/data
- Real examples and case studies

**Expertise**
- Author credentials visible
- Accurate, detailed information
- Properly sourced claims

**Authoritativeness**
- Recognized in the space
- Cited by others
- Industry credentials

**Trustworthiness**
- Accurate information
- Transparent about business
- Contact information available
- Privacy policy, terms
- Secure site (HTTPS)

### Content Depth

- Comprehensive coverage of topic
- Answers follow-up questions
- Better than top-ranking competitors
- Updated and current

### User Engagement Signals

- Time on page
- Bounce rate in context
- Pages per session
- Return visits

---

## Common Issues by Site Type

### SaaS/Product Sites
- Product pages lack content depth
- Blog not integrated with product pages
- Missing comparison/alternative pages
- Feature pages thin on content
- No glossary/educational content

### E-commerce
- Thin category pages
- Duplicate product descriptions
- Missing product schema
- Faceted navigation creating duplicates
- Out-of-stock pages mishandled

### Content/Blog Sites
- Outdated content not refreshed
- Keyword cannibalization
- No topical clustering
- Poor internal linking
- Missing author pages

### Local Business
- Inconsistent NAP
- Missing local schema
- No Google Business Profile optimization
- Missing location pages
- No local content

---

## Output Format

### Audit Report Structure

**Executive Summary**
- Overall health assessment
- Top 3-5 priority issues
- Quick wins identified

**Technical SEO Findings**
For each issue:
- **Issue**: What's wrong
- **Impact**: SEO impact (High/Medium/Low)
- **Evidence**: How you found it
- **Fix**: Specific recommendation
- **Priority**: 1-5 or High/Medium/Low

**On-Page SEO Findings**
Same format as above

**Content Findings**
Same format as above

**Prioritized Action Plan**
1. Critical fixes (blocking indexation/ranking)
2. High-impact improvements
3. Quick wins (easy, immediate benefit)
4. Long-term recommendations

---

## References

- [AI Writing Detection](references/ai-writing-detection.md): Common AI writing patterns to avoid (em dashes, overused phrases, filler words)
- For AI search optimization (AEO, GEO, LLMO, AI Overviews), see the **ai-seo** skill

---

## Tools Referenced

**Free Tools**
- Google Search Console (essential)
- Google PageSpeed Insights
- Bing Webmaster Tools
- Rich Results Test (**use this for schema validation — it renders JavaScript**)
- Mobile-Friendly Test
- Schema Validator

> **Note on `web_fetch` limitations:** `web_fetch` strips the entire HTML `<head>` section during markdown conversion. This means it cannot detect meta descriptions, canonical tags, hreflang, OG tags, robots meta, or JSON-LD schema. **Always use `curl` with raw HTML parsing to verify `<head>`-level SEO elements.** See the "`web_fetch` HTML Stripping Limitation" section above for the required verification method.

**Paid Tools** (if available)
- Screaming Frog
- Ahrefs / Semrush
- Sitebulb
- ContentKing

---

## Task-Specific Questions

1. What pages/keywords matter most?
2. Do you have Search Console access?
3. Any recent changes or migrations?
4. Who are your top organic competitors?
5. What's your current organic traffic baseline?

---

## Related Skills

- **ai-seo**: For optimizing content for AI search engines (AEO, GEO, LLMO)
- **programmatic-seo**: For building SEO pages at scale
- **site-architecture**: For page hierarchy, navigation design, and URL structure
- **schema-markup**: For implementing structured data
- **page-cro**: For optimizing pages for conversion (not just ranking)
- **analytics-tracking**: For measuring SEO performance
