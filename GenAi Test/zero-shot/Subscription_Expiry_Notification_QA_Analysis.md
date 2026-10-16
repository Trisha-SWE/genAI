# Requirement Analysis — Pre-Test-Design Review

> **Requirement:** *"The system should send a notification to the user when their subscription is about to expire, so they can renew it in time."*

---

## 1. Ambiguous Terms

| # | Ambiguous Term | Why It's Ambiguous | Question for the PO |
|---|---|---|---|
| 1.1 | **"about to expire"** | No concrete timeframe. Could mean 30 days, 7 days, 24 hours, or all of the above at once. | *What is the exact lead time — or the set of lead times — that should trigger a notification? Is it a single alert or a sequence (e.g., 30d → 7d → 1d)?* |
| 1.2 | **"notification"** | Ambiguous channel. Could be email, in-app alert, SMS, push notification, or any combination. | *Which delivery channels are in scope? Is the channel fixed by the system or configurable per user preference?* |
| 1.3 | **"the user"** | Assumes one account = one user. Doesn't address shared/team accounts or accounts managed by an admin. | *If a subscription belongs to an organization with multiple members, who receives the notification — the account owner, all members, or a designated billing contact?* |
| 1.4 | **"send"** | Does "send" mean dispatch only, or does it imply confirmed delivery? Does a failed send count as fulfilled? | *Is the system's obligation to attempt the send, or to ensure the notification is received? What happens on delivery failure?* |
| 1.5 | **"in time"** | Subjective. The lead time is the entire basis of testability, and this phrase doesn't define it. | *What lead time is considered "enough" time to renew? Is this defined by a business SLA or a fixed window?* |

---

## 2. Missing Conditions / Unhandled Branches

| # | Missing Condition | Risk If Unaddressed | Question for the PO |
|---|---|---|---|
| 2.1 | **User renews before the notification fires** | Notification sent for an already-active subscription → confusing UX. | *If the user renews before a scheduled notification window, should the notification be suppressed? What is the suppression logic?* |
| 2.2 | **Subscription is already expired** | The scheduler might catch an already-lapsed sub and send an "about to expire" notice retroactively. | *If the subscription has already expired at the time of the scheduled check, should an expiry-warning or a different "your subscription has lapsed" message be sent, or nothing at all?* |
| 2.3 | **Subscription is cancelled (not expired)** | A cancelled subscription has an end date too. Should cancellation trigger the same notification? | *Does a user who has explicitly cancelled their subscription receive expiry-approaching notifications, or does cancellation suppress them?* |
| 2.4 | **Auto-renewal is enabled** | If the system will automatically renew, a "you need to renew" alert is misleading or redundant. | *Should expiry notifications be suppressed when auto-renewal is active? If still sent, must the message acknowledge that auto-renewal will handle it?* |
| 2.5 | **User has multiple subscriptions** | Each subscription has its own expiry date. The requirement says "their subscription" (singular). | *If a user holds more than one active subscription, is each notified independently with its own alert?* |
| 2.6 | **User with no expiry date (lifetime / free tier)** | A free or lifetime account has no expiry; the scheduler must not send a false alert. | *How should the system handle accounts with no expiry date — explicitly excluded, or does the feature only apply to paid subscriptions?* |
| 2.7 | **Notification channel is unavailable** | Email bounces, push token is invalid. No fallback is defined. | *Is there a retry policy? Is there a fallback channel? How are failed deliveries logged and surfaced?* |
| 2.8 | **User has opted out of all notifications** | Opt-out preferences are not mentioned. Sending regardless would likely breach GDPR/CAN-SPAM. | *Must the system respect global and channel-level opt-out preferences? Are transactional subscription notifications exempt from marketing opt-outs?* |

---

## 3. Unstated Business-Rule Assumptions

| # | Hidden Assumption | Why It Must Be Surfaced | Question for the PO |
|---|---|---|---|
| 3.1 | **Notification windows are fixed and system-wide** | The requirement implies one universal schedule, but different plans (monthly vs. annual) may warrant different windows. | *Are notification lead times the same regardless of subscription duration (e.g., 7 days for both a monthly and a yearly plan)? Or should lead times scale with plan length?* |
| 3.2 | **One notification per window, no duplicates** | If the scheduler runs multiple times a day, will it fire multiple alerts for the same window? Deduplication logic is not specified. | *Must the system guarantee exactly one notification per configured window per subscription cycle? What is the deduplication strategy?* |
| 3.3 | **Timezone handling** | "About to expire" depends entirely on which timezone is used for the calculation. A subscription expiring at 00:00 UTC may be yesterday's date for a user in UTC+5. | *Should notification timing be calculated against the user's local timezone, the system's timezone, or the timezone of purchase?* |
| 3.4 | **The renewal link / CTA is implied** | The requirement says "so they can renew" but does not mandate that the notification includes a renewal link. | *Must the notification include a direct link to the renewal flow? If so, should the link be pre-authenticated or deep-linked to the specific plan?* |
| 3.5 | **Idempotency across system restarts or retries** | If the scheduler restarts mid-run, it could trigger a second send for the same window. | *Is there a requirement for idempotent sends — i.e., if the system processes the same expiry event twice, only one notification is ever sent?* |
| 3.6 | **Content and template ownership** | The message content (wording, branding, localisation) is completely unspecified. | *Who owns the notification template? Must content be localised to the user's language? Are there legal disclaimers required in the message body?* |

---

## 4. Not Measurably Verifiable (As Written)

| # | Unverifiable Element | Why It Cannot Be Tested In Its Current Form | Question for the PO |
|---|---|---|---|
| 4.1 | **"should send"** | "Should" is a weak modal. It implies the behaviour is recommended, not mandatory. This makes pass/fail criteria undefined. | *Is this a MUST (mandatory) behaviour? Can we restate this as "the system SHALL send"?* |
| 4.2 | **"in time"** | There is no measurable definition of "in time." A notification sent 1 minute before expiry technically satisfies the literal wording. | *What is the minimum acceptable lead time for the notification to be considered delivered "in time"? Is this tied to payment processing windows?* |
| 4.3 | **"a notification" (singular)** | Does this mean exactly one notification total, or at least one? A single notification in a 30-day window may not be "enough time" for all users. | *Is a single notification per expiry cycle sufficient, or is a multi-touch sequence (30d, 7d, 1d) required? What is the minimum acceptable notification count?* |
| 4.4 | **Success of delivery** | "Send" is verifiable at dispatch, but there is no acceptance criterion tied to delivery confirmation, open rate, or user action. | *What is the system's definition of a successful notification — dispatch logged, delivery confirmed by provider, or user acknowledged?* |

---

## Summary — 14 Questions for the Product Owner

1. What exact lead time(s) trigger a notification — and is it a single alert or a multi-step sequence?
2. Which delivery channels are in scope, and can users configure their preferences?
3. Who is notified on shared/team accounts — owner, all members, or billing contact?
4. What happens on delivery failure — retry, fallback channel, or silent fail?
5. Is the notification suppressed if the user renews before the window fires?
6. Does a cancelled or auto-renewing subscription still trigger this notification?
7. How are accounts with no expiry date (free/lifetime) excluded?
8. Must the system respect opt-out preferences, and is this notification transactional or marketing?
9. Do lead times differ by plan length (monthly vs. annual)?
10. What timezone governs the notification trigger?
11. Must the notification include a direct renewal link, and should it be pre-authenticated?
12. What is the deduplication/idempotency guarantee if the scheduler runs multiple times?
13. Can "should" be upgraded to "shall" to make this a hard requirement?
14. What constitutes a "successful" notification — dispatch, delivery, or user acknowledgement?

---

*Generated by senior QA review · Subscription Expiry Notification requirement · 2026-10-10*
