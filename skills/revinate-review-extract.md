---
name: revinate-review-extract
description: Paginate a hotel's review history out of the Revinate Porter API for offline analysis, applying rating, sentiment, response-status and language filters safely.
api: Revinate Porter API
base_url: https://porter.revinate.com
operations:
  - listHotels
  - listLanguages
  - getReviewsByHotelId
  - listReviews
  - getReview
---

# Extract reviews for offline analysis

> **Licence warning — read this first.** Content from these operations **cannot be used for display
> purposes** due to Revinate's content restrictions. It is for offline analysis only. If the output
> will be seen by an end user, stop and use `revinate-display-widget-feed` instead.

## Steps

1. **Resolve the hotel id** with `listHotels`, reading ids from the `links` array.

2. **Resolve language slugs (optional).** If filtering by language, call `listLanguages`
   (`GET /languages`) first — `languageSlug` only accepts slugs from that endpoint.

3. **Page through the reviews.** Call `getReviewsByHotelId`
   (`GET /hotels/{hotelId}/reviews`), or `listReviews` (`GET /reviews`) for every accessible hotel
   at once.

   Pagination and filtering parameters:

   | Parameter | Meaning |
   |---|---|
   | `page` | zero-based page number |
   | `size` | page size — **default 5 for reviews**, maximum 1000 |
   | `sort` | `{field},ASC` or `{field},DESC` |
   | `date` | `{startEpoch}..{endEpoch}`, epoch seconds |
   | `ratings` | comma-separated 1-5, e.g. `ratings=4,5` |
   | `feedback` | `positive`, `neutral`, or `negative` |
   | `response` | `none` or `posted` |
   | `languageSlug` | a slug from `listLanguages` |

4. **Terminate correctly.** Paging past the end **does not error** — it returns an empty result set.
   Loop until the returned collection is empty, or until `page.number + 1 >= page.totalPages`. Do
   not wait for an error status; you will loop forever.

5. **Read each review.** `ReviewRep` carries `title`, `body`, `author`, `dateReview`,
   `dateCollected`, `rating`, `nps`, `subratings`, `tripType`, the source `reviewSite`, the
   `language`, any management `response`, and for survey-sourced reviews a `guestStay` and
   `surveyTopics`. Use `getReview` (`GET /reviews/{reviewId}`) only to re-read a single review.

## Rules

- **Set `size` explicitly.** The default is 5 for reviews. Leaving it unset turns a 10,000-review
  extract into 2,000 requests.
- **Respect the review limit.** A per-key review limit exists and its value is not published — if
  `page` and `size` reach beyond it the endpoint returns an error rather than an empty page. Narrow
  with `date` and extract in time windows rather than paging ever deeper.
- **Pagination and sorting are entitlements.** They work only "if your API key is configured for
  it". If `sort` appears to be ignored, that is a provisioning question for the account manager, not
  a bug to retry around.
- **Treat payloads as personal data.** Survey-sourced reviews embed identified guest PII —
  `firstName`, `lastName`, `email`, `phone`, postal address, `loyaltyId` and `confirmationCode`
  under `guestStay.guest`. Store accordingly and do not echo these fields into logs or summaries.
- No rate limits are published and no rate-limit headers are returned. Pace conservatively and back
  off exponentially on any non-2xx.
