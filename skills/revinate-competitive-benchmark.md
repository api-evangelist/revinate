---
name: revinate-competitive-benchmark
description: Benchmark a hotel against its Revinate-designated competitive set using competitor hotels, competitor reviews and review snapshots.
api: Revinate Porter API
base_url: https://porter.revinate.com
operations:
  - getHotel
  - getCompetitorHotelsByHotelId
  - getCompetitorReviewsByHotelId
  - getReviewsSnapshotByHotelId
---

# Competitive benchmark

A reputation score means little in isolation. This skill puts a property's numbers next to the
competitive set Revinate has configured for it.

## Steps

1. **Confirm the subject hotel.** Call `getHotel` (`GET /hotels/{hotelId}`). Note `name`, `city`,
   `country`, `accountType` and `tripAdvisorId`.

2. **Get the competitive set.** Call `getCompetitorHotelsByHotelId`
   (`GET /hotels/{hotelId}/competitorhotels`). This is the comp set configured in Revinate for this
   property — it is not something you choose. Read competitor ids from the `links` array.

3. **Baseline the subject.** Call `getReviewsSnapshotByHotelId`
   (`GET /hotels/{hotelId}/reviewssnapshot`) with an explicit `date={startEpoch}..{endEpoch}`
   window.

4. **Pull competitor reviews.** Call `getCompetitorReviewsByHotelId`
   (`GET /hotels/{hotelId}/competitorreviews`) with the **same** `date` window. Page until the
   result set is empty.

5. **Compare.** Derive competitor rating averages and review volume from the returned reviews and
   set them against the subject's `aggregateValues`. Where a competitor hotel id is also directly
   accessible to your key, prefer its own `getReviewsSnapshotByHotelId` — Revinate's aggregation is
   more reliable than one you compute from a paged sample.

## Rules

- **Analysis only.** `getCompetitorReviewsByHotelId` is part of the analysis stream. Competitor
  review text must never be displayed, quoted publicly, or surfaced to an end user — this is both a
  licence restriction and an obvious commercial hazard. Report derived aggregates, not competitor
  review bodies.
- **Use one `date` window everywhere.** Mismatched windows are the most common way this comparison
  goes silently wrong.
- **Say when the sample is partial.** If a per-key review limit or an unentitled `sort` truncated
  your competitor pull, state that the comparison is based on a partial sample rather than
  presenting it as complete.
- Competitor coverage depends on what the key is entitled to. A `403` on a competitor id means the
  key lacks access — record it as unavailable rather than treating it as zero reviews.
