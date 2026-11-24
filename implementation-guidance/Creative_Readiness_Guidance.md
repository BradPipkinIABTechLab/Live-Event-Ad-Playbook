# Live Event Ad Playbook (LEAP)
# Creative Readiness Implementation Guidance

**LEAP Implementation Guidance v1.0 | Ad Management API v1.1**

> **Scope:** This guidance explains how the existing IAB Tech Lab Ad Management API (AMA) v1.1 can support creative submission and audit management ahead of live events. It does not amend AMA v1.1, the LEAP Forecast API, the Concurrent Streams API, or AdCOM, and it does not establish a new creative-readiness protocol. Supply-partner policies and delivery-path requirements remain applicable.

## Table of Contents

- [1. Purpose and scope](#1-purpose-and-scope)
- [2. How creative preparation fits the LEAP lifecycle](#2-how-creative-preparation-fits-the-leap-lifecycle)
- [3. What AMA v1.1 supports today](#3-what-ama-v11-supports-today)
- [4. Preparing creatives for an upcoming event](#4-preparing-creatives-for-an-upcoming-event)
- [5. Submitting and maintaining ads through AMA v1.1](#5-submitting-and-maintaining-ads-through-ama-v11)
- [6. Interpreting audit status](#6-interpreting-audit-status)
- [7. Audit approval, eligibility, and delivery readiness](#7-audit-approval-eligibility-and-delivery-readiness)
- [8. Live-event operational timeline](#8-live-event-operational-timeline)
- [9. Identity and reconciliation](#9-identity-and-reconciliation)
- [10. Worked interaction outline](#10-worked-interaction-outline)
- [11. Known limitations and future work](#11-known-limitations-and-future-work)
- [12. References](#12-references)

## 1. Purpose and scope

Live events create short, high-concurrency advertising opportunities that may be difficult to recover if an intended creative is rejected or cannot be delivered when an ad break begins. Creative preparation therefore belongs in pre-event planning, alongside forecast review, campaign configuration, and infrastructure preparation.

AMA v1.1 provides an existing standardized means for a demand partner to submit an AdCOM `Ad` to a supply partner and receive feedback on its audit status. By using that process **before** a live event, participants can identify and remediate approval issues while time remains. This guide describes that established workflow, not a new requirement that every LEAP participant implement AMA.

This document is intended for buyers/DSPs and sellers/SSPs, publishers, ad servers, and insertion partners where an AMA integration is supported. AMA's terms *bidder* and *exchange* can be understood more generally as *demand partner* and *supply partner*, respectively. An organization may occupy different roles across different integrations.

**Source of truth:** The [AMA v1.1 specification](https://github.com/InteractiveAdvertisingBureau/AdManagementAPI/blob/master/Ad%20Management%20API%20v1.md) governs its endpoints, data structures, authentication choices, status behavior, and pagination. The [AdCOM specification](https://github.com/InteractiveAdvertisingBureau/AdCOM) governs its `Ad` and `Audit` objects. The [LEAP Common API Guidance](Common_API_Guidance.md) provides context for the LEAP APIs but does not override AMA-specific conventions. All operational recommendations below are non-normative.

## 2. How creative preparation fits the LEAP lifecycle

| Stage | Existing interface or activity | Practical outcome |
|---|---|---|
| Pre-event discovery and planning | [Forecast API](../Forecast_API.md) | Discover upcoming events and review anticipated streams and inventory characteristics. |
| Pre-event creative preparation | **AMA v1.1**, where supported | Submit intended ads, obtain audit results, address rejections or corrections, and confirm applicable seller policies. |
| Pre-event delivery coordination | Existing partner operational processes | Confirm any required ingestion, transcoding, caching, policy review, or insertion-path preparation; **AMA approval alone does not confirm completion of those activities**. |
| Live event | [Concurrent Streams API](../Concurrent_Streams_API.md) and OpenRTB | Use current stream-count information for live planning and adjustments while observing the supply partner's creative eligibility and delivery rules. |
| Post-event | Existing reporting and partner review | Review approval issues, blocked creatives, changes, and lead times to improve the next event. |

The Forecast API and Concurrent Streams API **do not convey AMA creative audit results**. AMA v1.1 **does not convey LEAP event forecast or concurrent stream counts**. Their coordination is an implementation workflow, not an implied join or new interface between the APIs.

## 3. What AMA v1.1 supports today

AMA v1.1 defines established, reusable mechanisms:

| Capability | Existing AMA v1.1 mechanism | Live-event use |
|---|---|---|
| Submit one creative | `POST {baseUrl}/bidder/{bidderId}/ads` with an AdCOM `Ad` | Start an audit ahead of the intended event. |
| Read one creative | `GET {baseUrl}/bidder/{bidderId}/ads/{id}` | Confirm the supply partner's latest record for a particular ad. |
| Retrieve audit changes | `GET {baseUrl}/bidder/{bidderId}/ads?auditStart={timestamp}` | Reconcile changes to audit status across many creatives. |
| Update or request re-audit | `PUT` or `PATCH {baseUrl}/bidder/{bidderId}/ads/{id}` | Correct submitted material or touch an eligible ad for re-audit/reactivation according to seller policy. |
| Receive change notifications | Optional AMA webhook | Reduce polling where the supply partner supports a prearranged webhook integration. |
| Review corrections and feedback | AdCOM `audit.status`, `audit.feedback`, `audit.corr`, `audit.lastmod` | Identify material audit results and take appropriate action. |

The supply partner provides its base URL, bidder ID, authentication protocol, rate limits, attribute requirements, bidding policy, ad-ID policy, retention rules, and other integration details. AMA v1.1 deliberately leaves many commercial and policy decisions to the integrated parties. **No specific OAuth method, webhook availability, approval lead time, or transcoding workflow is mandated by this guide.**

## 4. Preparing creatives for an upcoming event

A buyer that learns of an upcoming live event through the Forecast API can begin selecting likely creatives rather than waiting for the first live bid request. Useful planning inputs include:

- **Event timing and certainty:** `scheduledstart`, `scheduledend`, `eventstatus`, and `flexibleend` can inform preparation windows. An event may change, so preparation schedules should account for updates.
- **Inventory characteristics:** When present, `inventoryconfig` fields such as `supportedmtype`, `totaladdurationsec`, and `expectedpodcount` provide planning context. An average pod duration or total available ad time **does not establish a per-creative maximum**.
- **Additional seller requirements:** Obtain supported creative durations, VAST compatibility, media encoding requirements, accepted delivery methods, review policies, and submission deadlines from the relevant supply partner through its existing integration documentation or other agreed communications. **Do not treat sample-only Forecast attributes such as `maxadduration` as mandatory or formally defined API fields.**

A supply partner may publish different requirements by publisher, inventory, device/player, event, or insertion path. Buyers should confirm which rules apply to the planned activation, and sellers should identify which organization will perform creative review and which systems must subsequently deliver the asset.

## 5. Submitting and maintaining ads through AMA v1.1

**Submit early.** A demand partner can submit a representative AdCOM `Ad` through AMA v1.1 as soon as the relevant supply partner and proposed creative are known, subject to that partner's process and policy. The supplied `Ad` must meet the receiving partner's requirements. For video, accurate duration, media characteristics, and representative markup or markup reference help avoid preventable review cycles.

**Monitor audit results.** Consumers may query a particular ad or retrieve updates using `auditStart`. The change feed is based on the Audit object's `lastmod`. Implementations should follow the AMA v1.1 `nextPage` / `paginationId` pagination behavior until the full result set is collected, retaining the correct cursor so ads with identical modification timestamps are not omitted. Responses may contain sparse fields; omitted fields must not automatically be interpreted as deletions or changes.

**Respond to findings.** Human-readable `audit.feedback` can explain a denial or review concern. `audit.corr` can communicate corrections to the submitted ad's fields. Buyers should reconcile that information into creative records, address material issues, and use `PUT` or `PATCH` where the supply partner permits an update or re-audit request. A request to re-audit does not guarantee approval or immediate eligibility.

**Use webhooks if agreed.** AMA v1.1 defines an optional webhook mechanism for audit updates. If a supply partner offers it and the parties configure it, consumers can receive change notifications and periodically reconcile using GET to cover missed deliveries. Webhooks are not mandatory and should not be assumed to exist in every implementation.

**Check expiration.** Ads may expire or be deleted under documented supply-partner policies. Partners should review expiration and reactivation requirements before an event, particularly when a creative was approved well in advance. Whether reactivation triggers a new audit or changes bidding eligibility is supply-partner policy.

## 6. Interpreting audit status

AMA v1.1 uses the AdCOM `Audit` object. The following values are already defined in AdCOM; their bidding consequences depend in part on the supply partner's documented restrictive or permissive policy.

| `audit.status` | AdCOM status | Suggested live-event consideration |
|---:|---|---|
| `1` | Pending audit | Follow the receiving partner's policy. Under restrictive bidding, do not assume the ad can be used before approval. |
| `2` | Pre-approved | Confirm the seller's policy and continue monitoring; this is not a completed audit. |
| `3` | Approved | Approval permits the buyer to apply the seller's relevant bidding policy; it **does not independently prove ingestion, transcoding, or event-specific eligibility**. |
| `4` | Denied | Review `audit.feedback`, correct the issue if possible, and request re-audit under the partner's process. |
| `5` | Changed; resubmission requested | Determine the effect on eligibility, update the submitted ad, and obtain the partner's required review. |
| `6` | Expired | Check the supply partner's expiration and reactivation policy before using the ad. |
| `500+` | Vendor-specific | Only interpret a documented bilateral code according to that implementation; **this guide does not define a standardized live-readiness status**. |

A restrictive bidder/seller policy may prohibit unapproved ads from serving. A permissive policy may allow some ads before explicit approval, subject to later rejection. Both models exist in AMA v1.1; this guide does not supersede the supply partner's documented policy. For high-value live events, early submission and review are useful under either model.

## 7. Audit approval, eligibility, and delivery readiness

Three questions should be kept separate:

1. **Audit approval:** Has the relevant AMA supply partner reviewed the submitted `Ad` under its audit policy? AMA v1.1 is intended to convey this result.
2. **Transaction eligibility:** Is that creative permitted for the particular publisher, placement, deal, event, and current bid opportunity? Eligibility may depend on policies and transaction information beyond AMA's audit status.
3. **Delivery readiness:** Has the necessary insertion/rendering path obtained and prepared the creative, including any ingestion, transcoding, caching, or distribution required for SSAI, SGAI, client-side, or hybrid delivery? AMA v1.1 **does not standardize an independent completion signal** for this condition.

A status of `3` must retain its published meaning of **Approved**, not be reinterpreted as a promise that every downstream delivery path is ready. Likewise, free-text audit feedback should not be treated as a standardized machine-readable readiness indicator. If delivery preparation requires separate confirmation, the parties should use existing documented operational arrangements. This guidance does not define their interface, state model, or success criteria.

When a publisher, SSP, or downstream platform performs separate reviews, partners should identify whose approval applies to each step. A positive audit result from one intermediary does not automatically establish acceptance by every participant in the delivery chain.

## 8. Live-event operational timeline

The schedule below is **illustrative**, not a prescribed deadline or new compliance requirement. The relevant supply partner's published review and asset-delivery lead times govern. An event can be tentatively scheduled and still justify early creative preparation; approval is not guaranteed to carry forward if material requirements change.

| Window | Suggested operational activity |
|---|---|
| Event discovery | Review Forecast information; identify likely creatives, the responsible supply partners, applicable review policies, and delivery requirements. |
| Ahead of the partner-agreed submission cutoff | Submit representative AdCOM ads through AMA v1.1 where supported; monitor pending or denied items and allow time for remediation. |
| Pre-event checks | Reconcile status through GET and/or optional webhooks; confirm the planned creative set, expiration policy, and any separate delivery-path preparations. |
| Immediately before and during the event | Monitor relevant audit changes at a partner-agreed cadence; stop or alter use of an ad when the receiving partner's policy or audit results require it. Live Concurrent Streams observations can inform campaign scale, but not creative approval. |
| Post-event | Review rejected, changed, expired, or unusable creatives; compare actual review and delivery lead times against the event plan. |

An implementation may choose concrete checkpoints, for example seven days, 24 hours, or one hour before start, but **AMA v1.1 and LEAP v1.0 do not standardize those milestones**.

## 9. Identity and reconciliation

**Creative identity and event identity are different.** AMA manages an `Ad` identified by its `id` in the supply partner's configured ID scheme; OpenRTB bid responses may carry a creative ID such as `crid`. Partners should document how they map the managed Ad record to the identifier used in bidding and reporting. **Do not assume `Ad.id` always equals `crid`:** AMA v1.1 permits bidder-assigned and exchange-assigned ad-ID models.

Separately, the LEAP Forecast API, Concurrent Streams API, and OpenRTB can correlate content by a consistent `content.id` **or** by an appropriately source-qualified Extended Content ID, as described in the [Common API Guidance](Common_API_Guidance.md). Neither approach inherently joins a creative's AMA audit record to a particular event.

For this release, buyers and sellers can maintain event-to-creative associations in their existing planning or campaign systems. **No standard AMA v1.1 field is defined by this guide for listing events associated with an Ad, indicating event-specific approval, or asserting that an Ad has become eligible for that event.**

## 10. Worked interaction outline

The following is an interaction outline using published AMA v1.1 operations; it is **not a complete creative payload**. Actual request bodies must use AdCOM `Ad` objects meeting the partner's integration requirements.

```http
POST {baseUrl}/bidder/{bidderId}/ads
Content-Type: application/json

{AdCOM Ad object submitted by the demand partner}
```

The response returns a collection containing the submitted ad and relevant audit information. A response indicating pending audit is not an approval.

Later, the consumer can retrieve changes using the Audit object's Unix-millisecond `lastmod` values:

```http
GET {baseUrl}/bidder/{bidderId}/ads?auditStart={unix_milliseconds}
Accept: application/json
```

The consumer follows `nextPage`, when supplied, through completion; interprets `audit.status`, `audit.feedback`, and `audit.corr` according to the specification; and updates its activation plan. An optional webhook may deliver the same kind of audit changes asynchronously. For a particular ad, a direct GET to `{baseUrl}/bidder/{bidderId}/ads/{id}` is also supported.

The chosen `baseUrl`, authentication method, bidder ID, actual AdCOM payload, and commercial eligibility rules are integration-specific. No LEAP-specific `Ad.ext` object, event-approval state, or readiness field is used in this example.

## 11. Known limitations and future work

AMA v1.1 already supports important pre-event creative-review tasks, but **a complete, interoperable live-event creative-readiness lifecycle is not yet standardized by this guide**. Areas requiring further discussion include:

- Distinguishing audit approval, publisher/event eligibility, and asset delivery-path readiness with consistent machine-readable semantics.
- Discovery of creative and delivery requirements that differ among publishers, platforms, and event types.
- Associating a creative with one or more events without assuming the creative ID is itself an event identifier.
- Representing multiple audit or validation authorities across intermediaries and insertion paths.
- Aligning existing and proprietary review integrations, and understanding preparation requirements in SSAI, SGAI, client-side, and hybrid environments.
- Coordinating the separate LEAP PreFetch & Cache work stream with the creative-review lifecycle.

These are known gaps, **not capabilities implemented or required by AMA v1.1**. They are appropriate topics for future LEAP Creative Readiness work and consideration of AMA enhancements (including the proposed AMA v1.2 discussions), subject to the relevant working groups and approval processes. The objective is to improve interoperability without creating breaking changes for current AMA v1.1 implementations.

Industry participants are encouraged to contribute use cases, implementation experience, and proposed solutions through the [LEAP repository](https://github.com/IABTechLab/Live-Event-Ad-Playbook/issues) and through IAB Tech Lab's Advanced TV / Live Event Ad Playbook working group activities.

## 12. References

- [Ad Management API v1.1 specification](https://github.com/InteractiveAdvertisingBureau/AdManagementAPI/blob/master/Ad%20Management%20API%20v1.md)
- [IAB Tech Lab AdCOM](https://github.com/InteractiveAdvertisingBureau/AdCOM)
- [LEAP Forecast API specification](../Forecast_API.md)
- [LEAP Concurrent Streams API specification](../Concurrent_Streams_API.md)
- [LEAP Common API Guidance](Common_API_Guidance.md)
