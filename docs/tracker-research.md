# What makes a funding tracker work: research notes

Research compiled 2026-09-27. It covers grant databases (Grants.gov / Simpler.Grants.gov, GrantWatch and others), filter and data-table UX research (Nielsen Norman Group, Pencil & Paper and others), dashboard design (Stephen Few), and data-pipeline practice (git scraping, deduplication, freshness monitoring, LLM grounding). Sources are at the end.

---

## 1. What doesn't work (common failure modes)

| Failure | Why it kills a tracker |
|---|---|
| **Stale data that looks current** | Stale data "looks perfectly normal": the page renders and nothing errors. A scheduled job can report success while writing zero new rows. Each time users see stale numbers they trust the tool less, and eventually they go back to their own spreadsheets. |
| **Expired listings shown as open** | This is the most common complaint about grant databases: expired or incomplete listings, and irrelevant results that force manual filtering. The better databases archive past-due grants automatically every day. |
| **Broad keyword search with no structure** | Broad terms return hundreds of irrelevant results. This causes search fatigue and missed deadlines. |
| **Filters that mirror the database schema** | Filters should match how users think about the content, not how the data is stored. Jargon and badly chosen filter values make filtering fail. |
| **Hidden active filters** | Users forget a filter is on and conclude that nothing exists. Active filters need to stay visible, and each one needs a one-click way to remove it. |
| **Too many alerts** | Unprioritised notifications get ignored, and then the important ones are missed too. |
| **Cluttered dashboards** | Stephen Few: if a dashboard doesn't tell you what you need to know at a glance, you won't use it. |
| **LLM-extracted facts without verification** | LLMs make more errors parsing free text than extracting into a typed schema. Deadlines, amounts and URLs they produce need to be checked against the source page. |

## 2. Filtering: what works

1. **Pick facets from how users decide.** For funding, that is: *Can I apply?* (eligibility: org type, region), *Is it worth it?* (amount, thematic fit), *When?* (deadline window, rolling vs fixed) and *Who?* (funder type). Put first the facets that cut the list down the most.
2. **Short, predictable values in plain language.** Use "Closes in 30 days" rather than a raw date picker, and "Think tanks eligible" rather than an internal code.
3. **Quick filters above the table** for the most common tasks, for example *Closing soon*, *New this week*, *Top fit (4–5)* and *Rolling*.
4. **Show every active filter as a removable chip**, plus a "Clear all" button. Show the result count, for example "7 of 16".
5. **Useful empty states.** When nothing matches, name the filter that removed everything and offer to clear it.
6. **Sensible default view.** Show only open opportunities, sorted by what needs action first (nearest deadline, then fit). Archived items should be one click away, not mixed into the main list.
7. **Saved searches and alerts** (Grants.gov "Save Search" plus email subscriptions). A cheap version for a static site is to store the filter state in the URL query string, so a view can be bookmarked and shared.
8. **Filter on the client for small datasets.** Under about 1,000 rows, in-browser filtering is instant. That fits this tracker.

## 3. A useful UI

- **Glanceable summary first**, following Few: counts for *open*, *closing in 14 days* and *new since last visit*, then the table.
- **Sortable columns with a visible sort indicator.** Default sort: needs action soonest.
- **Consistent deadline urgency cues**, such as colour plus text like "5 days left". Colour alone is not accessible.
- **Detail panel for the long text.** Keep the table scannable (name, funder, amount, deadline, fit) and put the description, notes and eligibility in the drawer.
- **"New since your last visit"** means exactly that, and it clears once seen. It is not a fixed 7-day window. On a static site, store the last-visit timestamp in `localStorage` and compare it with `date_added` and `last_changed`.
- **Show where each record came from and when it was checked.** Add a "last verified" date per row and a link to the source. Flag broken links, which this tracker already does with `broken_url`.
- **Show data freshness prominently.** Put "Last updated" next to the data, and show a warning when it is older than expected, for example "Data is 6 months old; the update job may have failed."
- **Work on mobile.** Filters go in a drawer and table rows become cards.
- **Export** to CSV or calendar (`.ics` for deadlines), because teams plan in their own tools.

## 4. Updating from new information

A reliable pipeline has these steps:

1. **Discover.** Search on a fixed schedule over a list of known funder pages and saved queries, not just open-ended web searches.
2. **Extract into a strict schema.** Validate types and required fields (deadline is ISO or `rolling`/`TBC`, amount is a number, URL is valid). Reject or quarantine records that fail.
3. **Normalise and deduplicate.** Canonicalise URLs (strip tracking parameters), then match on a stable ID or a hash of funder, programme and cycle. The same call often appears on several sites.
4. **Detect changes.** Hash the fields that matter (deadline, amount, status, URL) and label each update as `NEW`, `DEADLINE_CHANGED`, `AMOUNT_CHANGED`, `CLOSED` or `UNCHANGED`. Keep a `last_changed` date and a short change log.
5. **Verify.** Fetch the source URL and confirm the deadline and amount appear there. Record `last_verified`. Mark a record `needs_review` rather than publishing unverified LLM output.
6. **Manage the lifecycle automatically.** Move items whose deadline has passed to the archive every day. For recurring programmes, add a "watch for next cycle" state.
7. **Monitor the pipeline itself.** Alert when a run adds or changes nothing for N days, when the output is empty or shrinks sharply, or when the published date stops moving. A run that says it succeeded is not proof that the data is fresh.
8. **Store history in git ("git scraping").** Commit each run's data file so git history becomes an audit log you can diff (the Simon Willison / GitHub Flat Data pattern). It fits a GitHub Pages site well.
9. **Send notifications as a digest.** Send one summary per run or per week with new items and deadline changes, and include only what matches the user's saved filters.

---

## 5. Findings for this repo (as of 2026-09-27)

- **The data is about 6 months stale.** The header says "Last updated: 27 March 2026 · Auto-updated daily" and the last commit is 2026-03-27. The daily job has stopped, and nothing on the page shows it. This is exactly the silent-staleness failure described above.
- **Expired items are still marked `active`.** All five dated opportunities in `funding_opportunities.json` (deadlines 2026-04-03 to 2026-07-29) have passed but still have `"status": "active"`. The archive step depends on the pipeline running.
- **Filters:** there is keyword search plus Type, Fit and Region dropdowns. Missing: deadline-window / closing-soon filter, eligibility (org type), amount range, active-filter chips with a result count, URL-persisted state and a show-archived toggle.
- **"New" is a manually set `status: "new"`,** not computed from `date_added` against the user's last visit.
- **No per-record `last_verified` / `last_changed` fields,** so users can't tell how trustworthy a row is.
- **The data is duplicated inside `index.html`** (`var ACTIVE = [...]`) as well as the JSON files. Loading the JSON with `fetch()` would give one source of truth and smaller HTML diffs.
- **Scraped text is inserted with string concatenation into `innerHTML`.** Escape it, or use `textContent`, to avoid HTML or script injection from a scraped page.

### Suggested next steps, in priority order

1. Restart the scheduled update, add a staleness warning on the page, and add a pipeline check that fails when the data hasn't changed in N days.
2. Compute status on the client and in the pipeline: when the deadline has passed, archive the item or show it as closed.
3. Add `last_verified`, `last_changed` and `source_checked_url` to the schema, and show them in the detail panel.
4. Add quick filters (Closing ≤30d, New since last visit, Fit 4–5, Rolling), active-filter chips, a result count, and filter state in the URL.
5. Load the data from JSON instead of inlining it, and escape the rendered fields.
6. Optional: `.ics` / CSV export and a weekly digest.

---

## Sources

**Grant trackers and databases**
- [Grants.gov: Subscribe to Saved Searches](https://apply07.grants.gov/help/html/help/Connect/SubscribeToSavedSearches.htm)
- [Grants.gov: Search Grants Tab](https://apply07.grants.gov/help/html/help/SearchGrants/SearchGrantsTab.htm)
- [Grants.gov: Manage Subscriptions](https://grants.gov/connect/manage-subscriptions/)
- [Grants.gov blog: Saved search, improved filtering and sorting](https://grantsgovprod.wordpress.com/2019/08/22/new-on-the-mobile-app-saved-search-improved-filtering-and-sorting-more/)
- [Simpler.Grants.gov user research wiki](https://wiki.simpler.grants.gov/design-and-research/user-research)
- [Simpler.Grants.gov search experience on Grants.gov](https://grantsgovprod.wordpress.com/2025/07/22/the-simpler-grants-gov-search-experience-is-now-available-on-grants-gov/)
- [OpenGrants: 7 mistakes with federal grant search](https://opengrants.io/7-mistakes-youre-making-with-federal-grant-search-and-how-to-fix-them-2/)
- [Atom Grants: Best grant databases 2026](https://atomgrants.com/blog/best-grant-databases-2026)
- [Zeffy: Best grant databases for nonprofits](https://www.zeffy.com/blog/grant-database-for-nonprofits)
- [GrantWatch](https://www.grantwatch.com/)
- [Instil: Nonprofit grant tracking best practices](https://blog.instil.io/nonprofit-grant-tracking-best-practices-for-managing-grants)
- [Good Grants: Must-have dashboard metrics](https://goodgrants.com/resources/articles/5-must-have-metrics-on-your-grant-management-dashboard/)
- [GrantStation: Keeping track of funding prospects](https://grantstation.com/gs-insights/keeping-track-of-funding-prospects)
- [Airtable nonprofit grant tracker template](https://www.airtable.com/templates/nonprofit-grant-tracker/expwzMEi50HFbV7TN)

**Filtering and table UX**
- [NN/g: Defining helpful filter categories and values](https://www.nngroup.com/articles/filter-categories-values/)
- [NN/g: Ecommerce search incl. faceted search report](https://www.nngroup.com/reports/ecommerce-ux-search-including-faceted-search/)
- [Pencil & Paper: Enterprise filter UX patterns](https://www.pencilandpaper.io/articles/ux-pattern-analysis-enterprise-filtering)
- [Pencil & Paper: Enterprise data tables](https://www.pencilandpaper.io/articles/ux-pattern-analysis-enterprise-data-tables)
- [Pencil & Paper: Mobile filter patterns](https://www.pencilandpaper.io/articles/ux-pattern-analysis-mobile-filters)
- [LogRocket: Data table design best practices](https://blog.logrocket.com/ux-design/data-table-design-best-practices/)
- [UX Patterns for Developers: Data table](https://uxpatterns.dev/patterns/data-display/table)
- [UXPin: Filter UI and UX](https://www.uxpin.com/studio/blog/filter-ui-and-ux/)
- [Wikipedia: Faceted search](https://en.wikipedia.org/wiki/Faceted_search)

**Dashboards, freshness and trust**
- [Stephen Few: Dashboard design for at-a-glance monitoring (PDF)](https://www.perceptualedge.com/files/Dashboard_Design_Course.pdf)
- [Why dashboards lose trust over time](https://vixbyte.substack.com/p/why-dashboards-lose-trust-over-time)
- [Data freshness monitoring (DEV)](https://dev.to/iblaine/data-freshness-monitoring-how-to-detect-stale-data-before-it-breaks-dashboards-5hkm)
- [Basedash: Data freshness explained](https://www.basedash.com/blog/data-freshness-how-current-your-dashboard-data-really-is)
- ["New since last visit" design proposal (pulseboard #121)](https://github.com/freeCodeCamp-Summer-Cohort-2026/pulseboard/issues/121)

**Update pipelines**
- [Simon Willison: git scraping](https://simonwillison.net/tags/git-scraping/)
- [GitHub Next: Flat Data](https://githubnext.com/projects/flat-data/)
- [DataHen: Change detection, backfills and deduplication](https://www.datahen.com/blog/beyond-basic-scraping-historical-backfills-change-detection-and-deduplication/)
- [ScrapingAnt: Deduping, canonicalisation, drift alerts](https://scrapingant.com/blog/building-a-web-data-quality-layer-deduping-canonicalization)
- [Context.dev: Scraper monitoring in production](https://www.context.dev/blog/scraper-monitoring-in-production)
- [Firecrawl: Reducing hallucinations in search-grounded LLM responses](https://www.firecrawl.dev/glossary/web-search-apis/reduce-hallucinations-search-grounded-llm-responses)
- [Courier: Reducing notification fatigue](https://www.courier.com/blog/how-to-reduce-notification-fatigue-7-proven-product-strategies-for-saas)
- [PagerDuty: Alert fatigue](https://www.pagerduty.com/resources/digital-operations/learn/alert-fatigue/)
