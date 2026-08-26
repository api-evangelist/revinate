---
name: revinate-display-widget-feed
description: Fetch Revinate reviews that are licensed for public display or republishing — the only Porter API operations whose content may be shown to an end user.
api: Revinate Porter API
base_url: https://porter.revinate.com
operations:
  - getWidgetReviewsByHotelId
  - listWidgetReviews
---

# Display-licensed review feed

Use this skill whenever review content will be **seen by a person** — a website widget, a booking
page, a marketing surface, an emailed digest, or an agent response quoted back to a user.

## Why this skill exists

The Porter API has two review streams and they are not interchangeable:

| Stream | Operations | May display? | Coverage |
|---|---|---|---|
| Review Stream | `listReviews`, `getReviewsByHotelId`, `getReviewsForHotelSet`, `getCompetitorReviewsByHotelId` | **No** — analysis only | All 100+ sources |
| Widget Review Stream | `listWidgetReviews`, `getWidgetReviewsByHotelId` | **Yes** | Excludes TripAdvisor and Yelp |

The widget stream is a strict subset by source. Revinate withholds TripAdvisor and Yelp content
here because of content-licensing agreements with those partners.

## Steps

1. **Resolve the hotel id** via `listHotels`, reading ids from `links`.

2. **Fetch display-licensed reviews.** Call `getWidgetReviewsByHotelId`
   (`GET /hotels/{hotelId}/widgetreviews`) for one property, or `listWidgetReviews`
   (`GET /widgetreviews`) across every accessible property.

3. **Page and filter** exactly as in `revinate-review-extract` — `page`, `size` (max 1000), `sort`,
   `date`. Stop when the returned collection is empty; an over-range page returns empty, not an
   error.

4. **Render.** Show `title`, `body`, `author`, `rating`, `dateReview`, and the source
   `reviewSite.name`. Where the hotel has replied, `response` carries the management reply — render
   it alongside the review rather than as a separate item.

## Rules

- **Never substitute the analysis stream.** If a review you want is missing from the widget feed, it
  is almost certainly TripAdvisor or Yelp content, and it is missing on purpose. Do not fall back to
  `listReviews` to fill the gap — that is a licence breach, not a workaround.
- Do not imply the feed is exhaustive. If you show a review count or an average, say it is based on
  the display-licensed subset, or take the aggregate figures from `getReviewsSnapshotByHotelId`
  instead and label them as full-coverage metrics.
- Attribute the source site. `reviewSite.name` and `reviewSite.mainUrl` are provided for this.
- This stream may still contain survey-sourced reviews with guest PII under `guestStay.guest`. Never
  render guest email, phone, postal address, loyalty id or confirmation code.
