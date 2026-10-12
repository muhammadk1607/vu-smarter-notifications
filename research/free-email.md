# Which free email service should send Reminders?

Research for issue #5 (parent map #1). Checked 2026-10-12 against vendor pricing pages and docs. Free tiers change often, so recheck the numbers before you build.

## Requirement

- 5–15 **Members**. Each gets a few **Reminders** or **Window Alerts** a day plus one **Digest**. Worst case is about 15 × 6 = **~90 emails/day, ~2,700/month**. A typical day is well under that.
- Free tiers only. Node/TypeScript.
- The operator **may not own a domain**. This turns out to be the deciding constraint.
- Every email goes to a Member's own inbox, which is mostly Gmail, so deliverability to Gmail matters most.

## Comparison

| Service | Free limits | Domain / DNS needed? | Gmail deliverability with no domain | Node ergonomics | Serverless / edge |
|---|---|---|---|---|---|
| **Gmail SMTP + app password** | 500 messages/day on a personal account [1] | **No.** Sends as your own `@gmail.com`. | **Best available.** Gmail→Gmail mail is signed and aligned by Google itself. | Nodemailer, `service: "gmail"`. About 5 lines. | Needs outbound TCP 465/587: fine on GitHub Actions, Render and any Node VM. Not Cloudflare Workers, where Nodemailer doesn't run. |
| **Gmail API** (`users.messages.send`) | Same mailbox cap (500/day). API quota is not the bottleneck: 100 units per send, 6,000 units/min per user [2] | No | Same as above | `googleapis`, or a raw `fetch` with an OAuth refresh token. Heavier setup. | HTTPS only, so it works from edge/Workers too |
| **Resend** | 100/day, 3,000/month, 3 domains [3] | **Yes.** "You must add and verify at least one domain" [4]. The `resend.dev` test sender only delivers to your own account address [5]. | Excellent with a verified domain. Unusable without one. | Best of the lot: `resend` SDK, fetch-based, typed | Yes, HTTPS works everywhere |
| **Brevo** | 300/day, transactional included (vendor page did not render; figure from multiple 2026 secondary sources [6]). Brevo branding. | A single verified sender address is allowed, but Brevo's guidance pushes a custom domain. A free-mail From may get rewritten [6]. | Weak with a `@gmail.com` From: it fails DMARC alignment, see note below | REST API plus `@getbrevo/brevo` SDK, or SMTP relay | Yes, over HTTPS |
| **Mailjet** | 200/day, 6,000/month, 1,000 contacts, Mailjet branding, API + SMTP [7] | Single-sender validation is possible. Domain recommended. | Weak with a `@gmail.com` From (same DMARC issue) | `node-mailjet` | Yes, over HTTPS |
| **SendGrid** | **No free plan.** Retired from 28 May 2025. Now a one-time 60-day trial at 100/day, after which sending pauses [8] | — | — | — | Ruled out |
| **Amazon SES** | No SES-specific free allowance any more. New AWS accounts get credits, and the free plan lasts 6 months [9]. Sandbox: verified recipients only, 200 per 24 h, 1/sec [10] | Each From identity must be verified. Leaving the sandbox means a reviewed request with a website URL [10]. | Good with a domain. Poor with a gmail.com identity. | AWS SDK v3, verbose | Yes, over HTTPS |
| **Cloudflare Email Service (`send_email`)** | Sending to arbitrary recipients is **not available on Workers Free**. Workers Paid includes 3,000/month. Sends to *verified destination addresses* are free on all plans [11]. | **Yes.** The sender must be on a domain onboarded to Email Service [12] | Good with a domain | Native Workers binding | Workers only |
| **Postmark** | 100 emails/**month** on the Developer plan [13] | Yes, in practice | Excellent | Good SDK | Yes, over HTTPS |
| **MailerSend** | 500/month, 100 API requests/day, 1 domain [14] | Yes. The trial domain is for testing only. | Good with a domain | Good SDK | Yes, over HTTPS |

### Note: a `@gmail.com` From via a third-party service

Without a domain, Brevo, Mailjet and SES can only send "From: you@gmail.com" through their own servers. Those servers cannot DKIM-sign as `gmail.com` and aren't in Gmail's SPF, so DMARC alignment fails. Google's sender guidelines say "Don't impersonate Gmail From: headers" [15]. Today `_dmarc.gmail.com` publishes `p=none; sp=quarantine` (checked with `dig` on 2026-10-12). Mail isn't hard-rejected, but it is unauthenticated and spam-foldering is likely. Google has announced it will tighten this to quarantine (secondary sources [16]). For this project, then, those services only make sense once a domain exists.

## Recommendation

**Use Gmail SMTP from a dedicated Gmail account (for example `vu-reminders.<name>@gmail.com`), with a 16-character app password, via Nodemailer.**

- No domain or DNS work. It's free forever, and 500/day is about 5× our worst case.
- Deliverability to Members' Gmail inboxes is as good as person-to-person mail. Ask each Member to add the address to contacts or mark the first email "Not spam".
- Setup: turn on 2-Step Verification, then create an app password [17]. App passwords are **revoked whenever the account password changes** [17]. Keep the app password in an env var or secret store, and never log or commit it, the same as Members' LMS passwords.
- Use a dedicated account, not the operator's personal one, so a lockout or quota hit can't affect personal mail.

Hide the transport behind a small `sendEmail({ to, subject, html, text })` interface. Swapping providers later is then a config change.

**Upgrade path:**

1. **If the operator ever buys a domain (about $10/yr), switch to Resend.** It has the best SDK, works on edge runtimes, and gives you a branded From. Its 100/day is enough for 15 Members, but tight: alerts plus a Digest for 15 Members is roughly 45–90/day. **Brevo** (300/day) is the roomier alternative on the same domain.
2. **If the scheduler must run on Cloudflare Workers** (no SMTP from Nodemailer), use the **Gmail API over HTTPS** with an OAuth refresh token instead of SMTP. Gotcha: while the OAuth consent screen is in "Testing" status, refresh tokens expire after **7 days** [18]. Set the app to "In production". It can stay unverified for the operator's own single account, which shows a warning screen once at consent.

**Ruled out:** SendGrid (no free plan), Postmark and MailerSend (monthly caps too small), SES (sandbox plus domain plus review overhead, and no lasting free tier), Cloudflare Email Service (needs Workers Paid and a domain).

## Impact on the plan

- **Hosting:** the sender needs a Node runtime with outbound TCP on 465/587, such as GitHub Actions cron or Render. If the hosting ticket picks Cloudflare Workers, use the Gmail API path instead.
- **Secrets:** add a Gmail app password, or an OAuth client plus refresh token, to the deliberate secret-handling list alongside the LMS passwords.

## Sources

1. Gmail sending limits (personal): https://support.google.com/mail/answer/22839
2. Gmail API usage limits: https://developers.google.com/workspace/gmail/api/reference/quota
3. Resend pricing: https://resend.com/pricing
4. Resend domains: https://resend.com/docs/dashboard/domains/introduction
5. Resend `resend.dev` 403: https://resend.com/docs/knowledge-base/403-error-resend-dev-domain
6. Brevo free plan (secondary; brevo.com pricing and help pages returned 403/empty): https://dreamlit.ai/blog/brevo-review , https://www.palisade.email/email-deliverability/why-are-brevo-emails-going-to-spam
7. Mailjet pricing: https://www.mailjet.com/pricing/
8. Twilio changelog, SendGrid free plan changes: https://www.twilio.com/en-us/changelog/changes-coming-to-sendgrid-s-free-plans ; trial terms: https://www.twilio.com/docs/sendgrid/ui/account-and-settings/upgrading-your-plan
9. SES pricing: https://aws.amazon.com/ses/pricing/
10. SES sandbox: https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html
11. Cloudflare Email Service pricing: https://developers.cloudflare.com/email-service/platform/pricing/
12. Cloudflare send_email from Workers: https://developers.cloudflare.com/email-routing/email-workers/send-email-workers/ ; Workers block port 25: https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/
13. Postmark pricing: https://postmarkapp.com/pricing
14. MailerSend pricing: https://www.mailersend.com/pricing
15. Google email sender guidelines: https://support.google.com/a/answer/81126
16. Gmail DMARC policy change (secondary): https://www.valimail.com/blog/the-gmail-dmarc-policy-update-you-may-not-know-about/
17. Google app passwords: https://support.google.com/accounts/answer/185833
18. Google OAuth 2.0, refresh token expiry: https://developers.google.com/identity/protocols/oauth2
