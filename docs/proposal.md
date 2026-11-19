# Proposal: Watchlist and weekly email digest

Status: draft for review
Author: radar maintainers

## Context

Teams use the radar to pick mini-program frameworks and libraries, then rarely come back. They miss it when a resource they depend on changes status, for example when a GitHub repo is archived and enrichment moves it to `hold`.

We already produce a weekly report (`lib/weekly.ts`) every week through the cron route `app/api/cron/weekly/route.ts`, and enrichment (`lib/enrichment.ts`) already detects status and risk changes. This proposal lets a user follow specific resources and receive one email a week about what changed for them.

## Goals

- A visitor can follow up to 30 resources by entering an email address, without creating an account.
- Every week, each subscriber receives one email listing status, risk and maintenance changes for the resources they follow, with a link to the full weekly report.
- Subscribers can unsubscribe with one click.

## Non-goals

- Accounts or login.
- Real-time alerts. Weekly is enough.

## Design

### 1. Subscribing

A "关注" (follow) button on each resource page and in the radar list adds the resource to a local watchlist in the browser. A small panel asks for an email address and sends:

`POST /api/watchlist { email, resourceIds }`

The route checks that the email looks like an email address and that every resource id exists, then saves the subscription. It is rate-limited with the existing `rateLimit` helper (`lib/rate-limit.ts`), 10 requests per hour per IP, like the advisor.

The subscription is active immediately, so the user starts getting emails from the next week.

### 2. Storage

Subscriptions are stored in Upstash Redis as one hash per subscriber, `watch:<email>`, holding the resource ids and the date of the last email sent, plus a set `watch:subscribers` of all emails.

When Redis is not configured, we use the same in-memory fallback that `lib/rate-limit.ts` already uses. That keeps local development and small self-hosted deployments working without extra setup.

### 3. Sending the digest

We extend the existing weekly cron route. After it creates and persists the weekly report, it:

1. reads all emails from `watch:subscribers`,
2. for each subscriber: loads their watchlist, calls `getResources()` and the latest enrichment snapshot, finds changes since `lastSentAt`, renders the email and sends it through the email provider's API,
3. updates `lastSentAt`.

This runs inside the same cron request, one subscriber at a time, so we never send two emails to the same person at once.

The weekly cron route is already protected against overlapping runs by `acquireTaskLock("cron.weekly")` (`lib/task-lock.ts`). So even if the scheduler calls it twice, the digest is sent only once per week.

### 4. Unsubscribing

Every email ends with a link:

`https://<site>/api/watchlist/unsubscribe?email=<email>`

Opening it deletes the subscriber's hash and removes them from `watch:subscribers`, then shows a confirmation page.

### 5. Email provider

We use a transactional email API (Resend or similar) with one new environment variable, `EMAIL_API_KEY`. Without it, the cron route logs the emails it would have sent instead of sending them.

## Rollout

1. Ship the follow button and `POST /api/watchlist`, with sending switched off.
2. Turn on sending for maintainers' own addresses for two weeks.
3. Announce the feature on the site and in the weekly report.

## Metrics

- Number of active subscribers.
- Unsubscribe rate per email.
- Click-through from the email to resource pages.

## Open questions

- Should we also send an immediate email when a followed resource moves to `hold`?
- Do we need a privacy notice for storing email addresses?
