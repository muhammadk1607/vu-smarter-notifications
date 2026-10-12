# Which free tier can run a scheduled scraper against VU?

Research for issue #4 (parent map: #1). Researched 2026-10-12. Each claim cites the page it came from; where no primary source exists, the claim is labelled **community** or **measured**.

## TL;DR

- **The VU LMS login page uses invisible reCAPTCHA v3.** This changes the plan more than any hosting limit does. Logging in needs a real browser that can run Google's script and send a `g-recaptcha-response` token, so a plain `fetch` + form POST won't work. reCAPTCHA v3 also gives datacenter IPs and headless browsers low scores. Whether VU rejects low scores is still unknown and needs one test login (see "Open risks").
- **VU is reachable from cloud IPs.** HTTPS to `vulms.vu.edu.pk` returned 200 from AWS, Google Cloud, Microsoft/Azure (the network GitHub-hosted runners use), Oracle, Hetzner and DigitalOcean in the US, EU and Asia. Neither VU site sits behind Cloudflare: both are self-hosted IIS on Pakistan's education network (PERN). I found no reports of geo-blocking.
- **Recommendation:** run the checker on **GitHub Actions in a public repo**, which has free, unlimited standard-runner minutes and Playwright/Chromium out of the box. Keep state in **Turso** (or Supabase). If GitHub's cron timing proves too loose, trigger the workflow from a **Cloudflare Workers Cron Trigger**. **Oracle Always Free VM** is the fallback if we need a long-lived browser session or a private repo.
- **Ruled out for running the scraper:** Vercel Hobby (cron at most once a day), Render free (no free cron jobs or workers), Fly.io (no free tier for new organizations), Supabase Edge Functions and Deno Deploy (no headless Chromium). Cloudflare Browser Rendering's free 10 browser-minutes a day is too small for logins for 5–15 Members.

## Our load

From map #1: Institutional Windows every 30 min and Personal Deadlines every 6 h, for 5–15 Members.

| Check | Runs/day | Runs/month | Browser logins/day (worst case: no session reuse) |
|---|---|---|---|
| Institutional Windows (shared; assume one account's login) | 48 | ~1,440 | 48 |
| Personal Deadlines (per Member) | 4 | ~120 | 4 × 15 = 60 |
| **Total** | 52 | ~1,560 | **~108** |

At roughly 15–30 s of browser time per login and page walk, that is **~27–54 browser-minutes a day**. If the Institutional Windows check turns out not to need a login (public pages on vu.edu.pk), it drops to ~15–30 min a day.

## Comparison

| Option | Scheduling | Chromium / Playwright? | Free quota vs our load | Secrets | Gotchas | Verdict |
|---|---|---|---|---|---|---|
| **GitHub Actions, public repo** | `schedule` cron, at most every 5 min; can be delayed, especially at the top of the hour, and jobs can be dropped under heavy load [GH-events] | Yes: full Ubuntu VM, `npx playwright install --with-deps chromium` | Standard GitHub-hosted runners are **free for public repos** [GH-billing], so effectively no limit | Encrypted Actions secrets, masked in logs | **Logs and artifacts are public**: never print derived data and turn off Playwright traces. **Scheduled workflows are disabled after 60 days with no repo activity in public repos** [GH-events], so we need a keepalive. ToS bars Actions "as part of a serverless application" and "activity unrelated to the… software project" [GH-terms], a grey area for a scheduled scraper | **Primary choice** |
| GitHub Actions, private repo | same | Yes | **2,000 min/month** on Free [GH-billing]. Each job is billed in whole minutes, so ~1,440 window runs × ~2 min + ~120 deadline runs × ~6 min ≈ 3,600 min. **Over quota.** Usage is blocked once the quota is spent, if no card is on file [GH-billing] | same | Same cron delays. Fits only if Window checks run hourly *and* have no browser start-up, which is unlikely | Too tight |
| **Cloudflare Workers + Cron Triggers** | Cron Triggers in UTC, every minute at the finest; **5 per account** on Free [CF-limits]; start times vary, and changes take up to 15 min to propagate [CF-cron] | Only through Browser Rendering (`@cloudflare/playwright` fork, Chromium only, a subset of the API) [CF-pw] | Workers Free: 100k requests/day, **10 ms CPU per cron invocation**, 50 subrequests [CF-limits]. Browser Rendering Free: **10 browser-minutes/day**, 3 concurrent, 1 new browser every 20 s [CF-br-limits]. That is **well under our ~27–54 min/day** | Workers secrets (encrypted env vars) | 10 ms CPU leaves little room for HTML parsing. D1 Free: 10 DBs, 500 MB each [CF-d1] | Not for scraping. **Good as a precise cron that triggers GitHub `workflow_dispatch`** |
| Render | **Cron jobs not available on Free**; minimum $1/month per cron service [Render-free][Render-cron] | Free web service is a container, so possible, but RAM isn't stated | Free web services spin down after 15 min with no inbound traffic; 750 instance-h/month; free Postgres **expires after 30 days** [Render-free] | Env vars | Outbound SMTP ports 25/465/587 are blocked [Render-free] | No |
| Supabase | **Supabase Cron** (pg_cron): every second to yearly; runs SQL or makes HTTP calls, e.g. to Edge Functions; ≤8 concurrent jobs, each ≤10 min recommended [SB-cron] | **No.** Edge Functions have 256 MB, 2 s CPU and 150 s wall clock on Free; no workers or multithreading [SB-fn] | 500 MB DB, 500k function invocations, 5 GB egress, 2 active projects [SB-pricing] | Vault / function secrets | **Free projects pause after 7 days of low activity** [SB-pricing]. Activity means dashboard visits or API calls [SB-pause]. A Supabase reply says real queries count and pg_cron may count (**community**). Our every-30-min writes over the API would keep it alive | **Fine as the database**; cannot run the browser |
| **Turso** | none (database only) | n/a | Free: 100 DBs, 5 GB, 500M rows read and 10M rows written per month [Turso]. Our load is a tiny fraction | Auth token | The pricing page describes no inactivity pausing [Turso] | **Preferred database** (no pausing, plain libSQL over HTTP) |
| Fly.io | Machines + cron | Yes (any container) | **No free tier for new organizations**; trial of 2 machine-hours or 7 days; card required [Fly] | Fly secrets | Only legacy (pre-Oct-2024) orgs keep free allowances [Fly] | No |
| **Oracle Cloud Always Free** | Your own `cron`/systemd timer on a VM | Yes: full Linux VM. A1 Arm has 2 OCPU / 12 GB total; 2× AMD micro VMs have 1 GB each [OCI] | Effectively unlimited for our load | Your own files / env (we own encryption) | Must be in the home region. **Idle instances are reclaimed** when CPU p95, network *and* (on A1) memory all stay under 20% for 7 days [OCI], which our light load would hit. Converting to Pay-As-You-Go reportedly exempts the instance (2023 Oracle notice, **community**). Card needed at signup; A1 capacity is often unavailable (**community**) | **Fallback**: can keep a browser session warm, and the repo stays private |
| Deno Deploy | `Deno.cron`; 10 crons per revision on Free; overlapping runs are skipped [Deno-cron] | No subprocess or Chromium support documented | Free: 1M requests/month, 10 h active CPU [Deno-pricing] | Env vars | — | No (no browser) |
| Vercel Hobby | **Once per day at most**, ±59 min precision; `*/30` fails to deploy [Vercel-cron] | — | — | — | — | No |

## Reachability of VU from outside Pakistan (measured 2026-10-12)

- DNS: `vulms.vu.edu.pk` → 111.68.103.38 and `www.vu.edu.pk` → 111.68.103.36, both AS45773 PERN, Lahore. `vu.edu.pk` → 58.27.231.136 (AS38264). Servers are `Microsoft-IIS/10.0` with `ARR/3.0`. There are **no Cloudflare headers**, so there is no Cloudflare bot wall in front.
- From this machine (Lahore, AS9541): `https://vulms.vu.edu.pk/` and `/LMS_LP.aspx` return 200 in ~0.5 s.
- Through [Globalping](https://globalping.io) probes (HTTPS HEAD `/LMS_LP.aspx`), **every probe returned HTTP 200**: AWS (Singapore, Tokyo), Google Cloud (US), **Microsoft AS8075** (Amsterdam, Singapore, Seoul), Oracle (Tokyo, London), Hetzner (DE), DigitalOcean (NY, AMS), LeaseWeb/xTom (SG) and HostPapa (US). `www.vu.edu.pk` also returned 200 from AWS, Hetzner and US probes. Latency was 0.4–1.5 s.
- A web search found no reports of geo-blocking. One scraped page quotes a VU error, "This site cannot be accessed over the ip address 40.77.167.3… contact vu network department" (a Microsoft IP). It suggests VU has **some IP-based block mechanism**, but the context is unclear (**weak signal**).
- **TLS gotcha:** `www.vu.edu.pk` / `vu.edu.pk` send an **incomplete certificate chain**: the Sectigo DV R36 intermediate is missing (`openssl` verify error 21). `curl` and Node `fetch` will fail on them. Fix it by adding the intermediate through `NODE_EXTRA_CA_CERTS`, not by turning off verification. `vulms.vu.edu.pk` serves the full chain.

## Login page findings (anonymous GET only, no login attempted)

`/LMS_LP.aspx` is an ASP.NET WebForms page (`__VIEWSTATE`, `__EVENTVALIDATION`, a single `<form method="post">`). It loads `https://www.google.com/recaptcha/api.js?render=…`, calls `grecaptcha.execute(…, {action: 'LMS20LandingPage'})`, refreshes the token every 119 s and posts it as `g-recaptcha-response`. This is reCAPTCHA **v3**, which is score-based and shows no challenge to the user.

Implications:
1. A real browser (Playwright/Chromium) is needed **at least for login**. Hosts without a browser (Supabase functions, Deno, plain Workers) can't log in.
2. After login, ASP.NET session/auth cookies can probably be reused with plain HTTP for the rest of the page walk. Logging in once and reusing the cookie jar until it expires cuts browser time and the number of reCAPTCHA calls. This favours a host that keeps state between runs: cookies stored encrypted in the DB, or a long-lived VM.
3. Datacenter IP + headless Chromium is the worst case for a v3 score. If VU enforces a threshold, cloud logins may fail **on every host in this table**. The only $0 fallback would then be a residential IP, for example the operator's own always-on machine or Raspberry Pi in Pakistan, which isn't a free tier.

## Recommendation shortlist

1. **GitHub Actions (public repo) + Turso.** $0, real Chromium, unlimited minutes, Azure IPs proven to reach VU. Mitigations:
   - Schedule at an off-hour minute (e.g. `7,37 * * * *`) to avoid top-of-hour delays [GH-events].
   - If drift matters, fire `workflow_dispatch` from a Cloudflare Workers Cron Trigger (free, 5 triggers).
   - Keepalive against the 60-day disable, e.g. a monthly workflow that calls `PUT /repos/{owner}/{repo}/actions/workflows/{id}/enable` or makes a small commit.
   - Keep Member credentials encrypted (one Actions secret holding the key; ciphertext in Turso). Never echo them, and don't upload traces or screenshots: logs are public.
   - Keep the repo code public but free of data. Check the ToS grey area [GH-terms]. The private-repo variant is over quota.
2. **Oracle Always Free A1 VM + SQLite/Turso.** Use it if a public repo is unacceptable, if session reuse needs a warm browser, or if GitHub's cron proves too unreliable. Watch idle reclamation (PAYG conversion) and signup friction.
3. **Supabase** is a drop-in alternative to Turso for the database (500 MB, Vault for secrets). Our frequent writes would prevent the 7-day pause.

## Open risks / follow-ups for other tickets

- **reCAPTCHA v3 from cloud IPs**: this is the biggest unknown. Before committing to a host, run one manual Playwright login with an operator-owned account from a GitHub Actions runner, outside this research, which did not log in. If it is rejected, the hosting answer becomes "residential IP".
- Session lifetime on VU LMS: decides how often we must log in, and so the browser-minute budget.
- Whether Institutional Windows can be read from public pages without login: decides most of the load.
- Polite checking (map #1, "Not yet specified"): VU appears to have IP-level blocking somewhere, so keep request rates low and stagger Members.

## Sources

- [GH-billing] GitHub Actions billing: https://docs.github.com/en/billing/concepts/product-billing/github-actions
- [GH-events] Events that trigger workflows, `schedule`: https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows
- [GH-terms] GitHub Terms for Additional Products, Actions: https://docs.github.com/en/site-policy/github-terms/github-terms-for-additional-products-and-features
- [CF-limits] Workers limits: https://developers.cloudflare.com/workers/platform/limits/
- [CF-cron] Cron Triggers: https://developers.cloudflare.com/workers/configuration/cron-triggers/
- [CF-br-limits] Browser Rendering limits: https://developers.cloudflare.com/browser-rendering/limits/ (pricing: https://developers.cloudflare.com/browser-rendering/pricing/)
- [CF-pw] Playwright on Browser Rendering: https://developers.cloudflare.com/browser-rendering/playwright/
- [CF-d1] D1 limits: https://developers.cloudflare.com/d1/platform/limits/
- [Render-free] Render free instances: https://render.com/docs/free
- [Render-cron] Render cron jobs: https://render.com/docs/cronjobs
- [SB-pricing] Supabase pricing: https://supabase.com/pricing
- [SB-fn] Edge Function limits: https://supabase.com/docs/guides/functions/limits
- [SB-cron] Supabase Cron: https://supabase.com/docs/guides/cron
- [SB-pause] Project pausing: https://supabase.com/docs/guides/platform/free-project-pausing (community detail: https://github.com/orgs/supabase/discussions/13121)
- [Turso] Turso pricing: https://turso.tech/pricing
- [Fly] Fly.io pricing: https://docs.fly.io/about/pricing
- [OCI] Oracle Always Free resources: https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm (PAYG exemption, community: https://blog.51sec.org/2023/02/oracle-cloud-cleaning-up-idle-compute.html)
- [Deno-cron] Deno Deploy cron: https://docs.deno.com/deploy/reference/cron/
- [Deno-pricing] Deno Deploy pricing: https://deno.com/deploy/pricing
- [Vercel-cron] Vercel cron usage & pricing: https://vercel.com/docs/cron-jobs/usage-and-pricing
- Reachability: Globalping measurements `2t4bpNiPUHDDlGTLB00021Iqe`, `2KfMKQ4wr6pxGJP7T00021Iqe`, `2V4D5XBo3NME15kgP00021Iqe` (https://api.globalping.io/v1/measurements/{id}); local `curl`/`openssl s_client`/`dig` from Lahore.
