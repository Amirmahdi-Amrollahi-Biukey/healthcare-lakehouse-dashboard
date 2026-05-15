# Decision Support Dashboard

Four-page Power BI report built around a Decision Support Manager persona. The persona reports to the COO; she sends pages to leadership and uses the report to answer "where should the organization put its money and attention next quarter?"

The file is `Leadership_Dashboard.pbix` (43 MB). Page screenshots are in `screenshots/`.

## The questions the dashboard answers

These framed the design before any visuals were built.

**Volume and revenue trends**
- Are encounter volumes growing year over year, by state and by encounter class?
- Is total claim billing trending up or down, and is that driven by volume, case mix, or pricing?
- How much outstanding balance is sitting with primary insurer vs. secondary vs. patient, and is that mix shifting?

**Payer mix and concentration risk**
- What percent of encounters and what percent of claim value goes through each payer?
- Which payers are growing as a share of our business and which are shrinking?
- Where do patients have dual coverage (primary + secondary), and what does that look like by state?

**Service line and clinical activity**
- Which encounter classes drive volume (ambulatory, wellness, emergency, inpatient, etc.)?
- Which provider specialties absorb the most encounters?
- What are people coming in for? Are the top reasons changing over time?

**Geographic and demographic shifts**
- Which states are growing or shrinking?
- Is the patient population aging, and where?
- Are demographic mixes shifting in ways leadership should know about?

## Pages

### Page 1: Executive Summary

The single page that has to work in 30 seconds. KPI cards across the top with year-over-year deltas (patients, encounters, claims, outstanding). Charts below: encounter trend (this year vs. last year), top payers, encounters by state, outstanding by responsible party.

This is the page sent to the COO.

### Page 2: Encounter Activity

One level deeper than the summary. Encounters by class, average encounters per patient, monthly trend split by class, top 10 encounter reasons, provider specialty mix, and an encounter duration distribution.

The duration distribution surfaces something useful: hospice and SNF encounters dominate by length even though they're a small share of volume. That's the kind of operational insight leadership pays attention to when planning staff allocation.

### Page 3: Financial / Claims

Claim volume over time, outstanding balance broken down by responsible party (primary / secondary / patient), payer concentration, and a payer mix shift indicator. Helps answer "is our revenue at risk from any single payer renegotiating?"

### Page 4: Geographic and Demographic

Patient population profile, growth by state, demographic shifts over the window. Mostly used for resource-allocation conversations: which states to invest in, which to monitor.

## Design choices worth flagging

**Relative-date DAX everywhere.** Time-bounded measures use `TODAY()`-anchored windows rather than hard-coded years. The report doesn't go stale when the calendar rolls over.

**Synced slicers across pages.** State and date range are sync-enabled, so picking a state on the summary keeps it filtered on every other page.

**Top-N with a stable "Other" bucket.** Where top-N visuals are used (top 10 reasons, top 5 payers), the grouping is built as a calculated column with rank stability rather than a slicer-dynamic TOPN measure. The dataset's top-N is stable across state and date filters, so this is the right trade-off here; the alternative would be considerable DAX complexity for no real benefit.

**Note on data.** The data is Synthea-generated, frozen between 2021 and 2025. The refresh logic mirrors a production setup, but new patients won't appear on refresh unless Synthea is regenerated with an extended window.

## How to open it

You need Power BI Desktop. The .pbix is in Import mode, so the data is embedded and the report opens standalone without a Databricks connection. To refresh against a live source, the model expects a Databricks SQL warehouse endpoint pointing at `healthcare_demo.healthcare_gold`.
