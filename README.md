# Dark-funnel attribution — discussion notes

Capture of product thinking around a dark-funnel attribution platform, a micro-SaaS wedge, and validation.

## Core idea

A customer installs a lightweight script / GTM tag and connects Search Console. Every URL they share can be converted into a trackable link with a unique referral ID and UTM.

When that link is shared across Reddit, Slack, Discord, WhatsApp, Telegram, DMs, forums, etc., the platform tracks visits and connects them to downstream goals like signup, demo, or purchase.

For dark-funnel activity that cannot be directly attributed (e.g. organic Reddit discussions), the platform can combine public intent signals with website/conversion data to provide **estimated influence and confidence**, rather than claiming deterministic attribution.

**Is this possible?** Yes.

## Complexity

A useful MVP is achievable. Credible "estimated dark-funnel influence" is the hard part.

| Layer | Difficulty | Why |
|---|---|---|
| Script / GTM tag | Low–medium | First-party tracking, cookies/localStorage, event forwarding |
| Trackable links + referral IDs | Low | URL shortener + redirect + click logging |
| Click → visit → conversion | Medium | UTM/referrer capture, identity stitching, analytics integrations |
| Multi-channel sharing detection | Medium | Clean data only when people use your links |
| True dark-funnel inference | High | Probabilistic modeling, noisy public signals, confidence scoring |
| Enterprise-grade product | Very high | Privacy, GDPR, bot filtering, deduping, reporting, integrations |

### Straightforward

- Lightweight JS snippet for pageviews/events
- Unique links (`yoursite.com/r/abc123`)
- Redirect + click logging (timestamp, referrer, device, geo)
- Passing UTMs into the site and matching them to signups/purchases
- Dashboards: link X → N visits → M conversions

This is link attribution + analytics.

### What gets hard

1. **Attribution is messy** — shares without clicks, in-app browsers stripping referrers, multi-device, copied/shortened links, organic mentions with no trackable URL. Deterministic attribution only works when the link is actually clicked. Everything else is inference.
2. **Dark-funnel estimation** — public signal ingestion, mention detection, time-series correlation with traffic/branded search/signups, models that output influence + confidence. Correlation ≠ causation; delayed effects; overlapping channels; small samples; trust.
3. **Identity stitching** — anonymous click → session → signup → demo → purchase across cookies, CRM, Stripe, HubSpot. Never perfect.
4. **Privacy / compliance** — GDPR, consent, cookie deprecation, not claiming what cannot be proven.

### Tiers

- MVP (trackable links + basic attribution): medium
- Real product (integrations + reporting): high
- Differentiated dark-funnel estimation: very high (data platform + research, not just an app)

Hardest part is not the script or links — it is **credible probabilistic attribution** for activity that cannot be directly observed.

Easiest credible wedge: trackable links + first-party conversion tracking, then estimated influence only with explicit uncertainty.

## Tag + Search Console

Onboarding: install tag, then connect Search Console.

| Source | Primary value |
|---|---|
| Tag | First-party truth: pageviews, sessions, conversions, referral params, on-site behavior |
| Search Console | Organic demand: branded vs non-branded queries, impressions, clicks, landing pages, trends |

Tag = owned event layer. GSC = external signal layer.

### Direct attribution (high confidence)

Someone clicks a trackable link → tag captures referral ID / UTM → tie session to signup/demo/purchase.

### Dark-funnel inference (medium confidence)

Someone sees a Reddit thread without clicking, then converts later. No click event, but possible:

- Spike in branded search in Search Console
- Timing correlated with public mention activity

Example framing: "This Reddit discussion likely influenced 12–28 signups over the next 7 days (confidence: medium)."

### Content / landing-page attribution

Search Console helps: which pages gained impressions/clicks after a mention; shift from generic to branded queries; dark-funnel showing up as search intent before conversion.

### Connect flow

Google OAuth for Search Console, user picks verified site.

- **Tag only**: click → visit → conversion for owned links. Limited organic/discussion view.
- **Tag + GSC**: best for dark-funnel estimation. Bridge: public mention → branded search → site visit → conversion.

Adding GSC raises complexity from medium to high. The integration is standard; joining noisy signals on a timeline without overclaiming is hard.

**Limitations:**

- Tag should remain primary for conversions you care about.
- Search Console is delayed and aggregated (~2–3 days). Trends, not user-level journeys.
- Cannot attribute individual Reddit readers deterministically. Cohort/trend level only.
- User must have admin/access to the correct GSC site.

## Unique UTM + referral ID on every external share

Every external share is a first-class attribution object.

Example:

```
https://yoursite.com/pricing
  → https://go.customerbrand.com/r/abc123
  → redirects to:
https://yoursite.com/pricing?utm_source=reddit&utm_medium=social&utm_campaign=launch-thread&utm_content=abc123&ref_id=abc123
```

Params can live only in the redirect layer and be written into first-party storage by the tag.

| Field | Purpose |
|---|---|
| Referral ID (`ref_id`) | Canonical ID — unique per link, used in DB and tag |
| UTM params | Human-readable campaign metadata |

One referral ID maps to one UTM set, stored at link creation.

### Flow

1. Customer creates link for a destination + channel
2. Platform saves `ref_id`, destination, UTMs, channel, creator
3. Returns trackable URL
4. Visitor clicks → click event logged → 302 with UTMs + `ref_id`
5. Tag captures params, persists cookie/localStorage, fires attribution event
6. Conversion events attach `ref_id`

### Link record (minimum)

- `ref_id` (unique, primary)
- `customer_id`
- `destination_url`
- `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`
- `channel`
- `created_by`, `created_at`
- optional `label`

### Click record (minimum)

- `click_id`, `ref_id`, timestamp
- `ip_hash`, user agent, referrer, geo, device

### Tag on landing

1. Read `ref_id` and UTMs from URL
2. Persist first-party (e.g. 30–90 days)
3. Fire `link_attribution`
4. Keep `ref_id` on subsequent events
5. On conversion, send `ref_id` with the event

UTMs alone are weak (editable, reused). Referral ID alone is strong internally. Together: `ref_id` = source of truth; UTMs = campaign metadata.

Reporting per link: clicks (redirect), visits (tag), unique visitors, time to conversion, signups/demos/purchases, bounce/pages, assisted branded search lift (GSC, inferred).

**Design rules:**

1. One link = one intent (separate links per destination + channel + placement)
2. Redirect must append UTMs + `ref_id` without stripping existing destination query params
3. First-touch vs last-touch: last-click for campaign performance; first-touch stored separately for influence
4. Explicit attribution window (7/30/90 days)
5. On signup, bind `ref_id` to `user_id`
6. If someone copies the final URL with params, attribution still works if params survive

Does not solve: no-click mentions, word-of-mouth with no URL, later branded Google search without params.

## WhatsApp, Telegram, Instagram, X

Works when people click the trackable link. Does not work for private conversations you cannot see.

| Platform | Click tracking | Referrer data | Private chat attribution | Organic mention attribution |
|---|---|---|---|---|
| WhatsApp | Yes | Usually none | No | No |
| Telegram | Yes | Usually none | No | Only public channels/groups |
| Instagram | Yes | Usually none | No | Limited (public posts/stories) |
| X | Yes | Usually none | No | Yes for public posts (with listening) |

Referral ID + UTM are the source of truth, not platform referrer. In-app browsers often strip referrer; tag still wins if it captures `ref_id`.

- **WhatsApp**: click → attributed. Verbal rec / screenshot / brand name only → not attributable. No API to private chats.
- **Telegram**: same for private chats. Public channels can be monitored if posted. Click → conversion works.
- **Instagram**: bio/story/caption links trackable on click. Stories/reels with no link: no click attribution.
- **X**: tweet/reply/bio links trackable. t.co wrapping does not break redirect tracking. Mentions without link: estimable via listening + search lift.

Clicked link = high-confidence attribution. No click = estimated influence, not proof.

## Private share — definition

A private share is when the product, link, or message is shared in a place you cannot observe and the platform does not expose publicly.

Closed or personal context, without a trackable public signal.

- **Private**: WhatsApp 1:1 or group (unless you are in it), Telegram DM, Instagram DM, X DM, Slack DM, email to one person, SMS/iMessage, Zoom rec, in-person, screenshot without the link, brand name with no URL.
- **Not private**: tweet, public IG post/reel, public Telegram channel, Reddit thread, LinkedIn post, public Discord, YouTube comment, forum post.

| Share type | Detect that share happened? | Attribute conversion? |
|---|---|---|
| Private share with trackable link clicked | No | Yes (via click) |
| Private share with no link clicked | No | No |
| Public share with trackable link clicked | Often yes | Yes |
| Public share with no link clicked | Often yes | Estimated only |

A share can be private even if a link is included. Alice sends trackable link to Bob on WhatsApp: you cannot see Alice shared it; if Bob clicks, you can attribute Bob's visit/conversion; you still cannot see Alice was the sharer unless encoded in the link and clicked.

Private share = hidden distribution. Not the same as untrackable. Becomes trackable only when the recipient clicks the unique link.

## Private share: estimates, traffic, impressions

Partly. Some things can be estimated. True private impressions cannot be counted with confidence.

| Metric | Private share (no click) | Private share (link clicked) |
|---|---|---|
| Impressions | No direct count | No direct count |
| Estimated impressions | Very rough guess | Rough estimate possible |
| Traffic | No | Yes |
| Conversions | No | Yes |
| Estimated influence | Maybe, aggregate | Yes, with more signal |

No access to message, recipient count, or reads. No real impression data like ads.

Traffic: if unique link is clicked — clicks, visits, signups. Example: 18 clicks from `ref_id=abc123` ≠ group size or how many saw it.

Estimated impressions: only via proxies (clicks, multi-device/location on same link, branded search lift, direct/organic spikes, conversion lift after a known window). Range + confidence, not exact counts.

Traffic without clicks: only weakly, aggregate level (e.g. branded search +25% after a known private share). Cannot tie cleanly to one message.

Honest framing: measured = clicks/visits/conversions from unique link. Estimated = possible exposures / influenced visits. Confidence = low/medium/high. Never claim exact impressions.

## Website backlinks (link placed on a linking page)

If the trackable URL (or destination with UTMs intact) is placed on another site (blog, directory, forum signature, "tools we use"), that is a backlink + referral source. This is one of the easier cases.

On click you can often capture: `ref_id`, UTMs, HTTP referrer (the linking page), visit + conversion.

Two layers:

- Link attribution: `ref_id` / UTM → original created link
- Referrer attribution: `referrer=someblog.com` → actual sending page

They can differ. Example: link created for Reddit (abc123) later copied onto a blog → traffic from blog, still tied to abc123.

Works if published URL is the short trackable URL or destination with params intact.

Breaks if: UTMs/`ref_id` stripped, query params stripped, wrapping redirect drops params, text mention without link.

Vs private share: website backlinks usually give traffic + often referrer + preserved `ref_id`. Still no exact impressions on the linking page.

Report: clicks/visits, conversions, referring domains, top linking pages, original `ref_id`.

## How big is the problem

Big and real, especially B2B/SaaS. Messy, which is why it is still unsolved well.

Most buying happens in places analytics cannot see cleanly. Teams know dashboards are wrong and still make budget decisions with them.

Directional industry signals (ranges, not gospel):

| Signal | Rough scale |
|---|---|
| B2B journey done before sales contact | ~60–70% (6sense 2025: 61%) |
| Buying influence in dark social / dark funnel | ~30–60% |
| Social sharing in untracked channels | ~80%+ |
| B2B teams still on last-touch | ~67% |
| Pipeline from channels analytics cannot see (self-reported) | ~30–50% |
| "Direct" traffic for major B2B brands | often ~65–72% of visits |

Pain: misallocated budget; broken trust in attribution; buyers trust peers (~73% in Reddit/SurveyMonkey research) more than vendor sites; Reddit/forums as shortlist builders that later show as anonymous/direct.

High pain: B2B SaaS, dev tools, PLG, founder-led GTM, content + community, brand spend without ROI proof.

Lower pain: pure ecommerce with clean paid ads, short cycles, single obvious channel.

Market: attribution tools, product analytics, social listening, community analytics, self-reported attribution, MMM/incrementality. Players touch parts of it (HockeyStack, Dreamdata, Northbeam, Common Room, 6sense, Factors.ai). Fragmented. Full "dark-funnel attribution platform" still open because nobody can perfectly solve it and customers distrust fake precision.

Does not mean every private WhatsApp impression is measurable, or every Reddit thread is deterministic, or every company buys day one.

Does mean: large invisible influence, Direct as junk drawer, demand for better channel answers, value in measured clicks + estimated influence + confidence.

Three layers:

- **Measurable**: trackable links, tag, conversions — high confidence
- **Partially visible**: backlinks, branded search — medium
- **Hidden**: private shares, WOM, offline recs — estimated only

## 6sense, HockeyStack, and peers

They sell into the problem. None fully solve dark-funnel attribution if "solved" means deterministic proof for every influence path.

| Company | Strong at | Does not fully solve |
|---|---|---|
| 6sense | Account intent, buying stage, ICP fit, sales timing | Private WhatsApp/Slack, exact channel credit |
| HockeyStack | Multi-touch attribution, journey viz, some dark-funnel modeling | True private-share measurement, perfect causality |
| Dreamdata | B2B revenue attribution across ads + site + CRM | Offline/WOM, unseen community influence |
| Northbeam | Media mix + modeled attribution for DTC | B2B long-cycle peer influence |
| Common Room | Community signals, advocates, public intent | Closed DMs, private group attribution |

Why they sell: Direct is bloated; community influence is obvious but unmeasured; budget defense; "leads from nowhere"; WOM unquantified. Buyers pay for better visibility, not perfect truth.

Why unsolved: structurally missing private data; inference presented confidently; B2B is account-level and long (many stakeholders, many untracked touches); incentives to overclaim.

6sense is more "who is likely in-market" than "which Reddit comment caused revenue." HockeyStack is more "connect the visible journey and model the gaps."

Room exists if positioned as: trackable links for every share; tag + GSC; honest measured vs estimated; built for founders/marketers sharing everywhere — not just enterprise ABM.

Do not claim complete dark-funnel attribution. Claim: measure what can be measured, estimate what cannot, with confidence levels.

## AI as a factor

AI makes the problem bigger and a better solution more possible.

New invisible research: ChatGPT, Perplexity, Gemini, Copilot, Claude. Buyers ask for tools, then Google the brand or visit direct. Analytics often shows Direct / Organic / Unknown. Some self-reported "how did you hear about us?" already puts AI search in a meaningful share for B2B SaaS.

Why it worsens attribution: zero-click research; no clean referrer; AI blends Reddit + reviews + site + competitors; peer-like trust.

Where AI helps a solution: mention detection; intent classification; sentiment; correlating mention spikes with branded search/traffic/signups; influence estimation with ranges; detecting trackable links on new sites; AI citation monitoring; explainable reports.

Does not give perfect attribution. Can make estimation better.

Risk: fake precision and causal claims from weak correlation. Prefer measured vs estimated, confidence bands, explainable signals.

Winning approach: deterministic tracking where possible + AI-powered estimation where not.

## Factors.ai (factors.ai)

AI ABM + B2B attribution for GTM teams.

Focus: which companies visit the site; unify website + ads + CRM + G2; score in-market accounts; attribute pipeline/revenue; help sales/marketing act.

Closer to 6sense / HockeyStack / Dreamdata than to "track every shared link."

Strong: anonymous visitor ID (company), ABM scoring, paid channel attribution (LinkedIn, Google, Meta, Bing), CRM journey, G2/third-party intent, AI copilot (Scout / MCP), enterprise GTM workflows.

Not really built around: unique referral links per external share; WhatsApp/Telegram/IG DMs; Reddit/forum dark-funnel estimation; "this specific shared link in a private chat drove X signups"; GSC branded-intent lift as a core loop; confidence-based influence for unseen shares.

Factors answers: which target accounts engaged, and what influenced pipeline.

This concept answers: which exact link shared externally drove visits/conversions, and what unseen influence happened around it.

Different wedge. Factors validates that companies pay for better attribution. It does not fully solve link-sharing + dark-funnel estimation for founder-led distribution.

Do not compete as a cheaper Factors. Compete on links, posts, shares, and estimated influence that account ABM does not own.

## Micro-SaaS first vs the big platform

The big vision is valid. Start as micro-SaaS, prove people pay for one painful slice, then expand.

Do not start as: "AI dark-funnel attribution platform for B2B GTM teams."

Start as: "Create a trackable link for every place you share your product, and see which share drove signups."

Why: full platform is hard to sell before knowing what they pay for; the wedge is immediately useful; revenue + usage language come faster; estimation later sits on real click→conversion data.

**Phase 1 — link attribution micro-SaaS**
Tag; unique links; clicks / visits / conversions; basic dashboard.
Skip initially: AI influence engine, Reddit scraping, account deanonymization, CRM, ABM, enterprise workflows.

**Phase 2 — better measurement**
Search Console; referrer domain reporting; backlink propagation.

**Phase 3 — dark-funnel estimation**
Public mention detection; branded search lift; estimated influence + confidence.

**Phase 4 — full platform (only if demand proves it)**
AI agents; account intelligence; CRM; team workflows; enterprise plans.

A lean product can stay profitable as micro-SaaS. Optional expansion.

Mistake: building the full AI platform first and hoping people understand it.

Positioning now: track every link you share and see which one drives signups.
Positioning later: measure and estimate dark-funnel influence across links, communities, search, and AI.

## Distribution-first validation

Belief: find 25 people who sign up and 5 who pay before overbuilding.

| Milestone | What it proves |
|---|---|
| 25 signups | The message resonates |
| 5 paying | The pain is urgent enough to pay for |

Early offer can be free for first 25, or founding members at low monthly. Manual/concierge setup is fine.

Promise: unique trackable link per share; see clicks, visits, signups per link.

One-sentence options discussed:

- "You share your product on Reddit, X, LinkedIn, and DMs — but you never know which link actually drove signups."
- "If you're doing founder-led distribution, you need a unique link for every share."

Tiny validation asset: landing page, demo/mock of per-link stats, founder post. No full product required.

Channels: Reddit, X, Indie Hackers, LinkedIn, Slack/Discord SaaS groups. Subreddits mentioned: r/SaaS, r/startups, r/Entrepreneur, r/indiehackers, r/marketing, r/growthhacking.

Pitch as the problem, not a platform. Qualify: share weekly, founder-led distribution, care about signups not just clicks.

Signup questions discussed:

- Where do you share most?
- How often do you share external links?
- Want clicks, visits, or signups most?

Concierge: they give destination URL; generate `go…/r/abc123`; track clicks; help with tag; report results.

Payment after they have live links and one useful insight.

Funnel discussed as directional: hundreds–thousands of the right people → 100–300 landing visits → 25 signups → 10–15 activated → 5 paying.

Measure: signup source, activation (first link), return to check results, paid, objections.

Do not, for validation: AI estimation, GSC first, full dashboard, 6sense comparisons, enterprise features, waiting for a perfect product.

ICP: solo SaaS founders, indie hackers, B2B marketers in communities, devtool founders, agencies running launches. Avoid enterprise ABM, paid-ads-only, people who never share links.

Useful user language: which Reddit post worked; DMs with no idea; different link per post; pay to stop guessing.

## Working rules (from this conversation)

- Decisions stay with the user.
- No unsolicited follow-ups or "want me to…" unless asked.
- No steering the decision. Answer what is asked.
- Keep responses from breaking flow.
