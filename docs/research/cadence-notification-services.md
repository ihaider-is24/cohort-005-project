# Third-party notification services for Cadence

**Research date:** 12 November 2026  
**Scope:** First-release in-app notifications for Cadence Students and Instructors: purchase, enrollment, refund, and course-question events. Cadence owns the experience and domain actions; a provider may supply event workflows, feed persistence, and live updates. Email, push, SMS, promotional campaigns, and admin notifications are out of scope.

## Recommendation

Shortlist **Novu Cloud** and **Knock** for a short integration spike. Keep Cadence's own event/outbox record as the source of truth, and use a provider's feed as the notification delivery/read-state store only after confirming retention, deletion, and regional-processing requirements.

Novu is the best initial fit if its published EU region and low entry price meet Cadence's procurement and retention needs. Its current pricing gives only 24-hour feed retention on Free, 7 days on Pro, and 90 days on Team, so confirm whether this is enough for a persistent inbox before selecting a plan. Knock offers a strong React/Node path, configurable feed lifetime, real-time counts and read state, and trigger idempotency; however, its documentation says EU data residency is still forthcoming (expected by the end of 2026), which is a material issue for a Europe-based product.

OneSignal is not a leading candidate for this requirement: its in-app product centers on contextual in-app messages, and its pricing page does not describe an inbox feed with persistent per-user read/archive state. It is more relevant if Cadence later adds push or campaign messaging. A provider-hosted UI is useful as a spike baseline, but the production experience should use a headless/custom feed so Cadence controls action wording, navigation, accessibility, and visual design.

## Comparison

| Provider | Persistent in-app inbox | UI and stack fit | Delivery and operations | Data controls | Published price fit |
| --- | --- | --- | --- | --- | --- |
| **Novu Cloud** | Inbox supports real-time notifications and read/archive/snooze actions. Feed retention is 24 hours (Free), 7 days (Pro), 90 days (Team); Enterprise offers custom retention. | Official React Inbox plus headless hooks; a TypeScript workflow SDK and server-side trigger path fit Node. Cadence can embed the default Inbox for evaluation, then use headless hooks for its own UI. | Stored in Novu and pushed over websocket; workflow activity and failed-message webhooks support investigation. Its current delivery retry documentation says provider sends are attempted once (no automatic retry or provider failover), while outbound webhook callbacks retry up to 8 times. In-app delivery avoids a downstream messaging provider. | Pricing advertises US and EU data residency. Novu publishes a DPA/SCC and SOC 2 Type II / ISO 27001 materials. Select the account region deliberately and verify where all feed, logs, and backups reside contractually. | Free: 10,000 workflow runs/month. Pro from **$30/month**, 30,000+ runs and 7-day activity feed. Team from **$250/month**, 250,000+ runs and 90-day activity feed. Exact charge depends on what counts as a run and selected plan. |
| **Knock** | Feed API and components provide persistent per-recipient feeds, read/seen/archive tracking, badge counts, and real-time updates. Feed items are available indefinitely; message logs are retained 30 days on non-Enterprise plans (90 days Enterprise). | Official `@knocklabs/react` feed components and headless React hooks; official Node SDK and API fit the React/Node stack. Cadence can use the prebuilt feed or create its own. | Trigger endpoint supports `Idempotency-Key` for 24 hours, helping Cadence safely retry event submission. Knock documents real-time websocket support, feed status tracking, and workflow/message logs. | Knock reports SOC 2 Type 2 and GDPR credentials, and offers enhanced client security with signed user tokens. Current plan docs state EU data residency is not yet available and is expected by end of 2026. Review DPA, subprocessors, retention, and deletion behavior before sending personal or course data. | Free: 10,000 messages/month. Starter: **$250/month** for 50,000 messages, then $0.005/message; enterprise custom. A message is one successful user/channel delivery, so in-app-only activity is easier to forecast than multichannel use. |
| **OneSignal** | In-app messaging supports contextual in-product messages. Current public materials reviewed do not establish a durable inbox feed with the feed lifecycle Cadence needs. Treat this as a gap to validate, not a confirmed absence. | React/Node integrations can trigger messages, but the strongest first-party materials focus on mobile in-app messaging and push/email channels rather than a custom web inbox. | Mature multichannel platform with a free tier and usage-priced Growth plan. The pricing page includes message history features at paid tiers; clarify whether those meet recipient-facing inbox requirements. | Pricing advertises GDPR compliance and DPA on paid plans; evaluate exact residency, retention, and deletion terms against Cadence requirements. | Free tier available. Growth starts at **$19/month**; mobile in-app messaging is included with paid mobile push based on MAU, but this does not establish a suitable inbox price or design fit. |

## Cost model and assumptions

For a small launch, assume **2,000 monthly active Students/Instructors** and **8 in-app notifications per active user/month**: 16,000 recipient-message deliveries, one in-app channel, no external email/push sends, no high-volume burst requirement. This is an illustrative planning case, not a vendor quote.

- **Novu:** 16,000 workflow runs exceeds the 10,000 Free allowance. Pro starts at $30/month and includes 30,000+ runs, so the likely planning floor is **$30/month**, with 7-day feed retention. If a 90-day feed is required, Team starts at **$250/month**. Check that one Cadence event maps to one billable workflow run and that retries/steps are not separately metered.
- **Knock:** 16,000 successful in-app messages exceeds the 10,000-message Free allowance. Starter is **$250/month** and includes 50,000 messages, so the planning price is **$250/month**. The paid plan's message-log retention is 30 days, while feed items persist indefinitely. Confirm if message log retention affects support/debugging or retention obligations.
- **OneSignal:** At the assumed web-only launch, a precise cost cannot be established from the pricing page because Growth variable costs are tied to web-push subscribers and email sends, while in-app is included with mobile push. Do not treat its $19 base as the cost of a Cadence inbox without confirming the product and entitlement.

Costs exclude VAT, negotiated discounts, setup/support, and downstream channel fees. If the actual feed volume is at or below 10,000 messages/runs monthly, Novu Free or Knock Free may work for a pilot, subject to retention and feature limits.

## Event delivery and integration shape

Regardless of vendor, do not make the provider the only record that a purchase, enrollment, refund, or course question occurred. In the same SQLite transaction as each successful domain action, persist a small notification-outbox row with a stable event ID, recipient ID, event type, minimal entity/action identifiers, creation time, and dispatch state. A worker can submit it to the vendor and retry with the stable ID. This avoids losing an actionable notification if provider submission is temporarily unavailable; provider idempotency prevents duplicates where supported. Keep payments, refund state, course content, and question text in Cadence and link the notification to an authorized Cadence route.

For Knock, pass a stable idempotency key to workflow triggers. For Novu, validate its current trigger deduplication guarantees and put delivery retries under Cadence's outbox worker unless the chosen integration explicitly supplies equivalent guarantees. For either service, test identity binding, unread counts, pagination, user deletion, and delayed/out-of-order events. In the browser, never expose provider secret keys; issue signed, short-lived client credentials or use the vendor's supported secure identity mechanism. Only send opaque Cadence user IDs and the minimum templating data needed.

## Unknowns that could change the ranking

1. Required user-visible feed history and retention/deletion period (days, indefinite, or user-configurable).
2. Whether EU-only processing/residency is a hard requirement, including backups and support access.
3. Expected monthly recipients, notifications per recipient, peak event rate, and whether retries count toward usage.
4. Whether Cadence needs only in-app delivery or expects email/push fallbacks soon.
5. Whether feed items may contain course/question text, purchase details, or other personal data; recommended default is identifiers and brief neutral copy only.
6. Whether course-question updates need grouping/threading, per-event preferences, localization, or separate Student/Instructor feed views.
7. Operational tolerance for provider outages and the acceptable delay before in-app events appear.

## Sources (first-party)

### Novu

- [Pricing and plan comparison](https://novu.co/pricing) — workflow run allowances, Inbox/feed retention, regions, and plan prices.
- [Inbox setup and React integration](https://docs.novu.co/platform/inbox/setup-inbox) — drop-in Inbox, subscriber binding, React installation.
- [Inbox headless mode](https://docs.novu.co/platform/inbox/headless-mode) — custom UI using React hooks, state, and actions.
- [Delivery retry policy](https://docs.novu.co/platform/developer/delivery-retries) — in-app storage/websocket delivery, provider-send retry policy, webhook retry policy, and failure handling.
- [Security and compliance](https://docs.novu.co/platform/additional-resources/security) and [Novu DPA](https://novu.co/dpa) — certifications and data-processing terms.

### Knock

- [Pricing](https://knock.app/pricing) — message allowances, prices, feed retention, and plan features.
- [React feed component](https://docs.knock.app/in-app-ui/react/feed) and [custom feed UI](https://docs.knock.app/in-app-ui/feeds/custom-ui) — prebuilt and custom React approaches.
- [In-app UI overview](https://docs.knock.app/in-app-ui/overview) — websocket support, counts, and read/seen/archive state.
- [Workflow triggering](https://docs.knock.app/send-notifications/triggering-workflows/api) and [API idempotency](https://docs.knock.app/api-reference/overview) — safe retry behavior for workflow triggers.
- [Data retention](https://docs.knock.app/manage-your-account/data-retention) — message-log and feed-item retention behavior.
- [Security](https://docs.knock.app/developer-tools/security) and [plans and usage](https://docs.knock.app/manage-your-account/knock-plans) — security posture and current EU residency status.

### OneSignal

- [Pricing](https://onesignal.com/pricing) — current plan prices and channel metering.
- [In-app messaging](https://onesignal.com/in-app) — product positioning and web/mobile in-app capabilities.

