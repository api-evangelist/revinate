---
name: revinate-reputation-snapshot
description: Produce a reputation snapshot for one hotel from the Revinate Porter API — review volume, average rating, TripAdvisor rank, and sentiment broken down by operational topic category.
api: Revinate Porter API
base_url: https://porter.revinate.com
operations:
  - listHotels
  - getHotel
  - getReviewsSnapshotByHotelId
  - getSentimentSnapshotByHotelId
  - getSentimentBreakdownByHotelIdAndTopicCategoryId
---

# Reputation snapshot for a hotel

## Before you start

Every request needs four headers. Compute them fresh for each call:

- `X-Revinate-Porter-Username` — a username with access to the hotel and to the API key
- `X-Revinate-Porter-Timestamp` — epoch seconds, must be within **5 minutes** of now
- `X-Revinate-Porter-Key` — the API key issued by Revinate
- `X-Revinate-Porter-Encoded` — `HMAC_SHA256(apiSecret, username + timestamp)`, hex

The timestamp is part of the signed string. If you refresh the timestamp you **must** recompute the
signature. Reusing an old signature with a new timestamp returns `401`.

## Steps

1. **Find the hotel.** Call `listHotels` (`GET /hotels`) and match on `name` or `slug`. Ids are not
   returned as a property — read them from the `links` array on each entry. If you already hold an
   id, call `getHotel` (`GET /hotels/{hotelId}`) to confirm access before doing more work.

2. **Pull the review metrics.** Call `getReviewsSnapshotByHotelId`
   (`GET /hotels/{hotelId}/reviewssnapshot`). This returns `aggregateValues`, `valuesByReviewSite`
   and `valuesByTime` — new review volume, average rating, and TripAdvisor rank, plus the
   per-source breakdown. Narrow the period with `date={startEpoch}..{endEpoch}` (epoch seconds).

3. **Pull category sentiment.** Call `getSentimentSnapshotByHotelId`
   (`GET /hotels/{hotelId}/topiccategoriessnapshot`). Each `TopicStatisticsRep` carries `text`,
   `score`, `scorePercentChange`, and mention counts split positive / neutral / negative.

4. **Drill into a weak category.** For any category whose `score` is low or whose
   `scorePercentChange` is falling, call `getSentimentBreakdownByHotelIdAndTopicCategoryId`
   (`GET /hotels/{hotelId}/topicssnapshot`) to get the child topics beneath it, and read
   `tokenCounts` for the specific words guests used.

5. **Report.** Lead with the direction of travel (`scorePercentChange`), not the absolute score —
   the absolute value is only meaningful against the competitive set. See
   `revinate-competitive-benchmark`.

## Rules

- **Do not display anything you pulled here.** Snapshot metrics are aggregates and are safe to
  report, but if you also fetched raw reviews to illustrate a point, review text from
  `getReviewsByHotelId` / `listReviews` is licensed for **offline analysis only** and must never be
  shown to an end user. Use `revinate-display-widget-feed` for anything user-facing.
- Use the same `date` window on every call in one snapshot, or the numbers will not reconcile.
- On `401`, re-derive the timestamp and signature together and retry once. On a repeated `401`, the
  credentials or the key's access permissions are wrong — stop and escalate to a human.
- There is no request-id header. If you need Revinate support to investigate, record the exact URL,
  epoch timestamp and response body yourself.
